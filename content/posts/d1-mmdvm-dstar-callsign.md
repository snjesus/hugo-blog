---
title: "Newifi D1 D-STAR 呼号排查：N0CALL 改完了，配置就干净了吗"
date: 2026-11-20
---

上一篇 [Newifi D1 MMDVM 重编译实录](https://blog.bd4two.eu.org/posts/d1-mmdvm-cflags-rebuild/) 末尾留了个"附加谜题"：D-STAR 日志里的源呼号是 `N0CALL` 而不是 `BR4SN`，根因是 `/etc/ircddbgateway` 这个上游默认配置文件里全是占位符，`sed` 把 5 处 `N0CALL` 换成 `BR4SN` 就结案了。

昨天改完之后，今天早上又把这套配置翻了一遍，越看越觉得"结案"两个字用早了。这篇补记四件事：呼号到底住在哪几个地方、那 5 个键分别管什么、反射器上每小时整点准时冒出来的 `N0CALL` 到底是什么、以及那份配置其实还剩下哪些没清干净的占位符。

设备还是那台 Newifi D1（ramips/mt7621，OpenWrt 24.10.5），呼号 BR4SN，DMR ID 460042，D-STAR 走 XRF055 C 反射器。

## 呼号住在两个文件里，不是一个

D-STAR 链路上一共有两套配置，各自定义了一遍身份。

`/etc/MMDVM.ini` 的 `[General]` 段管 MMDVMHost 自己的身份，它决定日志开头那几行：

```
I: 2026-11-20 04:26:43.724     Callsign: BR4SN
I: 2026-11-20 04:26:43.725     DMR ID: 460042
```

`/etc/ircddbgateway` 管 IRC-DDB Gateway 自己的身份，它决定反射器上把我们认成谁。昨天那次 `sed` 改的是后者。

问题在于，这俩**没有任何机制保证一致**。改了一个忘了另一个，日志里明明写着 `BR4SN`，反射器上却挂着一个别的呼号，谁也说不清是哪边错了。

而且两份配置不是各写各的，它们是靠一对 TCP 端口接起来的。MMDVMHost 启动时把自己那一侧的参数原样打出来：

```
I: 2026-11-20 04:26:43.701 D-Star Network Parameters
I: 2026-11-20 04:26:43.701     Gateway Address: 127.0.0.1
I: 2026-11-20 04:26:43.701     Gateway Port: 20010
I: 2026-11-20 04:26:43.702     Local Port: 20011
```

`Gateway Port: 20010` 对应 `/etc/ircddbgateway` 里的 `hbPort=20010`，`Local Port: 20011` 对应 `repeaterPort1=20011`。**呼号在左边那个文件，链路在右边那个文件，中间靠端口对接。** 排查时这两头都要看，只看一头就会觉得"配置明明是对的"。

## 五个键，分别管什么

上一篇只说了"5 处占位符全替换"，没解释每处对应哪条行为。补上这张表：

| 键 | 管什么 | 日志里的证据 |
|---|---|---|
| `gatewayCallsign` | 网关节点自身的呼号，用于 CCS / DCS 注册 | `CCS: BR4SN  B connected to server ccs706.xreflector.net` |
| `repeaterCall1` | 本地中继身份，反射器上显示的就是它 | `Adding BR4SN B to the cache as a local repeater` |
| `ircddbUsername` | 向 `dns.xreflector.net` / `auth.dstargateway.org` 注册用的名字 | `Registered with dns.xreflector.net using callsign BR4SN` |
| `dplusLogin` | D-Plus 链路登录名 | `D-Plus ACK message received from XRF055 C` |
| `url` | 中继资料里的呼号查询链接 | 网关日志不打印，随中继资料发布到反射器 |

看日志的先后顺序，正好能把这五个键的分工看得很清楚：

```
M: 2026-11-19 14:54:27 Adding BR4SN B to the cache as a local repeater
M: 2026-11-19 14:54:27 Linking BR4SN B at startup to XRF055 C
M: 2026-11-19 14:54:28 D-Plus ACK message received from XRF055 C
M: 2026-11-19 14:54:28 D-Plus link to XRF055 C established
M: 2026-11-19 14:54:28 Registered with dns.xreflector.net using callsign BR4SN
M: 2026-11-19 14:54:28 Registered with auth.dstargateway.org using callsign BR4SN
M: 2026-11-19 14:54:29 CCS: BR4SN  B connected to server ccs706.xreflector.net
```

`repeaterCall1` 决定前两行（本地中继身份），`dplusLogin` 决定中间两行（D-Plus 链路），`ircddbUsername` 决定后面两行（DNS 目录注册），`gatewayCallsign` 决定最后一行（CCS 集群）。**这五个键两两独立**，漏改任何一个都只会坏掉其中一条链路，其余四条照样工作。

这正是它难查的原因：坏的那条路和好的那几条长得一模一样。你看到 CCS 通了、D-Plus 也通了，就以为配置整体没问题，但真正影响反射器上身份的那一环可能是断的。

所以 `sed -i "s/N0CALL/BR4SN/g"` 这种全局替换在上游默认配置上是安全的，前提是你确认了文件里所有 `N0CALL` 都该换成同一个呼号。这次确实如此，因为这个文件里只有呼号占位符用 `N0CALL`。

## 怎么证明改对了

改完别急着收工。用三个层次的证据互相印证，层次越靠后越可靠。

**第一层：差异本身就是证据。** 改前我留了 `/etc/ircddbgateway.bak-n0call`，它就是上游出厂原文。拉下来一比，5 个键、10 行差异，不多不少：

```diff
3c3
< gatewayCallsign=N0CALL
---
> gatewayCallsign=BR4SN
13,14c13,14
< url=http://www.qrz.com/db/N0CALL
< repeaterCall1=N0CALL
---
> url=http://www.qrz.com/db/BR4SN
> repeaterCall1=BR4SN
96c96
< ircddbUsername=N0CALL
---
> ircddbUsername=BR4SN
105c105
< dplusLogin=N0CALL
---
> dplusLogin=BR4SN
```

**第二层：日志里每条链路都换成了新呼号。** 就是上面那 7 行。

**第三层，也是最有说服力的一层：真实流量的形态变了。** 改之前，MMDVMHost 收到的 D-STAR 流量都带一层反射器后缀：

```
D-Star, received network header from N0CALL /INFO to CQCQCQ via XRF055 C
```

改之后，这个后缀**整个消失了**。今天 00:00 到 08:18 这段，MMDVM 日志里 `via XRF055` 出现 **0 次**：

```
M: 2026-11-20 02:38:31.208 D-Star, received network header from BH1CTS  /AMBE to CQCQCQ
M: 2026-11-20 03:00:19.402 D-Star, received network header from LX6FB   /705  to CQCQCQ
M: 2026-11-20 06:04:06.748 D-Star, received network header from LX6FB   /M    to CQCQCQ
```

这不是"显示变好看了"，是**路由路径变了**：反射器现在能直接匹配到 `BR4SN B` 这个中继，不再需要退回到"经由某个 repeater 转发"的模式。这条才是判断身份是否真正生效的硬指标，比日志里打印什么都可靠。

## Links.log 是启动快照，不是实时状态

上一篇把 `Links.log` 当成改好的证据引用了。这次仔细看，这个说法得修正。

`/var/log/mmdvm/Links.log` 全文只有一行：

```
2026-11-19 06:54:27: DPlus link - Type: Dongle Rptr: BR4SN  B Refl: XRF055 C Dir: Outgoing
```

时间戳是 UTC（+8 是 14:54），正好是改配置那一次网关启动的时刻。

为什么只有一行？因为这个网关进程从昨天 14:54 起就**再没重启过**。这个结论不是看日志猜的，是算出来的——`/proc/<pid>/stat` 第 22 个字段是进程启动时间（jiffies），和 `/proc/uptime` 相减就是进程年龄：

```
进程年龄 ≈ 17.4 小时
系统 uptime ≈ 14.5 天
相减 = 昨天 14:54
```

所以 `Links.log` 的语义是：**每次网关启动时追加一条链路记录**。它是启动快照，不是实时状态表。今天（11-20）网关日志里从 00:00 到 08:00 有 11 次 `Registered with dns.xreflector.net using callsign BR4SN`，但 `Links.log` 一个字没多——因为网关没重启，它没有理由再写一行。

用错这个文件会得出很离谱的结论。有人看到 `Links.log` 好几个小时没更新，以为链路断了，其实链路一直好着。反过来，拿它当"当前状态"的凭据，看到的其实是十几小时前某一次启动的残留。

判断链路现况，看这两个更实时的：

```sh
# 网关侧的实时注册与连接状态
grep "Registered with dns.xreflector.net" /var/log/mmdvm/ircDDBGateway-$(date +%F).log | tail -3
grep "CCS: .* connected" /var/log/mmdvm/ircDDBGateway-$(date +%F).log | tail -3

# MMDVMHost 侧的 D-STAR 实际收发
grep "D-Star" /var/log/mmdvm/MMDVM-$(date +%F).log | tail -10
```

## 每小时整点那 5 帧 N0CALL

改完呼号之后，反射器每小时整点的自测帧还是收不到。上一篇的结论是"那是别人家没改呼号的设备，不是我们的残留"，这个结论没错，但当时只用了"等半小时看它消不消失"这种被动办法。

这次统计了一下规律，发现它比想象的规整得多。网关日志里 `Incoming G2 header from N0CALL to unknown repeater - N0CALL B` 的**按小时计数**：

| 日期 | 各小时计数 |
|---|---|
| 2026-11-19 | 15 时 5、16 时 5、17 时 5、18 时 5、19 时 5、20 时 5、21 时 5、22 时 5、23 时 5 |
| 2026-11-20 | 00 时 5、01 时 5、02 时 5、03 时 5、04 时 5、05 时 5、06 时 5、07 时 5、08 时 5 |

连续 18 个小时，每小时整点，一帧不多一帧不少。这不是随机流量，是反射器自己的定时自检。

`unknown repeater - N0CALL B` 这个措辞值得琢磨：网关收到一帧，头部声明它来自 `N0CALL B` 这个中继，但网关自己的中继表里没有这条记录（我们的是 `BR4SN B`），所以认不出来，只能记成 "unknown repeater"。如果这个 `N0CALL B` 是我们改名前的自己，它会在某个时刻自己消失；18 个整点不差地继续来，就排除了这个可能。

**判定方法可以固化成一条：改呼号之后，观察异常自测帧的计数是否还在按原周期增长。增长 = 别人的设备；停 = 自己的残留。** 这比"等半小时看它消不消失"快得多，也确定得多。

至于影响：真实电台流量完全不受影响。今天 00:00–08:18 一共收到 8 个 D-STAR 网络头、7 次完整收发，丢包情况是：

| 电台 | 模式 | 时长 | 丢包 | BER |
|---|---|---|---|---|
| BH1KZQ | /ID50 | 0.8 s | 0% | 0.0% |
| BH1CTS | — | 1.4 s | 1% | 8.3% |
| BH1CTS | /AMBE | 2.7 s | 0% | 0.0% |
| BH1CTS | — | 2.2 s | 0% | 8.3% |
| BH1CTS | /AMBE | 6.1 s | 0% | 0.0% |
| LX6FB | /705 | 0.4 s | 4% | 0.0% |
| LX6FB | /M | 0.3 s | 0% | 0.0% |

另外还有 1 次网络看门狗超时：

```
M: 2026-11-20 03:00:21.266 D-Star, network watchdog has expired, 1.2 seconds,  98% packet loss, BER: 0.0%
```

这一条是同一个电台连发两个网络头、第二个没等到正常结束就被看门狗掐掉的，不是"丢包 98%"，别被这个数字吓到。

D-STAR 的 `/TIME` 自测帧和真实语音流量在网关上走的是两条路，认不认得那个 `N0CALL B` 中继，不影响语音帧的转发。

## 但配置其实没干净

上面全是"改对了"的证据。把上游出厂原文重新打开，还剩几处占位符没动。

**`/etc/ircddbgateway` 里：**

```ini
description1=Nowhere
description2=Somewhere
...
description1_1=Nowhere
description1_2=Somewhere
```

`description1` / `description2` 是中继发布出去的描述文字，`description1_1` / `description1_2` 是第一条中继自己的描述。这四处还是 `Nowhere` / `Somewhere`，昨天那次 `sed` 只清了 `N0CALL`。

**`/etc/MMDVM.ini` 的 `[Info]` 段里：**

```ini
Latitude=0
Longitude=0
Height=0
Location=Nowhere
Description=Multi-Mode Hotspot
URL=https://www.qrz.com/db/BR4SN
```

`URL` 那行是对的（呼号已经填了 BR4SN），但 `Location=Nowhere` 和三个坐标全 0 是出厂值。MMDVMHost 每次启动都会把自己的身份块打进日志，所以这个 `Nowhere` 每天都能看见一次：

```
I: 2026-11-20 04:26:43.726     Latitude: 0.000000deg N
I: 2026-11-20 04:26:43.726     Longitude: 0.000000deg E
I: 2026-11-20 04:26:43.726     Height: 0m
I: 2026-11-20 04:26:43.726     Location: "Nowhere"
```

更有意思的是**两个文件的坐标互相矛盾**。`/etc/ircddbgateway` 里第 1 条中继的坐标是设过的：

```ini
latitude1=22.543
longitude1=114.037
agl1=3.000
```

而 `/etc/MMDVM.ini` 的 `[Info]` 里还是 0。两份配置描述的是同一台设备，一个有坐标一个没有。所以严格说，现在反射器看到的是"一个叫 BR4SN B、坐标在 22.543/114.037、描述写着 Nowhere 的中继"——信息是拼起来的，来源还互相打架。

教训很直接：**上游配置文件里的占位符是一整套，不是一处。** 昨天用 `grep N0CALL` 定位，只覆盖了用 `N0CALL` 填的那些；用别的词填的占位符（`Nowhere` / `Somewhere` / 全 0 坐标）就漏了。下次装完这类东西，把整份文件里"看起来像默认值"的行都过一遍：

```sh
# 出厂占位符与可疑默认值一次扫出来
grep -nE 'N0CALL|Nowhere|Somewhere|^[^#]*=0(\.0+)?$' /etc/ircddbgateway
```

## 两个密码，性质完全相反

同一份文件里还有两个密码字段，上游出厂就没改过。这次是靠改前备份和现文件对比确认的——只比对是否相同，不看具体值：

```
remotePassword : 与上游出厂值完全相同（未被修改）
ircddbPassword : 与上游出厂值完全相同（未被修改）
```

**这俩不能一视同仁。**

`ircddbPassword` 是向公共 IRC-DDB 网络注册时用的字段，它出厂就是空的，现在也是空的——而且注册一直成功（`Registered with dns.xreflector.net using callsign BR4SN` 每小时都在刷）。也就是说对公共网络来说这个字段本来就"不设口令"，**保持原样，别去动它**。

`remotePassword` 是另一回事。它配的是本机的远控口：

```ini
remoteEnabled=1
remotePort=10022
```

这是"每台装了 IRC-DDB Gateway 的设备都一样的出厂值"，属于必须改的那类。实事求是地说，当前**没有实际暴露**——我查了网关进程的全部监听端口：

```
tcp   0  0  0.0.0.0:20011  0.0.0.0:*  LISTEN  13649/ircddbgateway
```

只有 `20011`（本地中继口）在听，`10022` 没有处于 LISTEN 状态。所以这是个**配置层面的隐患，不是正在开着的后门**。但配置写着 `remoteEnabled=1` 而口令是公开的出厂值，等于把一道锁挂在那儿，等哪天换个版本就自动上锁。改掉它没什么副作用，值得改。

## 顺手确认的一件事：MMDVMHost 自己重启

今天 04:26:41，`MMDVMHost` 自己重启了一次：

```
M: 2026-11-20 04:26:41.649 MMDVMHost-20221211-OSTAR is starting
```

网关**没有**跟着重启（它从昨天 14:54 活到现在）。按上一篇的结论，`ircddbgatewayd` 只在自身启动时推一次 link 状态，所以今天 MMDVM 日志里 `D-Star link status` 出现 **0 次**：

```sh
grep -c "D-Star link status" /var/log/mmdvm/MMDVM-2026-11-20.log
# 0
```

这等于在全新实例上又验证了一遍上一篇那条结论：**每次重启 MMDVMHost 后若没有 link status，顺手重启一次 `/etc/init.d/ircddbgateway`。**

不过这次不用重启网关，链路本来就是好的——MMDVMHost 起来之后 D-STAR 流量照常进来：

```
M: 2026-11-20 06:04:06.748 D-Star, received network header from LX6FB   /M    to CQCQCQ
M: 2026-11-20 06:04:06.964 D-Star, received network end of transmission from LX6FB   /M    to CQCQCQ  , 0.3 seconds, 0% packet loss, BER: 0.0%
```

今天凌晨还遇到过一次 CCS 的短暂断链：

```
M: 2026-11-20 05:41:16 CCS: Connection has failed (poll inactivity) for BR4SN  B, reconnecting
M: 2026-11-20 05:43:24 CCS: BR4SN  B connected to server ccs706.xreflector.net
```

两分钟自愈。**这类"自己会恢复"的断链和"呼号写错"长得很像，但特征完全不同**：呼号写错是永久性的，重启不会变好，自愈更不可能；poll inactivity 是间歇的，会自己回来。看到断链先别急着翻配置，看它会不会自己好。

## 排查清单

下次再遇到"反射器上呼号不对"，按这个顺序走，别跳步：

| 步骤 | 命令 | 判断 |
|---|---|---|
| 1. 确认网关读哪个文件 | `ls -l /proc/$(pidof ircddbgatewayd)/cwd` | cwd 是 `/`，读的就是 `/etc/ircddbgateway` |
| 2. 数剩余占位符 | `grep -c N0CALL /etc/ircddbgateway` | 应为 0 |
| 3. 比对两处呼号 | `grep -E '^Callsign=' /etc/MMDVM.ini` 与 `grep -E '^(gatewayCallsign\|repeaterCall1)=' /etc/ircddbgateway` | 两边必须一致 |
| 4. 看注册用的呼号 | `grep "Registered with dns.xreflector.net" /var/log/mmdvm/ircDDBGateway-$(date +%F).log \| tail -3` | 应是你自己的呼号 |
| 5. 看真实流量形态 | `grep -c "via XRF" /var/log/mmdvm/MMDVM-$(date +%F).log` | 正常应为 0（反射器直接匹配到本机中继） |
| 6. 判定异常自测帧归属 | 统计每小时 `Incoming G2 header from N0CALL` 的条数 | 仍在按周期增长 = 别人的设备 |
| 7. 清理其它占位符 | `grep -nE 'Nowhere\|Somewhere' /etc/ircddbgateway /etc/MMDVM.ini` | 出厂占位符，一个都不该留 |
| 8. 确认远控口令 | `grep -E '^remote(Enabled\|Port)=' /etc/ircddbgateway` | `remoteEnabled=1` 时必须改 `remotePassword` |

第 5 步最容易被跳过，但它才是真正的判据。

## 经验总结

1. **呼号是几处各写一遍的东西。** MMDVM.ini 和 ircddbgateway 各定义一遍身份，中间靠一对 TCP 端口对接，没有任何一致性校验。改的时候要当成两处都改。
2. **每个占位符管一件事，互不相干。** 五个键坏一个，那条链路坏，其余四条照常工作——所以"部分功能正常"恰恰是最迷惑的状态。
3. **改完要找"行为证据"，不是"打印证据"。** 日志里 `using callsign BR4SN` 是打印，`via XRF055` 后缀消失是行为。行为证据更难骗人。
4. **别把启动快照当实时状态。** `Links.log` 是网关启动时写一行，不是持续更新的状态表。想知道"现在怎样"，去看今天那的日志。
5. **被动等待换主动计数。** "等半小时看它消不消失"→"改呼号后看异常帧是否还按周期增长"，结论从"大概"变成"确定"。
6. **上游占位符是一整套。** `grep N0CALL` 只能找到用 N0CALL 填的那些，`Nowhere` / `Somewhere` / 全 0 坐标得单独扫。
7. **同一个文件里的两个密码字段，性质可能相反。** 一个是公共网络的注册字段（出厂就是空的，别去动），一个是本机远控口令（不改是隐患）。别一把 sed 全替换。
8. **"自己会好的断链"和"改错了的断链"要分开。** 前者间歇、会自愈；后者永久、重启也不变好。先看它会不会自己恢复，再决定翻不翻配置。

## 常用命令速查

```sh
# —— 网关读的是哪个配置文件 ——
ls -l /proc/$(pidof ircddbgatewayd)/cwd

# —— 呼号相关的全部键 ——
grep -nE '^(gatewayCallsign|repeaterCall[0-9]|repeaterBand[0-9]|ircddbUsername|dplusLogin|url)=' /etc/ircddbgateway

# —— 出厂占位符扫描（ircddbgateway + MMDVM.ini）——
grep -nE 'N0CALL|Nowhere|Somewhere' /etc/ircddbgateway /etc/MMDVM.ini

# —— 两处呼号比对 ——
grep -E '^Callsign=|^Id=' /etc/MMDVM.ini
grep -E '^(gatewayCallsign|repeaterCall1|ircddbUsername|dplusLogin)=' /etc/ircddbgateway

# —— 实时链路状态（别看 Links.log）——
grep "Registered with dns.xreflector.net" /var/log/mmdvm/ircDDBGateway-$(date +%F).log | tail -3
grep "CCS: .* connected" /var/log/mmdvm/ircDDBGateway-$(date +%F).log | tail -3

# —— D-STAR 实际收发 ——
grep "D-Star" /var/log/mmdvm/MMDVM-$(date +%F).log | tail -20

# —— 异常自测帧按小时计数 ——
grep "Incoming G2 header from N0CALL" /var/log/mmdvm/ircDDBGateway-$(date +%F).log | awk -F" " '{print $3}' | cut -d: -f1 | sort | uniq -c

# —— 进程年龄（判断"有没有重启过"）——
cat /proc/$(pidof ircddbgatewayd)/stat | awk '{print $22}'
cat /proc/uptime

# —— 段内安全看 ini（不打印密码）——
awk '/^\[D-Star/{f=1;print;next} /^\[/{f=0} f' /etc/MMDVM.ini | grep -viE 'password|passwd|key'
```

老规矩，路由器上没有 `diff`，配置文件比对得拉到 Mac 上做。

---

下一篇大概会写写 Newifi D1 上这套 MMDVM 的日常运维：ID 库和反射器表怎么自动更新、D-STAR 中继怎么切、BM 主站挂了怎么自愈，以及那台 VPS 上跑的转发链路。
