# Padavan 云编译自定义教程（歌华链 GHL-R-001）

> 仓库：https://github.com/LoganJinDev/Padavan-4.4-Build
> （★ 仓库由 `Padavan-4.4-GHL` **改名**而来，旧名走 GitHub 重定向仍能打开；但本文命令已统一用现名，避免哪天重定向失效。）
> 工作流：`.github/workflows/custom.yml`（`workflow_dispatch`）
> 参考机型：GHL-R-001 — MT7621AT / 32MB Flash / **512MB RAM** / MT7603EN(2.4G) + MT7612EN(5G)

---

## 一、先搞清两层：运行期 vs 编译期

| 层 | 什么时候改 | 什么时候生效 | 要不要重编 |
|---|---|---|---|
| **运行期** | 刷完进 WebUI 改 | 立即 | 不用 |
| **编译期** | 云编译时注入 | **仅首次启动 / 恢复出厂 / 清 NVRAM 后** | 要 |

**判据**：
- 只想改 IP / SSID / 密码 / 信道 / 端口转发 → **进 WebUI 改，不用重编**
- 想让"恢复出厂"就自带你的值 → 云编译时注入
- 想改功能矩阵（加插件、砍体积）→ 必须重编

⚠️ NVRAM 存在 flash 的 `Config` 分区（**GHL 是 `0x30000` / 64 KB**；其他机型偏移不同，见第七节坑 8），刷固件**不覆盖**它。所以跨板刷机、或改完编译期默认值后，**必须清 NVRAM / 恢复出厂**才看得到新默认值。

---

## 二、云编译参数速查

| 参数 | 取值 | 说明 |
|---|---|---|
| `target` | `GHL` | 板子。**选错就是 WiFi 失效的根因** |
| `toolchain` | `mipsel-linux-musl` / `mipsel-linux-uclibc` | musl 体积小、默认 |
| `customization` | JSON | 地址/账号/SSID/密码，见第三节 |
| `plugins` | JSON | 功能开关，见第四节 |
| `cpufreq` | bool | 是否超频 |
| `mt7621_oc` | `0x312`=1000MHz / `0x362`=1100MHz / `0x372`=1120MHz / `0x3B2`=1200MHz | 仅 `cpufreq=true` 时有效 |
| `nanoversion` | bool | **砍掉全部插件做最小固件**。要插件时务必 `false` |
| `release` | bool | 是否发 GitHub Release |
| `oldworkflow` | bool | 删历史构建记录 |

---

## 三、`customization` — 地址 / SSID / 密码

JSON 键**全小写**，脚本会转大写当环境变量用。

| key | 作用 | 落到的宏 |
|---|---|---|
| `lanip` | LAN IP | `DEF_LAN_ADDR`（连带把 `DEF_LAN_DHCP_BEG`/`_END` 的前三段一起改）＋ `trunk/user/www/dict/CN.dict` 的提示文案 |
| `signaccount` | WebUI / root 账号 | `SYS_USER_ROOT` |
| `signpassword` | root 密码 | `DEF_ROOT_PASSWORD` |
| `wifi2gssid` | 2.4G SSID | `DEF_WLAN_2G_SSID` |
| `wifi2gpsk` | 2.4G 密码 | `DEF_WLAN_2G_PSK` |
| `wifi5gssid` | 5G SSID | `DEF_WLAN_5G_SSID` |
| `wifi5gpsk` | 5G 密码 | `DEF_WLAN_5G_PSK` |

改动落在**两个**文件：`trunk/user/shared/src/defaults.h` 与 `trunk/configs/boards/<PID>/board.h`。
后者带 `#ifndef` 守卫、会**盖住**前者 —— 只改一个就是「改了一半」。

> ⚠️ **这里故意不写行号。** 脚本是按**宏名**改写的，行号会随上游改动漂移：本文早期版本写的那组
> `L33`/`L47`/`L40`/`L44`/`L41`/`L45` 在 `defaults.h` 从 60 行变 81 行之后**已经整列错位**
> （`DEF_ROOT_PASSWORD` 实际在 **L68**，而 L47 位置现在是 `#endif`）。行号只能当"往哪找"的方向，
> **别拿它定位**。

**DHCP 范围不用给**：`setip.sh` 取 `lanip` 前三段自动算 `x.x.x.100` – `x.x.x.244`。
例：`lanip=192.168.8.1` → DHCP 自动 `192.168.8.100`–`192.168.8.244`。

### 脚本改不到、只能手改 `defaults.h` 的默认值

| 宏 | 现值 | 说明 |
|---|---|---|
| `DEF_LAN_MASK` | `255.255.255.0` | 子网掩码 |
| `DEF_WLAN_2G_CC` / `DEF_WLAN_5G_CC` | `CN` | 国家码 → 决定可用信道域 |
| `DEF_WLAN_2G_GSSID` / `DEF_WLAN_5G_GSSID` | `BOARD_PID "_GUEST_%s"` / `"_GUEST_5G_%s"` | **访客网络** SSID（运行期拼 MAC 后缀，所以脚本不碰） |
| `DEF_SMB_WORKGROUP` | `WORKGROUP` | SMB 工作组 |
| `DEF_TIMEZONE` | `CST-8` | 时区 |
| `DEF_NTP_SERVER0` / `DEF_NTP_SERVER1` | `ntp1.aliyun.com` / `time1.cloud.tencent.com` | NTP |

> 不传 `customization` 时，默认 SSID 是 `BOARD_PID "_%s"`（运行期拼 MAC 后缀），形如 `GHL_XXXX`，**不是** `Padavan`。

---

## 四、`plugins` — 功能开关

机制：JSON 的每个键 `k: v` → 追加一行 `CONFIG_FIRMWARE_INCLUDE_<K大写>=<v>` 到 `trunk/.config` 末尾。
`trunk/Makefile` 用 `include .config`，make **后赋值覆盖先赋值** → 追加行盖掉模板里的静态值。

**这是通用逃生口**：键名不限于默认那 9 个，只要对应某个 `CONFIG_FIRMWARE_INCLUDE_*` 就能开关。

★★ **`plugins` 是「覆盖补丁」，不是整体替换**（本文早期写反了，见下方勘误）。

`custom.yml` **只把 JSON 里出现过的键**追加进 `trunk/.config`：`:171` 的 `export_json` 是
`jq -r 'to_entries[] | "\(.key)=\(.value)"'`，`:206-207` 再 `for key in $(env | grep '^PLUGIN_')`
逐键写 `CONFIG_FIRMWARE_INCLUDE_${key#PLUGIN_}=${!key}`。**没写的键，一个字节都不动。**

模板里 `CONFIG_FIRMWARE_INCLUDE_*` 的**行首生效**条数：GHL **90**、MI-R3G **85**、MI-R3P **88**
（其中 `=y` 分别 40 / 29 / 33）。所以传 `{"frpc":"y"}` **只覆盖 `frpc` 这一项** ——
`xray`（模板 `GHL.config:253` 已是 `y`）、`shadowsocks`、`trojan`、`sqm` 等**全部照旧保留**。

> ⚠️ **勘误**：本文早期版本写的是「JSON 是整体替换，不是合并」—— **那句是错的**。
> 按错的说法操作只是白抄一长串键；真正有害的是反过来 ——「怕丢默认项所以不敢只传一个键」，
> 会让人白白放弃这个参数。

### 4.1 FRP 内网穿透 ✅ 可用

| 键 | 组件 | 未压缩 | `gzip -9` 后 |
|---|---|---|---|
| `frpc` | 客户端（`usr/bin/frpc`） | 4,135,480 B | **4,039,224 B = 3.85 MiB** |
| `frps` | 服务端（`usr/bin/frps`） | 5,097,556 B | 4,980,819 B = 4.75 MiB |

> 两个数都是从 `trunk/user/frp/frp_0.60.0_linux_mipsle.tar.gz`（9,028,946 B）里解出来实测的。
> `gzip -9` 后 ≈ 未压缩（压缩比 **1.023**）—— **Go 二进制几乎压不动**，所以这类插件的 squashfs 增量基本就等于二进制本身大小。

- **家用只开 `frpc`**，`frps` 是公网服务器用的，开了白吃 4.75 MiB
- WebUI 面板：`Advanced_frp.asp` ＋ `Advanced_frp_log.asp`，保留条件 `FRPC_n_FRPS != n_n`
- 启动脚本：`/usr/bin/frp.sh`、`/etc_ro/frp_script.sh`；编译宏 `-DAPP_FRP`
- 版本 0.60.0，mipsle，预编译包已在仓库 `trunk/user/frp/frp_0.60.0_linux_mipsle.tar.gz`

### 4.2 LAN 设备网速控制 ✅ 可用，但要说清能力边界

**Padavan 只有一个限速 UI：SQM**（`Advanced_SQM.asp`，中文菜单「QOS流控」，字典键 `menu5_37`）。

SQM 面板字段：

| 字段 | 含义 |
|---|---|
| `sqm_enable` | 总开关 |
| `sqm_up_speed` / `sqm_down_speed` | **整条线路**的上行/下行总带宽（kbps） |
| `sqm_qdisc` | 队列算法（cake / fq_codel / sfq …） |
| `sqm_script` | 脚本模式，见下 |
| `sqm_log_level` / `sqm_debug_log` / `sqm_active` | 日志与状态 |

`trunk/user/sqm-qos/files/usr/lib/sqm/` 里的脚本模式：

- `simplest_tbf.qos` — 最简单 TBF，**硬性总带宽限速**
- `simple.qos` / `simplest.qos` — fq_codel 系
- `layer_cake.qos` — **CAKE + 分层公平队列 → 每台设备公平分享，防止单台吃满**
- `piece_of_cake.qos` — CAKE 基础版

**它做不到什么**：没有「给 192.168.1.5 固定 10Mbps」这种按设备限速的 UI。
`CN.dict` 里那套 `BM_*`（EzQoS 带宽管理）是华硕原厂遗留的**死词条**（机器名占位符 `ZVMODELVZ` 还留着），Padavan 没实现。

**想要按设备硬限速怎么办**：开了 SQM 就装了 `bin/tc`（iproute2）＋ IFB ＋ CAKE 模块，
可自己写 `tc` 脚本放进 WebUI 的「脚本」页（`Advanced_Scripts_Content.asp`）实现 per-IP 限速。
日常更推荐直接用 CAKE 的 `layer_cake.qos`——不设死限速，但任何一台设备都抢不满带宽。

**⚠️ 必开三件套**：`sqm` ＋ `qos` ＋ `ifb` 三个都要 `y`，缺一不可。原因见第七节坑 1。

| 键 | 作用 |
|---|---|
| `sqm` | 装 `bin/tc`、SQM 脚本、WebUI 页面；开 `NET_SCH_SFQ/CAKE/CODEL`、`NET_ACT_IPT` 等内核参数 |
| `qos` | 保留内核 `NET_SCHED` / `NET_CLS` / `XT_TARGET_CLASSIFY`（默认 `n` 会被显式关掉） |
| `ifb` | 保留 `NET_SCH_INGRESS` / `NET_CLS_ACT` / `CONFIG_IFB`（**下行限速必需**） |

体积：三者合计 **≈ 0.3–0.5 MB**（tc 二进制 + 若干内核模块 + 一个 asp + 少量脚本），非常划算。

`CONFIG_FIRMWARE_INCLUDE_IMQ` **已废弃**——`trunk/configure` 里 IMQ 分支整段被注释掉了，开关无效，别管它。

### 4.3 科学上网节点分流 ✅ 基本可用，边界要说清

当前固件已自带 `usr/bin/xray`（**6,828,824 B 未压缩，最大单体**）、`ss-local`、`ss-redir`、`trojan`、`chinadns-ng`、`smartdns`。

**分流模式**（WebUI「SS」面板 → `ss_run_mode`）：

| 值 | 中文 | 行为 |
|---|---|---|
| `gfw` | gfw列表模式 | gfwlist 内域名走代理，其余直连 |
| `router` | 绕过大陆IP模式 | chnroute 内 IP 直连，其余走代理 ← **最常用** |
| `all` | 全局模式 | 全走代理 |
| `oversea` | 海外用户回国模式 | 反向：只把国内流量拉回 |

配套资源：
- `ss_chnroute_url` / `ss_update_chnroute` — 大陆 IP 段（`trunk/user/chnroute/chnroute.bz2`，仅 25.5KB）
- `ss_gfwlist_url` / `ss_update_gfwlist` — 被墙域名表
- `ss_chdns` / `china_dns` / `pdnsd_enable` — 国内 DNS 与 DNS 缓存
- `smartdns`（`Advanced_smartdns.asp`）— 上游 DNS 分组、DoH/DoT、按域名指定 DNS

配置生成器在 `trunk/user/shadowsocks/ss/`：`genxrayconfig.lua` / `genv2config.lua` / `gentrojanconfig.lua` / `genssconfig.lua`，主调度 `dlink.lua`。

**分流是两层的，别混**：**外层**由 chnroute/gfwlist ＋ iptables 决定「**哪些流量进 xray**」；**内层**由 xray 的 `routing` 段决定「**进 xray 后走哪个出站**」。`genxrayconfig.lua` 生成**多出站**（`default` / `direct` / `block` / `node_<序号>`，含 `node_5_via_node_3` 形的前置代理链），`routing` 段由 `xrayroute.lua` 生成。所以：

- ✅ 「国内直连、国外走代理」——开箱即用
- ✅ 「域名 A 走节点1、域名 B 走节点2」——**面板支持**。UI 在 `Shadowsocks_Route.asp` 的分流规则页与 ACL 页，每条规则绑一个出站（可再指定前置代理）
- ⚠️ 多节点：节点库存 **dbus `ssconf_basic_json_N`**；`allping.sh`→`ping.sh`→`ssping.lua` **只测延迟/丢包并写回节点库，不会自动切换**；nvram `global_server`（`shadowsocks.sh:36`）是 routing 的**无条件兜底 default**，只能 WebUI 手选。**没有自动故障切换** —— xray 自动选路必须靠 `observatory` / `balancer`，本仓库全树**零命中**

> ⚠️ **勘误（2026-11-16 源码核验）**：本节旧版写「`genxrayconfig.lua` 只生成**一个 outbound ＋ 一个 `freedom`(direct)**，**没有 `routing` 段**」——**已作废**，那描述的是更早的版本。
> 现版（`genxrayconfig.lua` **432 行** / `xrayroute.lua` **495 行**）文件头即自陈「多出站 + 前置代理 + routing 段」，`:393` 调 `xrayroute.build_routing(resolve)`，`:429-430` 最终 JSON 同时含 `outbounds` 与 `routing`。★ 与旧结论相反，故上面三条一并更正。

> ⚠️ **勘误（2026-10-14 实测）**：早期版本本教程写「多节点用 `ssp_*` 字段做测速选优/故障切换」——**错误**。
> `ssp_*` 是**死 nvram 变量**：全仓库没有任何 `.sh`/`.lua` 读取（只在 `shared/src/defaults.c`、`httpd/src/variables.c` 注册，`Shadowsocks.asp` 表单里显示）。真正的节点库是 dbus `ssconf_basic_json_N`。

### 4.3.1 分流/节点机制的源码证据

**启动链**：`rc/src/rc.c:797` → `scripts/files/usr/bin/autostart.sh:64-67` → `/usr/bin/shadowsocks.sh start`
（运行期重启走 `services.c:297-306` → `restart_ss()`）

**配置生成**：`shadowsocks/scripts/shadowsocks.sh:97-145` `gen_config_file()` 按 `d_type` 调 lua，产物**全部重定向到 `/tmp`**：

| lua | 产物 |
|---|---|
| `genxrayconfig.lua` / `genv2config.lua` | `/tmp/v2-redir.json` |
| `gentrojanconfig.lua` | `/tmp/tj-redir.json` |
| `genssconfig.lua` / `genssrconfig.lua` | `/tmp/shadowsocksr.json` |

**喂给进程**：`shadowsocks.sh:273-276` → `run_bin /usr/bin/xray -config /tmp/v2-redir.json`

**三道结构约束**（都跟「选哪个节点」无关，但决定了自定义配置的天花板）：

1. **兜底单节点**：`shadowsocks.sh:36` `GLOBAL_SERVER=\`nvram get global_server\`` —— 它是 routing 的**无条件兜底 default**，**不**代表「只能有一个出站」（多出站由分流规则按需追加，见上）
2. **单出口端口**：`ss-rules:180-184` 全部 `-j REDIRECT --to-ports 1080` —— 这是**外层**：进 xray 的流量统一从 1080 交给它，再由 xray 内部 routing 分派
3. **无自定义配置入口**：`gen_config_file()` **每次无条件覆盖** `/tmp/*.json`，没有「存在即跳过」、没有 nvram 开关、没有目录优先级

> 原第 3 条「分流只管『走不走代理』」已并入第 2 条 —— `ss-rules` 的 `ipset_r()`（:87-157）按 gfw/router/oversea/all 建 ipset 链，那正是外层在决定「哪些流量进 xray」。

**免改 C 的用户挂钩**（想做自定义分流只能靠这些）：

| 钩子 | 执行时机 | 源码位置 |
|---|---|---|
| `/etc/storage/started_script.sh` | boot 完成后（`autostart.sh` 之前） | `rc/src/rc.c:796` |
| `/etc/storage/post_wan_script.sh` | WAN 链路建立后 | `rc/src/net_wan.c:1265,1384`、`detect_link.c:90` |
| `/etc/storage/post_iptables_script.sh` | 防火墙重建后 | `rc/src/firewall_ex.c:2146` |
| `/etc/storage/ss-rules` | 覆盖原生 iptables 规则 | `shadowsocks.sh:38-39` |
| `/etc/storage/ss_dom.sh` / `uss_dom.sh` | 自定义域名黑白名单 | `ss-rules:143-146` |

> 注册与落盘：`www/n56u_ribbon_fixed/Advanced_Scripts_Content.asp` 表单 → `httpd/src/web_ex.c:895-907`（`write_textarea_to_file()`，`.sh` 自动 `chmod 0755`），目录 `/etc/storage`。

**做 passwall2 式分流：路线 A 已落地，另两条只在需要更细粒度时才考虑**：

| 路线 | 要重编? | 说明 |
|---|---|---|
| **A. 用面板（✅ 已具备）** | ❌ 不用 | `Shadowsocks_Route.asp` 的分流规则页 + ACL 页直接配「哪个域名/IP 走哪个出站」，底层就是 `genxrayconfig.lua` 的多出站 + `xrayroute.lua` 的 routing 段。**默认走这条** |
| **C. 多实例 + iptables 分发** | ❌ 不用 | 起多个本地代理端口（如 1081/1082/1083）各连一个节点，用 ipset + iptables 把不同目标打到不同端口。仅在需要**绕开 xray**（例如两个节点协议不同、各要一个内核）时才用，脚本挂 `started_script.sh` |
| **B. 绕过面板自建** | ✅ 要改脚本 | 手写完整 xray config；**但 `/tmp/*.json` 每次 start 被覆盖**，必须挂钩子脚本在固件启动之后重写回去。仅在面板表达不了目标语义时才用 |

⚠️ **没有任何「保护自定义配置不被覆盖」的机制**——B 路线只能靠钩子事后覆盖（这也是优先用 A 的原因之一）。

**sing-box 不要开**：模板里有 `CONFIG_FIRMWARE_INCLUDE_SINGBOX`（开关在 `trunk/configs/templates/*.config`，**当前默认已是 `n`**，连二进制都不进镜像），但 `trunk/user/www` 和 `trunk/user/shadowsocks` 里**没有任何 sing-box 集成**（无 asp、无 lua 生成器）。开了只有二进制躺着，**面板配不了**，还白吃 7–8MB。

---

## 五、体积预算（硬约束）

GHL 的 flash 布局（`mt7621-spi-32m-layout.dtsi`）：

| 分区 | 偏移 | 大小 |
|---|---|---|
| Bootloader | `0x00000` | 192 KB |
| Config (NVRAM) | `0x30000` | 64 KB |
| Factory (RF 校准) | `0x40000` | 64 KB |
| **firmware** | **`0x50000`** | **0x1F30000 = 32,702,464 B = 31.19 MiB** |
| Storage | `0x1F80000` | 512 KB |

→ **`.trx` 文件上限 = 31.19 MiB**（这就是"不能超过 31m"的出处）。

当前基准：本仓库最新 Release `20261027` 里的 GHL 产物。

```
firmware 分区 : 32,702,464 B = 31.19 MiB
当前 trx      : 26,483,010 B = 25.26 MiB   （Release 20261027 的 GHL .trx）
剩余          :  6,219,454 B =  5.93 MiB
```

> ⚠️ **别再引用本文早期那组 `20261013` 的数（19,593,901 B / 余 12.50 MiB）** —— 早过期了。
> 那一版插件集比现在小得多，两组余量差了 **6.5 MiB 以上**；照那组数往里加插件会**把 firmware 分区撑爆**。

SquashFS 用 **gzip**（`MKSQUASHFS_OPTS` 未指定 `-comp`，取 mksquashfs 默认），
但 **Go 二进制几乎不可压**（`frpc` 实测 1.023:1）—— `xray` / `frpc` / `frps` 都属这一类。

### 加东西前先算，别信写死的数

**别信任何「进固件约 X MB」** —— 口径不统一（有的指未压缩二进制，有的指 squashfs 之后），换一版源码就过期。唯一可靠做法是**差值法实测**：

1. 编一版基线，记下 `.trx` 字节数
2. 只改这一项插件，再编一版
3. 两次 `.trx` 相减 = 该项的真实增量

下面这些**未压缩二进制**的量级只用来看谁大谁小，**不能直接当增量**：

| 项 | 配置 | 未压缩 | 说明 |
|---|---|---|---|
| FRP 客户端 | `frpc=y`（`frps=n`） | 4,135,480 B | `gzip -9` 后 4,039,224 B = 3.85 MiB，可当增量的**上界**（Go 二进制压不动） |
| FRP 服务端 | `frps=y` | 5,097,556 B | 公网服务器用的，家用开了白吃 |
| LAN 限速 | `sqm`+`qos`+`ifb` | ≈ 0.3–0.5 MiB | `bin/tc` ＋ 内核模块 ＋ asp ＋ 脚本 |
| 节点分流 | **无需新增** | 0 | xray / smartdns / chnroute **已在当前固件内** |

按上面的余量，GHL 现在只剩 **≈ 5.9 MiB**：加 SQM 这类小件没问题，`frps` / `aria` 这种大件要**先算再开**。

### 想腾地方可关的大件

| 键 | 省 | 说明 |
|---|---|---|
| `aria` + `aria_web_control` | ≈ 4.2 MB | 不用下载器就关（`aria2c` 未压缩 813,388 B，加 Web 控制） |
| `xupnpd` | ≈ 0.3 MB | 不用 IPTV 组播转单播就关 |
| `hdparm` + `parted` | ≈ 0.4 MB | 不折腾 U 盘分区就关 |
| `vlmcsd` | ≈ 0.2 MB | 不用 KMS 激活就关 |
| `openvpn` | ≈ 0.4 MB | 不用就关（openssl 被 sftp/openssh 共用，单关无效） |

### 推荐的 `plugins` JSON

```json
{
  "frpc": "y",
  "frps": "n",
  "sqm": "y",
  "qos": "y",
  "ifb": "y",
  "smartdns": "y",
  "shadowsocks": "y",
  "xray": "y",
  "trojan": "y",
  "ssobfs": "y",
  "vlmcsd": "y",
  "v2ray": "n",
  "singbox": "n",
  "ttyd": "n",
  "iperf3": "n"
}
```

若要同时瘦身，追加：
```json
{"aria": "n", "aria_web_control": "n", "xupnpd": "n", "hdparm": "n", "parted": "n"}
```

---

## 六、发起构建

**WebUI**：Actions → **Custom Build** → Run workflow → 把 JSON 填进 `customization` / `plugins` 字段。

**CLI**：

```bash
# 只加 FRP + LAN 限速（分流已具备）
gh workflow run custom.yml --repo LoganJinDev/Padavan-4.4-Build \
  -f target=GHL -f toolchain=mipsel-linux-musl \
  -f plugins='{"frpc":"y","frps":"n","sqm":"y","qos":"y","ifb":"y","smartdns":"y","shadowsocks":"y","xray":"y","trojan":"y","ssobfs":"y","vlmcsd":"y","v2ray":"n","singbox":"n","ttyd":"n","iperf3":"n"}'

# 同时自定义地址/SSID/密码（7 个键一次给全，别漏 lanip）
gh workflow run custom.yml --repo LoganJinDev/Padavan-4.4-Build \
  -f target=GHL -f toolchain=mipsel-linux-musl \
  -f plugins='{"frpc":"y","sqm":"y","qos":"y","ifb":"y","smartdns":"y","shadowsocks":"y","xray":"y","trojan":"y","ssobfs":"y","vlmcsd":"y"}' \
  -f customization='{"lanip":"192.168.8.1","signaccount":"root","signpassword":"MyPass123","wifi2gssid":"MyHome","wifi2gpsk":"MyPass12345","wifi5gssid":"MyHome-5G","wifi5gpsk":"MyPass12345"}'

# 盯构建
gh run watch $(gh run list --repo LoganJinDev/Padavan-4.4-Build --limit 1 --json databaseId -q '.[0].databaseId') \
  --repo LoganJinDev/Padavan-4.4-Build
```

带 `nanoversion=true` 会**清空全部插件**，要插件时保持默认 `false`。

---

## 七、坑（踩过的）

1. **`sqm=y` 必须同时 `qos=y` ＋ `ifb=y`** —— `trunk/configure` 的执行顺序是 SQM(133行) → QOS(272行) → IFB(285行)，**后面的分支会覆盖前面的**：
   - `QOS != y` → 执行 `func_disable_kernel_param CONFIG_NET_SCHED` / `NET_CLS`，而 `sch_cake`/`sch_sfq` **依赖 NET_SCHED** → SQM 实际失效
   - `IFB != y` → 执行 `func_disable_kernel_param CONFIG_IFB`，把 SQM 分支刚开的 IFB 又关掉 → 无法做下行限速

   只看模板默认值 `QOS=n` 就以为"QoS 是可选功能"，会导致 SQM 装上却跑不起来。

2. ~~三个 sed 脚本用硬编码行号~~ → **已改成按【宏名】匹配，没有行号可错了**（早期版本这条已作废）。三脚本在 `trunk/custom/scripts/`：
   - `setip.sh` → `DEF_LAN_ADDR` / `DEF_LAN_DHCP_BEG` / `DEF_LAN_DHCP_END`，且 `defaults.h` 与 `board.h` **两个文件都改**（`board.h` 的 `#ifndef` 守卫会盖住 `defaults.h`，只改一个就是「改了一半」）；顺带刷 `CN.dict` 提示文案
   - `setaccount.sh` → `set_macro SYS_USER_ROOT` / `set_macro DEF_ROOT_PASSWORD`
   - `setwifi.sh` → SSID 走 `set_expr DEF_WLAN_2G_SSID` / `set_expr DEF_WLAN_5G_SSID`（替换**整段右值**，因为值是 `BOARD_PID "_%s"` 不是字符串字面量）；PSK 走 `set_quoted DEF_WLAN_2G_PSK` / `set_quoted DEF_WLAN_5G_PSK`（只换引号内）

   **历史教训**（就写在 `setaccount.sh`/`setwifi.sh` 头注释里）：commit `303108e3c` 把 `defaults.h` 从 60 行改成 81 行后，写死的 `47s`/`40s`/`41s`/`44s`/`45s` 全部落到 `#endif` 或注释上 —— **sed 不报错、退出码 0、什么都没改**，云编译传的 `wifi2gssid` 因此从来没生效过。
   ⇒ 规则：**别依赖行号；"改了没反应"时先用 `git diff` 确认文件真的变了。**

3. ~~`setip.sh` 没有空值保护~~ → **更正：它有空值守卫**（`setip.sh:36` `if [ -z "$lanip" ]`）。`lanip` 为空时它回读 `board.h` 的 `DEF_LAN_ADDR` 当生效地址（板型默认：GHL `192.168.9.1` / MI-R3G `192.168.32.1` / MI-R3P `192.168.31.1`），读不到才 `exit 0` 跳过。
   所以漏给 `lanip` **不会**把 LAN IP 写成空串 —— 但也**不会**变成你要的值，只会静默沿用板型默认。
   ⇒ **结论仍是「`lanip` 必给」**，只是危害从"变砖"降级为"改了等于没改"。

4. ~~`plugins` JSON 是整体替换不是合并~~ → **反了，它是「覆盖补丁」**：只传 `{"frpc":"y"}` **只覆盖 `frpc`**，`xray`/`shadowsocks` 等照旧保留。详见第四节。
   （顺带纠正一个常被引错的例子：`smartdns` 在 `GHL.config:297` 是 **`n`**，不是"默认 `y` 的项"。）

5. **`setip.sh` 只改 `CN.dict`**，其他语言界面显示的默认 IP 仍是 `192.168.2.1`（纯显示，不影响功能）。

6. **编译期值只是默认值**，存在 NVRAM 里，刷固件不覆盖 → 必须清 NVRAM / 恢复出厂才生效。

7. **`CONFIG_FIRMWARE_INCLUDE_IMQ` 已废弃**（configure 里整段注释），别开。

8. **Config (NVRAM) 分区的偏移随机型变，不是固定 `0x30000`**：

   | 机型 | 布局文件 | Config 偏移 | 大小 |
   |---|---|---|---|
   | GHL（含同族 8M/16M/32M SPI-NOR） | `mt7621-spi-*-layout.dtsi` | `0x30000` | 64 KB |
   | MI-R3G | `mi-r3g.dts` | `0x5600000` | 512 KB |
   | MI-R3P | `mi-r3p.dts` | `0x8600000` | 512 KB |

   后两者走 **NAND** 布局，偏移差两个数量级 —— 跨机型顺手套用同一个数字必然错。
   共同点：**刷固件都不动这个分区**。所以跨板刷机（例如刷 GHL 固件前装过 K2P 的）残留的是旧板的 NVRAM，WAN 口映射之类可能就是旧板的值 → **必须清 NVRAM / 恢复出厂**。

---

## 八、验证产物（不上机）

```bash
TRX=Logan-GHL-4.4-mipsel-linux-musl-<日期>.trx

# 1) 容器格式：应是 U-Boot uImage（Padavan 的 .trx 不是 HDR0）
xxd -l 64 "$TRX"                       # 头 4 字节 27 05 19 56
# 0x24 附近应有 "GHL"

# 2) 解内核，验 DTS 改动是否进镜像
dd if="$TRX" bs=1 skip=64 | xz --format=lzma -dc --single-stream > kernel.bin
strings -n 6 kernel.bin | grep -E "Linux version|ttyS0"   # 应见 console=ttyS0,57600

# 3) 定位 SquashFS（16 字节对齐）
LC_ALL=C grep -abo "hsqs" "$TRX"
dd if="$TRX" bs=1 skip=<偏移> of=rootfs.sqfs

# 4) 验组件是否真的进去
7z l rootfs.sqfs | grep -E "usr/bin/(frpc|xray)|bin/tc|Advanced_SQM.asp"

# 5) 验内核宏（比反编译快）
gh run view <RUN_ID> --repo LoganJinDev/Padavan-4.4-Build --log \
  | grep -oE "BOARD_RAM_SIZE=[0-9]+" | sort | uniq -c   # 应为 512

# 6) 验自定义值
7z e rootfs.sqfs -oout usr/bin/frpc 2>/dev/null; ls -l out/
```

体积校验：
```bash
ls -l "$TRX"    # 必须 ≤ 32,702,464 B (31.19 MiB)
```

---

## 九、一句话总结

| 需求 | 做法 |
|---|---|
| 改 IP / SSID / 密码 | 进 WebUI 改，**不用重编** |
| 让「恢复出厂」带自定义值 | `customization` JSON 重编 |
| 加 FRP 内网穿透 | `frpc=y`（`frps=n`）；`gzip -9` 后 ≈3.85 MiB。**GHL 余量只剩 ≈5.9 MiB，先算再开**（第五节） |
| 控制 LAN 设备网速 | `sqm`+`qos`+`ifb`=y，≈0.4 MiB；用 CAKE 公平队列或自写 tc 脚本 |
| 国内直连/国外代理 | **已具备**，`ss_run_mode=router` 即可，不用重编 |
| 按规则选不同节点 | **已具备**，不用重编 —— `Shadowsocks_Route.asp` 的分流规则页 / ACL 页，每条规则绑一个出站 |
| 改国家码 / 访客网络 / 时区 / NTP / 掩码 | 手改 `defaults.h` |
| 改功能矩阵 | `plugins` JSON 或 `templates/GHL.config` |
