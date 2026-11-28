---
title: "XLX516 G ↔ NRL 桥接全记录：从四条桥搭起来到只留 DMR 一路"
date: 2026-11-28T16:15:00+08:00
tags: [XLX, DMR, YSF, D-STAR, NRL, 桥, SvxReflector, xlxd, AMBE, 业余无线电]
description: "2026 年 11 月 27 日晚到 28 日下午，两天时间把 XLX516 的 G 模块（D-STAR、DMR、YSF）与 NRL 组 1001 接起来，又收窄成 NRL 只走 DMR 一路。这篇是全过程的详细记录：四条桥怎么搭的、D-STAR 为什么无声、YSF 怎么落到 C 模块又修回 G、声码器从 MBEVocoder 换成 MD380 固件与 xombe、DMR 的 TG 过滤与 AGC 修复、D-STAR 的呼号过滤误杀、DMR 的电流声、XLX 2.5.3 把 YSF 的 DN 改成 VW、DMR 语音头的 300 Hz 提示音。每个根因都有包级或源码证据，每个修复都有真机或离线验证。"
---

# XLX516 G ↔ NRL 桥接全记录：从四条桥搭起来到只留 DMR 一路

> 2026 年 11 月 27 日晚，我开始把 XLX516 反射器的 G 模块（D-STAR、DMR、YSF）与 NRL 组 1001（中继互联）接起来。28 日下午收尾，拓扑收窄成 NRL 只走 DMR 一路，DMR 与 YSF 在 XLX 内部原生互通，D-STAR 的转码工作拿掉。这篇是这两天全部工作的详细记录，按时间顺序写，每个根因都有包级或源码证据，每个修复都有真机或离线验证。
>
> 相关背景：NRL 与 FMO 的语音桥是[第 28 篇](/posts/nrl-fmo-bridge-connected/)，BD4TXS 盒子连不上是[第 29 篇](/posts/fmo-bd4txs-emqx-authz/)，FAS 装了半小时就停了是[第 31 篇](/posts/fmo-fas-installed-and-stopped/)。这篇是它们的续集，也是桥这条线的收尾。

---

## 目标与架构

目标：XLX516 的 G 模块（D-STAR / DMR TG 4007 / YSF）与 NRL 1001 组互相通话。纯软件，不用硬件声码器。

架构是中间一个 OPUS 枢纽（SvxReflector 协议 2.0），四条桥挂在同一个通话组 TG 1 上，音频经枢纽互通：

```
XLX516 G 模块 ── D-STAR ── xlx_bridge ──┐
              ── DMR TG4007 ── dmr_bridge ──┤
              ── YSF ── ysf_bridge ──┤      ├─ GeuReflector (TG 1, 127.0.0.1:5300)
NRL 1001 中继互联 ── nrl_bridge ──┘      └─（四条桥同一通话组，互通）
```

桥跑在国内那台阿里云 VPS（nrl-bd4two-site，101.132.21.119）上，与 FMO、EMQX、SAS 同机。桥源码是 audric 的 svx-bridges 系列，nrl_bridge 是我自写的（复用 xlx_bridge 的 SVX 客户端/滤波/AGC）。环境文件在 `/etc/svx-bridges/*_bridge.env`，600 权限，密钥只存在服务器上。

第一个坑在枢纽：桥启动后报 `Incompatible protocol version: 2.0. Should be 1.0.`——Ubuntu 的 svxreflector 19.09.2 只讲协议 1.0，桥讲协议 2.0。换成了 **GeuReflector**（audric/geureflector，commit dbad88e8，从源码编译）。二进制 `/usr/bin/svxreflector`（原备份 `.orig`），配置追加 `[GLOBAL] LOCAL_PREFIX=1` 与 `[SERVER_CERT] COMMON_NAME=nrl-bd4two-site`（GeuReflector 要求，否则启动即 abort）。服务仍叫 `svxreflector`，TCP+UDP 5300 在听。

## 11 月 27 日 21:17：四条桥全部连上

四条桥全部连上，服务 active：

| 桥 | 服务 | 呼号 / 身份 | 状态与证据 |
|---|---|---|---|
| xlx_bridge（D-STAR） | `svx-xlx-bridge` | BD4TWO，DCS 30051 | SVX 鉴权成功，DCS 连上 127.0.0.1:30051，XLX 回 ACK。21:17 有真实 DCS TX（MYCALL=BD4TWO-3，stream 9091）。 |
| dmr_bridge（DMR） | `svx-dmr-bridge` | BD4TWU，DMR ID 4602468，62030，TG 4007，TS 2 | SVX 鉴权成功，DMR 主站鉴权成功（收到 nonce，配置接受）。21:17 有真实 DMR TX（TG 4007 TS 2，stream 0907C537）。 |
| ysf_bridge（YSF） | `svx-ysf-bridge` | BD4TWO-2，42000，YSF_DGID=16 | SVX 鉴权成功，YSF 已连接。21:14–21:17 多次收到 TalkerStart BD4TWO-3 并开始转发。 |
| nrl_bridge（NRL） | `svx-nrl-bridge` | SVX 侧 BD4TWO-3；NRL 侧 BD4TWO，SSID 200，组 1001 | SVX 鉴权成功，NRL 连上 127.0.0.1:60050，切组 1001 成功（枢纽回 "1001中继互联"）。 |

端到端验证（21:14–21:17，真实通话）：NRL 组 1001 里用户 BD4TXS 用 Opus 发话，nrl_bridge 收到后转给 SvxReflector TG 1；三条 XLX 桥全部收到 TalkerStart BD4TWO-3 并开始转发。TalkerStop 同样贯通。BD4TXS 多次发话，每次都贯通。

反向（XLX 侧发话 → NRL）代码与回声抑制（3 秒内忽略自己发出的回声）都在，日志里有回声抑制生效的记录，但 21:17 之前没有 XLX 侧真实用户发话，需真机验证。

## D-STAR 无声：MBEVocoder 的 AMBE 跟真机不是一套东西

21:37，第一个大问题：NRL 发话，D-STAR 电台上能看到呼号 BD4TWO-3，听不到语音。

抓包（`/tmp/dcs_capture.pcap`，DCS 30051，21:37:42–21:37:51，412 帧）：AMBE 帧全在变、不是静音、不是固定重复帧，编码器确实在干活。

**根因：MBEVocoder 的 D-STAR AMBE（2400x1200）跟真机 DVSI 芯片不是一套东西。** 对照实测：全零 PCM 编出来是 `fd0db4e3928895a6f0`，而 xombe 参考实现编静音是 `9e8d3288261a3f61e8`；1000Hz 正弦波编完再解回来，峰值只剩 4392（输入 10000）。它编的 9 字节帧，D-STAR 电台的硬件解码器认不出来，所以呼号能显示、语音出不来。

**更正（23:35）**：早先诊断时断言的“已知 D-STAR 静音帧”`dc8e0a40adedad396e` 很可能是错的——xombe 实测：它 FEC 解出 b0=80（语音帧基频，静音要求 b0=124/125），解出来是越来越大的语音（峰值 3294→22065→32768 削顶），类别 VOICE。正确的 D-STAR 静音帧是 `9e8d3288261a3f61e8`（配 `dstar_dtx=1`，b0=124）。此前“MBEVocoder 把静音帧解出大声”那条证据因此不成立，但 MBEVocoder 坏的结论本身仍立得住（正弦往返、编静音对不上参考实现、真机听不到）。

22:56，我让先停 xlx_bridge（inactive，未 disable），其余桥不受影响。

## YSF 落在 C 模块：修回 G

22:43，第二个问题：XLX 仪表盘上 YSF 桥（BD4TWO-B）显示在 C 模块，不在 G。

根因两条：①旧 xlxd 二进制的 YSF 默认模块就是 C（22:48 日志实锤：新客户端 "added with protocol YSF on module C"）；②ysf_bridge 配置里没有模块项，只发 DGID=0，DGID=0 在 XLX 里意思是“沿用当前已链模块”，永远不会主动把模块改回 G。

XLX 的 YSF 模块判定（源码 `cysfprotocol.cpp`）：语音头 FICH 的 SQ 字段就是 DG-ID，DG-ID 10→A、11→B、12→C、…、16→G；DG-ID 小于 10 则沿用当前模块。

修复（两处都做了）：①我 22:48 重新编译 xlxd（`YSF_AUTOLINK_MODULE='G'`，`YSF_AUTOLINK_ENABLE=1`），二进制 `/xlxd/xlxd` 已更新；②ysf_bridge 环境文件加 `YSF_DGID=16`（= G 模块），桥重启后日志确认 "[YSF] Requested DG-ID 16 via YSFO"。

22:49:44 重启 xlxd（新 PID 66102），四条桥全部自动重连；XLX 日志 22:49:47 确认 "New client BD4TWO B at 127.0.0.1 added with protocol YSF on module G"。

Pi-Star 侧：XLX 不在官方 YSFHosts.txt 里（YSF 注册方明确不收 XLX），要在盒子 `/root/YSFHosts.txt` 手动加一行 `XLX516<TAB>101.132.21.119` 再跑 Update，启动主机才能选到 XLX516。

## 声码器更换：DMR 换 MD380 固件，YSF/D-STAR 换 xombe

23:09，我批准换声码器。DMR 换 MD380 固件声码器，YSF 与 D-STAR 换 xombe。

**DMR → MD380 固件声码器（23:34 完成并验证）**：桥作者 audric 的 wiki 写明 DMR 桥支持可选的 MD380 固件声码器（比内置 MBEVocoder 音质好）。固件 `md380fw.img`（994304 B）与 `md380ram.img`（131072 B）取自 `nostar/md380_vocoder` 的 Makefile 地址，SHA-256 与 wiki 给的值一致（fw `c0351f25…`、ram `d4039818…`）。落盘 `/opt/svx-bridges/firmware/md380/`，环境文件追加 `MD380_FIRMWARE_DIR`。堵点：桥要跑 `/usr/local/bin/qemu-arm-static /usr/local/bin/md380_helper`，两者都没有；helper 源码在 `~/workspace/svx-dash/dmr_bridge/md380_helper.c`，VPS apt 源连不上改在本机 VM 上编，第一版在 qemu 下 abort（Ubuntu gcc 默认 `_FORTIFY_SOURCE=3` 拦了固件定址缓冲），加 `-U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=0` 重编通过。验证：手动跑 helper，全零 PCM 编出 9 字节 `bbea81724272006a6f`（真 MD380 固件静音帧）；桥重启后日志 `[Vocoder] Using MD380 firmware vocoder`，随后配置接受、桥激活、xlxd 确认 DMR 桥重连。

**YSF + D-STAR → xombe（23:49 部署）**：xombe（x0rro/xombe，v0.1.0，MIT，C99，AMBE 2400 与 AMBE+2 编解码）装进 `/usr/local`（`libxombe.a`、`xombe.h`、`xombe_config.h`）。两条桥已换 xombe，MBEVocoder 已摘除：xlx_bridge（AMBE 2400，`dstar_dtx=1`）、ysf_bridge（AMBE+2 2450，默认配置）。新二进制 md5：xlx `38948181…`、ysf `7bcd3eb4…`，strings 里 0 处 mbevocoder、263 处 xombe。旧二进制备份在 `/opt/svx-bridges/bin/bak-20261127-mbe/`。

离线验证（Go test，真实 Vocoder 代码路径）四项全过：D-STAR 全零 PCM 编静音 `9e8d3288261a3f61e8`、解该帧峰值 ~0、1000Hz 正弦往返峰值 ~10087 波形正确；YSF 稳态静音帧 `f802b95f84d880`、解全零峰值 ~0、1000Hz 正弦往返峰值 ~10177 波形正确。flush 已加（停话时把编码器流水线残留帧刷出来，D-STAR 4 帧、YSF 5 帧一包）。

23:54:04 自然发生的 NRL 发话（BD4TWO-3，6 秒）：三桥同时起 TX；XLX 侧收到 DCS 语音包并开/关 G 模块流，xlx_bridge 收到回声流并正确忽略；ysf_bridge TX 起停正常。NRL→D-STAR 与 NRL→YSF 的转发链路在 xombe 下已贯通。但真机有没有声音仍未证实，需真机听。

## 11 月 28 日凌晨：三处修复

**NRL 帐号核查（00:05）**：三条桥不需要各建 NRL 帐号，共用 svx-nrl-bridge 的 BD4TWO-200 就够了。设备库 SQLite `/nrllink/udphub.sqlite3`（devices 表），未知设备自动入库；hub 发语音只查发送者 status；按呼号+SSID 认、不按 DMRID。BD4TWO-200（id 581）与 BD4TWU-200（id 693）都在组 1001、status 0、priority 100、DMRID 都是 0。

**DMR→NRL 的 TG 过滤修复（00:12）**：根因——XLX 发往 DMR 客户端的帧目的 TG 全写死 9（反射器设计：模块由客户端链定，不由 TG 区分），桥 `dmr_client.go` 的接收过滤只认 TG 4007，XLX 转发的帧全被静默丢掉。修复：过滤改成同时接受 9（发送仍用 4007 才能进 G 模块），加 debug 日志。备份在 `/opt/svx-bridges/bin/bak-20261128-tg9/`。教训：桥重启后未链定 G 模块，须先从 NRL 说一句话让桥链上 G，再从 DMR 发。

**D-STAR 呼号过滤修复（00:49）**：根因——桥把我自己的发射当回声丢了。桥的 DCS 呼号是 BD4TWO，我电台 MYCALL 也是 BD4TWO，`dcs_client.go` 的呼号过滤把 72 帧真语音（00:24:24，stream 6452）静默丢弃；桥已有 stream ID 比对加 isSvxTalking 加 3 秒回声窗两道防回声，呼号过滤是多余的误杀。修复：删掉呼号过滤、只留 stream ID 比对；静音常量改成 `9e8d3288261a3f61e8`。新二进制 md5 `270f207a…`，旧备份 `/opt/svx-bridges/bin/bak-20261128-callsign/`。NRL→D-STAR 桥确实在发真语音（抓包 93.7% 是语音帧），电台只显示呼号不出声，剩余怀疑是 XLX→热点转发、热点/电台设置、或 xombe/DVSI 兼容性（未证实）。

**DMR AGC 按 D 方案修复（00:50 拍板，00:54 完成）**：根因——BR4SN 的 DMR 语音解码是对的、就是太轻（MD380 解 RMS 234，正常约 900），桥 `agc.go` 把峰值 < 0.01 的帧当静音跳过（114 帧里 87 帧），增益只爬到 1.68 倍，输出 RMS 326。仿真否决了原定方案 B（静音帧乘当前增益仅 327）；真正瓶颈是 attack 0.01/帧太慢。**D 方案**：attack 0.01→0.05、静音阈值 0.01→0.002 且轻帧仍乘当前增益（maxGain 4、限幅器 0.9 不动）。真实 Go 代码仿真：轻语音 RMS 326→**851**、纯静音仍 0、正常音量约 1839、无削顶。新二进制 md5 `2d3b3776…`，旧备份 `/opt/svx-bridges/bin/bak-20261128-agc-d/`。

00:19 真机确认：DMR→NRL 有信号但只有噪音（TG 过滤修好了，帧能过来）；NRL→DMR 有延时但声音正常、能听清。00:40 抓包加离线解码实锤噪音根因是音量（BR4SN 那次发射解出来是对的、就是太轻，RMS 234，桥里 AGC 把 76% 的帧当静音跳过、增益只爬到 1.68 倍，送到 NRL 的 RMS 只有 326，轻到听不清）。

## 11 月 28 日 14:22：新拓扑，NRL 只走 DMR 一路

14:22，我提出新思路：**不要 YSF、DMR、D-STAR 三个都往 NRL 发；在这三个桥中选一个语音转 Opus 最稳定的，先把 YSF、DMR、D-STAR 三种数字互通搞定。** 14:26 确认：

- **NRL 只走 DMR 一路**（DMR 最稳：MD380 固件声码器，NRL→DMR 很清晰，DMR→NRL 音量已修好 RMS 326→851）。
- **DMR 与 YSF 在 XLX 内部不需要转码、本身就通**（同为 AMBE+2，XLX 原生互通）。
- 剩下研究 D-STAR 转 DMR，能通就都通了。

DMR↔YSF 最优方案研究（14:48）结论是已在用的 **XLX 原生跨模式转发，不用换**——DMR 与 YSF（DN）共用 AMBE+2，XLX 只重组帧不解码，音质逐比特无损、零附加延迟、无 AGC、无杂音，是音质天花板；全网共识是 DMR↔YSF 根本不应该经过任何一次转码（来源：n6jet.com、pistar.uk、ve1cra.net 实测）。当前真正要做的是把 Opus 桥接路径从 DMR↔YSF 链路上拿掉（停 ysf_bridge），NRL 仍只走 DMR 桥。Opus 桥那条路音质明显差（两次声码器串联加 Opus 16kbps 加双向 AGC，DMR 杂音正是其产物；延迟单向 150–500ms）。MMDVM YSF2DMR 与 XLX 原生同级无损但是重复建设；DVSwitch 过度设计；Pi-Star 增强版 xlxd（帧内电平归一化 YSF→DMR +10dB、DMR→YSF −3dB）是远期备选。

D-STAR 转 DMR 研究（14:33）推荐桥侧 Opus 中转，零改动、只需启动 xlx_bridge；XLX 原生转码必须装 AMBEd（与桥侧同一 xombe、音质无优势）。**14:46 我拍板移除 D-STAR 的语音转码工作**（这步最难，不做了）；语音桥转码只用 DMR 和 YSF；xlx_bridge 保持停用，不再启动验证。

14:50 批准执行推荐方案：① svx-ysf-bridge 已停（inactive）；② DMR 桥 B+C 配置已应用（见下节）；③ 验证待我和 BR4SN 分开做。

## DMR 杂音：AGC 把本底噪声放大了

14:30–14:31 真机验证：**DMR↔NRL 双向语音都通了**。但 14:37 确认一个新问题：**只在互相发射时有电流声，说话/发射时才有，静默时没有，双向都有。**

包级取证排除了三个嫌疑：不是 50Hz 工频哼声（50Hz 成分仅 RMS 的 0.1–0.2%，等于没有；桥内 300Hz 高通滤波器对 50Hz 有约 −31dB 衰减）、不是 AGC 削波（忠实仿真零触限幅）、不是 MD380 静音帧伪音（静默帧 RMS 仅 0.2–2.1，是真静默）。

最可能机制是 **AGC D 方案把电平抬起来后，原本听不见的本底噪声（声码器量化噪声加话筒底噪）在发射期间被一起放大到可闻**：增益在 1.6×–3.3× 之间泵动，语音从 RMS 741 抬到 1724（+7dB），噪声同倍放大；修复前音频太轻（RMS 326）噪声淹没在听阈之下，修复后电平正常了噪声也浮出来了。第二嫌疑点是 **NRL→DMR 输出过热**（RMS 1522–3613、峰值 23022，是 AGC 目标 9830 的 2.3 倍）。

修复 B+C（14:50 已应用）：只改环境文件配置加重启，不用重编。`/etc/svx-bridges/dmr_bridge.env` 加 `AGC_EXT_TO_SVX_MAX_GAIN=2.0`、`AGC_SVX_TO_EXT_MAX_GAIN=2.0`（maxGain 4→2.0 双向，噪声放大从 12dB 降到 6dB）、`AGC_SVX_TO_EXT_TARGET_LEVEL=0.2`（目标电平 0.3→0.2，收 NRL→DMR 过冲）。环境文件备份 `dmr_bridge.env.bak-20261128-bc`；DMR 桥重启（新 PID 113546）。回滚是删三行再重启。无效再做 A（AGC 加噪声门，需改代码）。

## NRL→YSF 没声音：XLX 2.5.3 把 DN 改成 VW

14:37，YSF→NRL 通了，但 NRL→YSF 没声音。电台已确认三遍：收发同一台、DN 模式、AMS 确认关闭。DG-ID 默认 00（收全部），已排除。

包级证据（`/tmp/ysf_1415_test.pcap`）：桥在 14:34:48、14:34:54、14:35:02 三次 TX 都发出去了，XLX 在 eth0 上确实把 155 字节 YSF 帧转发到了热点 49.84.96.129（42000 与 43578 两个端口）。NRL→YSF 在网络层已到热点，电台不出声，断点在热点/电台端。

**根因（14:50，包级加源码双重实锤）：XLX 2.5.3 把 YSF 转发帧的 DT 从 2（DN）改成了 3（VW）。** 桥发出的是正确 DN（DT=2、DGID=16、VoIP=1，FICH 解码证实），XLX 收到后重发给热点时把 DT 改写成 VW，DN 模式的电台解不出 VW 帧，所以没声音。证据：XLX 发往热点与桥的全部 155 字节 YSF 帧（10 个 FICH、约 256 帧）全是 DT=3、VoIP=0、DGID=0；diff v2.5.2（`/usr/local/src/xlxd/`）与 v2.5.3（`/root/xlxd/`）的 `cysfprotocol.cpp`：`EncodeDvHeaderPacket`（672 行）、`EncodeDvPacket`（727 行）、`EncodeDvLastPacket`（819 行）三处 `setDT` 全被上游由 `YSF_DT_VD_MODE2`（DT=2）改成 `YSF_DT_VOICE_FR_MODE`（DT=3）。在跑的 xlxd 是 v2.5.3（我 11-27 22:48 编的；该改动是上游变更，非我重编时引入）。这也解释了 00:21 那次“NRL→YSF 有信号但杂音断续”（VW 电台拿 IMBE 解 AMBE+2 数据出杂音）。DG-ID 已排除；热点双客户端已查清与无声无关（BR4SN B 是残留只发 keepalive，BD4TWO B / 43578 是在用的）。

其他路走不通：电台切 VW 只能单向且断 YSF→NRL；桥侧无 workaround（XLX 重新生成 FICH）；xlxd 无改 DT 的配置项；降回 v2.5.2 会连带回退其他改动、范围更大。

**修复（14:52）**：我亲口放开“xlxd 你可以随便动”后执行——`/root/xlxd/src/cysfprotocol.cpp` 三处改回 `YSF_DT_VD_MODE2`，make 重编（新二进制 md5 `e8fff9f1…`，旧 `0ba64826…`），装到 `/xlxd/xlxd`，重启 xlxd（新 PID 113944）；旧二进制与原源码备份在 `/root/xlxd-backup-20261128-dt2/`。DMR 桥已重连上新 xlxd（UDP 62030 ESTAB）。只影响 YSF 转发帧的 DT 标注，语音净荷、D-STAR/DMR、模块链接均不动。修好后 NRL→YSF 与 DMR→YSF 应都通；YSF→NRL 不受影响。

## DMR 语音头的哒哒声：300 Hz 提示音与错配的 DMR ID

14:57，我报：DMR 发射时，语音头会有哒哒声。在 NRL 听到、DMR→NRL 方向；NRL→DMR 与 NRL→YSF 都没问题。

桥侧三处都已排除：全链路离线仿真（用桥自己的代码与同一 Opus 参数，对 BR4SN 14:55:28 的 132 帧 AMBE 仿真）开头平滑无爆音；MD380 解码无启动瞬态（同一帧在三初态解出几乎相同）；AGC 无跳变（两桥 AGC 都在 TalkerStart 复位增益 1.0 平滑爬升）。

**根因（15:07 查清，15:35 挖透）：哒哒声不在桥里，是电台/热点那边发出来的真实突发爆音，被桥原样传到 NRL。** 爆音是干净的 **~300 Hz 单音正弦提示音，持续约 80ms（第 2–5 帧）**，不是按键机械声、不是宽带噪声。FFT 能量集中 281–336 Hz、峰值 297–305 Hz，波形平滑周期正弦、峰值因数约 2.0。是电台按 PTT 时 DSP 生成的固定提示音（talk-permit/按键提示音），且这台电台在提示音期间不静音话筒，所以提示音被原样发出来了。**哒哒声 100% 只出现在 src=4602468 的传输里（4/4 全有），src=4601256 的传输全干净（0/2）。** 爆音传输的头 3 帧 AMBE 逐字节相同，干净传输的头 3 帧也逐字节相同。爆音之后语音完全正常，4602468 的语音反而比 4601256 的更响更清晰。

旧问题非新问题：之前 DMR→NRL 太轻（RMS 326）把爆音压在听阈下，AGC D 方案抬到正常电平后爆音跟着浮出来（与杂音同一机制）。

**热点 DMR ID 已证实错配**：热点发出的语音帧源 ID 是桥的 DMR ID（4602468，BD4TWU），不是 BR4SN 的。dmrid.dat 里 4601256 登记为 BD4TWO（我），4602468 登记为 BD4TWU（桥），热点自报呼号是 BR4SN，但帧里用的全是别人（我/桥）的 ID——BR4SN 真正的 DMR ID 在数据里找不到。最可能场景是两台不同电台：一台 4601256（干净，应该是我的）、一台 4602468（BR4SN 的，错用桥 ID、提示音开着且会发出）。品牌无法仅凭 300 Hz 音调定死，须看电台实物。

给 BR4SN 的清单：① 报 4602468 那台电台品牌/型号；② 关提示音（MD-380 菜单 Utilities→Radio Settings→Tones/Alerts→Talk Permit 设 Off；D878UV Menu→Settings→Sound→Talk permit tone 关；摩托罗拉 CPS 信道 Talk Permit Tone 取消；海能达 CPS 通话允许提示音关；同时确认 Keypad Tones 与 PTT Sidetone）；③ 核对 DMR ID（热点 Pi-Star Configuration→General→DMR ID、各台电台 CPS 的 DMR ID/Radio ID；任何一台都不能是 4602468，BR4SN 自己的正确 ID 查到后改回）；④ 改完连续发 3–5 次验证，抓包 `/tmp/dmr_tick.pcap`（PID 114401）在跑对照。

## 收尾状态（2026-11-28 15:52）

| 服务 | 状态 | 说明 |
|---|---|---|
| svx-dmr-bridge | active（PID 113546） | DMR 唯一 NRL 通路。MD380 固件声码器。B+C 配置已应用。 |
| svx-nrl-bridge | active（PID 58844） | NRL 侧 BD4TWO-200，组 1001。 |
| svxreflector（GeuReflector） | active | OPUS 枢纽，TCP+UDP 5300，TG 1。 |
| xlxd（XLX516） | active（PID 113944） | DT 修复版。 |
| svx-ysf-bridge | **inactive（已停）** | DMR↔YSF 改走 XLX 原生。 |
| svx-xlx-bridge | inactive（已停） | D-STAR 转码已移除，不再启动。 |

已验证通的：DMR↔NRL 双向语音（14:30–14:31 真机）；YSF→NRL（14:37）；NRL→D-STAR/NRL→YSF 的转发链路在包级已贯通（真机声音待验）。

待复验（等我和 BR4SN 分开做）：DMR↔YSF 直连（XLX 原生）、YSF→NRL 经 DMR 桥、NRL→YSF 经 DMR 桥（依赖 xlxd DT 修复，已装）、杂音复测、哒哒声复测（BR4SN 修完后连续发几次）。

## 教训

这几天反复验证的几条，写下来免得以后再踩：

- **文件级验证全过不等于前提对。** 黑块修复版 v9fix 离线全过、真机却让 2DR 点不进菜单；这里也一样，声码器离线验证全过、真机听不到。显示类、声音类问题只认真机验收。
- **keepalive 超时移除是误诊。** 每次 xlxd 或桥重启，必然在约 30 秒后产生一条旧客户端超时移除日志——那是上一次进程重启后留下的旧客户端对象，不是当前在跑的桥。判断客户端死活，看 xlxd.xml 里该呼号的 ConnectTime 是否为当前进程启动时间。
- **`ambed openstream timeout` 是无害噪音。** XLX 开流时找没装的 AMBEd 转码，超时后跳过；同模式转发不需要转码。不要装 AMBEd。
- **电平正常化后，原本听不见的东西会被听见。** AGC 把 DMR 音量从 RMS 326 抬到 851 之后，杂音和哒哒声一起浮出来了——它们不是新问题，是旧问题第一次被听见。修音量之前，先想想音量下面藏着什么。
- **上游改动要 diff。** XLX 2.5.3 把 YSF 的 DT 从 2 改成 3，是上游源码里的三行改动，不看 diff 永远想不到。升版本先 diff 再编。

桥的部署说明在 `~/workspace/vps-maintenance/xlx-g-nrl-bridge/README.md`，已同步到最终状态。取证脚本与抓包在 VPS `/tmp/`（`extract_dmrd.py`、`decode_analyze.py`、`agc_sim.py`、`tick_analyze.py`、`deep_analyze.py`、`dmr_tick.pcap` 等）。
