---
title: "XLX 2.5.3 把 YSF 的 DN 改成 VW：NRL→YSF 没声音的根因是上游改了三行"
date: 2026-11-28T15:40:00+08:00
tags: [XLX, YSF, xlxd, 源码, 业余无线电]
description: "YSF→NRL 通了，但 NRL→YSF 没声音。包级加源码双重实锤：桥发出的是正确 DN（DT=2），XLX 2.5.3 转发时把 DT 改写成 VW（DT=3），DN 模式的电台解不出 VW 帧。diff v2.5.2 和 v2.5.3 的 cysfprotocol.cpp，三处 setDT 全被上游由 YSF_DT_VD_MODE2 改成 YSF_DT_VOICE_FR_MODE。修复是改回三行、重编、重启 xlxd。DG-ID 和热点双客户端都已排除，不是原因。"
---

# XLX 2.5.3 把 YSF 的 DN 改成 VW：NRL→YSF 没声音的根因是上游改了三行

> YSF→NRL 通了，但 NRL→YSF 没声音。桥在发，XLX 在转发，帧也到了热点，电台就是不出声。这篇记录根因是怎么定的：不是 DG-ID，不是热点双客户端，是 XLX 2.5.3 在转发 YSF 时把 DT 从 2（DN）改成了 3（VW）。修复是改三行源码、重编、重启 xlxd。
>
> 前情在[上一篇](/posts/xlx-dmr-ysf-nrl-topology/)（新拓扑与 DMR 杂音）。这篇是同一天 11 月 28 日的第二件事。

---

## 现象

11 月 28 日 14:37，YSF→NRL 通了，但 NRL→YSF 没声音。

电台已经确认过三遍：收发是同一台电台、DN 模式、AMS（自动模式选择）确认关闭。DG-ID 是默认 00（收全部），DG-ID 不匹配已经排除。

包级证据（`/tmp/ysf_1415_test.pcap`，`-i any`）显示：桥在 14:34:48、14:34:54、14:35:02 三次 TX 都发出去了，XLX 在 eth0 上确实把 155 字节 YSF 帧转发到了热点 49.84.96.129（42000 与 43578 两个端口都发）。也就是说 NRL→YSF 在网络层已经到了热点，电台不出声，断点在热点/电台端。

## 桥发的帧完全规范

先排除桥。用桥自己的 FICH 解码解桥→XLX 的帧（14:30:05、14:34:48、14:35:02 三次 TX）：

| 帧 | FI | FN | FT | DT | VoIP | DGID |
|---|---|---|---|---|---|---|
| 头帧 | 0 | 0 | 6 | **2（DN）** | 1 | **16** |
| 语音帧 | 1 | 0/1/2 | 6 | **2（DN）** | 1 | **16** |

呼号字段 srcGW=BD4TWO、srcRadio=BD4TWO、dst=ALL，均正确。桥的 `BuildYSFD` 按环境文件（YSF_DGID=16）正确编码，VoIP=1（Yaesu 固件靠此位打开网络音频解码）也对。DCH 全零是已知次要偏差，只影响呼号显示，与无声无关。

## XLX 转发时把 DT 2→3、DGID 16→0

包级证据：XLX 发往热点与桥的所有 155 字节 YSF 帧，10 个不同 FICH、约 256 帧，**全部 DT=3、VoIP=0、DGID=0**。14:34:48 窗口逐流对照（同一秒内）：

| 流向 | FICH | DT | DGID | 呼号 |
|---|---|---|---|---|
| 桥→XLX（lo） | `003c5f56…` | 2 | 16 | BD4TWO |
| XLX→热点 42000（eth0） | `213558a7…` | **3** | 0 | BD4TWO |
| XLX→热点 65070（eth0） | `213558a7…` | **3** | 0 | BD4TWO |
| XLX→桥（lo） | `213558a7…` | **3** | 0 | BD4TWO |

源码证据：在跑的 xlxd 是 **v2.5.3**（进程 PID 66102，11-27 22:49:44 启动，`/proc/66102/exe → /xlxd/xlxd`；md5 `0ba64826…` 与 `/root/xlxd/src/xlxd` 逐字节一致，是我 11-27 22:48 从 `/root/xlxd/` 编的）。diff v2.5.2（`/usr/local/src/xlxd/`）与 v2.5.3（`/root/xlxd/`）的 `cysfprotocol.cpp`，**三个 YSF 编码函数里的 `setDT` 全被上游改了**：

- `EncodeDvHeaderPacket`（行 672）、`EncodeDvPacket`（行 727）、`EncodeDvLastPacket`（行 819）：
- v2.5.2：`fich.setDT(YSF_DT_VD_MODE2);`（DT=2，DN）
- v2.5.3：`fich.setDT(YSF_DT_VOICE_FR_MODE);`（DT=3，VW）

也就是说 XLX 2.5.3 转发 YSF 一律标 VW，而语音净荷仍是 AMBE+2（DN 的码）。DN 电台收到 DT=3 直接忽略（没声音）；VW 电台收到则会拿 IMBE 去解 AMBE+2 数据（出杂音）。这也解释了 11 月 28 日 00:21 那次“NRL→YSF 有信号但杂音断续”。注意 XLX 入口校验（行 444）仍只认 DT=2，所以 XLX 收 DN、发 VW，前后不对称。

这个改动是上游变更，不是我 11-27 重编时引入的。`cimrsprotocol.cpp` 三处也同步改了。

## DG-ID 和双客户端都已排除

**DG-ID 已排除。** 电台 DG-ID 是默认 00（收全部），桥发 DGID=16，DG-ID 00 的电台本应收到；而且 XLX 转发时 DGID 字段本来就被清零（DGID=0），DG-ID 不是变量。对照 YSF→NRL 通的那次：电台上行帧解出 **DT=2、DGID=0、VoIP=0**（srcRadio=BD4TWO-2DR，10 个 FICH 全是 DT=2），与桥发的 DT=2/DGID=16 都是 DN——收发两端都是 DN，唯一的断点就是 XLX 中间把 DT 改成了 3。

**热点双客户端已查清，与无声无关。** 热点 49.84.96.129（我自己的）上的 YSF 客户端与源端口对应关系：

| 客户端 | 源端口 | ConnectTime | 状态 |
|---|---|---|---|
| BR4SN B | 42000 | 11-27 22:53 | 残留：只发 keepalive（341 个）、零条上行语音，收 XLX 转发的帧但不接电台 |
| **BD4TWO B** | **43578** | 11-28 14:34:28 | **在用：电台上行（BD4TWO-2DR，28 帧）走这个口，keepalive 14:34:28→14:43:39 连续稳定** |
| （短暂） | 37552 | — | 14:33:56–14:34:02 仅 5 个 keepalive 后消失 |
| （短暂） | 65070 | — | 14:34:07–14:34:26 仅 7 个 keepalive 后消失 |

14:34:33 与 14:34:57 两次“BD4TWO B keepalive 超时移除”，时间上正好是 37552（末次 keepalive 14:34:02 +30 秒）与 65070（末次 14:34:26 +30 秒）两个短暂客户端超时清理，**不是**在用的 43578 那个。热点在 14:33–14:34 之间重连了几次（应是我改 YSF 配置/重启网关），每次重连换个源端口，旧的 30 秒后被清，属正常现象。XLX 往三个端口都发帧，在用的 43578 收到了（14:35:02 收到 22 帧），所以双客户端不导致无声。

另外一个小插曲：XLX→热点 42000/43578 的帧呼号被改写为 BD4TWU（DMR 桥的呼号），往 65070 的保持 BD4TWO。这是 XLX 转发 DMR 语音到 YSF 时的呼号改写（00:18:22 已见过），与 DT 问题无关，不影响解码。

## 热点收到帧后有没有发给电台

VPS 上无法读 Pi-Star/MMDVM 日志（热点在我家里，不在 VPS 上），所以“热点 modem 有没有把帧发射出去”没有直接证据。但三点可定：① 帧已到在用的 43578 客户端（包级证实）；② 电台发射正常（上行 DT=2 到 XLX、YSF→NRL 通），说明电台↔热点射频链路与频率都没问题；③ DN 电台收 DT=3 帧必然静默。因此不需要再假设热点没发——DT 不匹配已是充分解释。若修好 DT 后仍无声，再去热点上看 MMDVM 日志。

## 修复：改三行，重编，重启

把 `/root/xlxd/src/cysfprotocol.cpp` 里三处 `fich.setDT(YSF_DT_VOICE_FR_MODE);`（行 672、727、819）改回 `fich.setDT(YSF_DT_VD_MODE2);`，在 `/root/xlxd/src/` 下 `make`，新二进制装到 `/xlxd/xlxd`，重启 xlxd 服务。改动只影响 YSF 转发帧的 DT 标注，语音净荷、D-STAR/DMR 路径、模块链接均不动。修好后 NRL→YSF（桥 DT=2 → XLX DT=2 → DN 电台解出）、DMR→YSF（同理）都通；YSF→NRL 不受影响（电台本来就发 DT=2，XLX 入口校验仍认 DT=2）。

这件事要动 xlxd，是我之前立的“绝不动 xlxd”硬约束的例外。11 月 28 日 14:52，我亲口放开：“xlxd 你可以随便动。” 约束解除后 14:52 当场执行：三处改完、make 重编（新二进制 md5 `e8fff9f1…`，旧 `0ba64826…`）、装到 `/xlxd/xlxd`、重启 xlxd（新 PID 113944）；旧二进制与原源码备份在 `/root/xlxd-backup-20261128-dt2/`。DMR 桥已重连上新 xlxd（UDP 62030 ESTAB）。

**为什么别的路走不通：**

- 电台切 VW：只能单向通 NRL→YSF，且 YSF→NRL 会断（桥只收 DT=2；VW 用 IMBE，xombe 只有 AMBE+2，解不了）。不可行。
- 桥侧 workaround：没有。XLX 收到帧后重新生成 FICH 再转发，桥控制不了 XLX 发什么 DT。
- 配置项：xlxd 没有改 YSF DT 的配置，只能改源码。
- 降回 v2.5.2：不推荐。会连带回退 2.5.3 的其他改动（EOT 位、描述串等），范围比三行补丁大得多。

## 顺带纠正的两个误诊

这几天查 YSF 时有两个误诊，这次一并纠正，免得以后再踩：

1. **keepalive 超时移除是误诊。** 每次 xlxd 或桥重启，必然在约 30 秒后产生一条旧客户端超时移除日志——那是上一次进程重启后留下的旧客户端对象（进程被杀后不再发 poll，30 秒后正常超时清理），不是当前在跑的桥。判断客户端死活，看 xlxd.xml 里该呼号的 ConnectTime 是否为当前进程启动时间。
2. **`ambed openstream timeout` 是无害噪音。** XLX 开流时找没装的 AMBEd 转码，超时后跳过；同模式转发不需要转码。不要装 AMBEd、不要动 xlxd（这条在 DT 修复前成立，DT 修复是另一回事）。

诊断脚本在 VPS `/tmp/ysfdump.py`、`/tmp/analyze_ysf.py`、`/tmp/analyze_ysf2.py`，本地副本在 `~/workspace/.tmp-ysf/`。
