---
title: "NRL 只走 DMR 一路：DMR 和 YSF 在 XLX 里原生互通，Opus 桥反而是音质的瓶颈"
date: 2026-11-28T15:30:00+08:00
tags: [XLX, DMR, YSF, NRL, 桥, SvxReflector, AGC, 业余无线电]
description: "11 月 28 日把桥的拓扑收窄了：不要 YSF、DMR、D-STAR 三个都往 NRL 发，NRL 只走 DMR 一路；DMR 和 YSF 同为 AMBE+2，在 XLX 内部原生互通，不需要任何转码，音质逐比特无损。Opus 桥那条路反而是音质的瓶颈——两次声码器串联加双向 AGC，DMR 的杂音就是它产的。DMR↔NRL 通了，但有电流声，根因是 AGC 把电平抬起来后本底噪声跟着被放大到可闻，修复是只改配置把 maxGain 从 4 降到 2.0、NRL→DMR 目标电平从 0.3 降到 0.2。"
---

# NRL 只走 DMR 一路：DMR 和 YSF 在 XLX 里原生互通，Opus 桥反而是音质的瓶颈

> 11 月 28 日下午，桥的拓扑收窄了：NRL 只走 DMR 一路，DMR 和 YSF 在 XLX 内部原生互通，D-STAR 的转码工作直接拿掉。这篇记录为什么这么定、DMR 通了之后那个电流声是怎么回事，以及全网搜索后得出的结论——DMR 和 YSF 之间根本不应该经过任何一次转码。
>
> 前情在[第 28 篇](/posts/nrl-fmo-bridge-connected/)（NRL-FMO 桥打通）和[第 29 篇](/posts/fmo-bd4txs-emqx-authz/)（BD4TXS 盒子连不上）。XLX516 G 模块 ↔ NRL 的四条桥是 11 月 27 日晚搭起来的，这篇是第二天的收尾。

---

## 先说结论

拓扑定成这样：

- **NRL 只走 DMR 一路。** 四个模式里转 Opus 最稳的是 DMR（MD380 固件声码器，NRL→DMR 很清晰，DMR→NRL 音量问题已修好）。
- **DMR 和 YSF 在 XLX 内部原生互通，不需要转码。** 两者同为 AMBE+2 编解码器，XLX 把语音帧从一种协议容器搬到另一种，不解码、不重编，出来的声音与电台发出的逐比特相同。
- **D-STAR 的语音转码工作拿掉。** D-STAR 用 AMBE 2400，是唯一需要转码的例外，也是最难的一步，不做了。xlx_bridge 保持停用，不再启动验证。
- **ysf_bridge 停掉。** DMR↔YSF 改走 XLX 原生，不再经过 Opus 桥。

DMR↔NRL 在 11 月 28 日 14:30–14:31 真机验证双向都通了，但语音有电流声。根因查清了，修复也应用了，等复测。细节在下面。

## 为什么是 DMR

11 月 27 日晚到 28 日凌晨，四条桥都经历了声码器更换：DMR 换成 MD380 固件声码器（23:34），YSF 和 D-STAR 换成开源的 xombe（23:49）。到 28 日下午实测，DMR 是转 Opus 最稳的一路：

- NRL→DMR 很清晰。
- DMR→NRL 之前音量太轻的问题已经修好：根因是桥的 AGC 把峰值低于 0.01 的帧当静音跳过，增益只爬到 1.68 倍，输出 RMS 只有 326。按 D 方案把 attack 从 0.01 提到 0.05、静音阈值从 0.01 降到 0.002，输出 RMS 326→851，纯静音仍是 0，正常音量约 1839，无削顶。11 月 28 日 00:19 真机确认 DMR→NRL 有信号，00:40 抓包证实解码是对的、就是太轻，00:54 D 方案二进制已装好。

YSF 最折腾（VW/DN 模式、热点争用），D-STAR 修好还没验。所以 28 日 14:22 定下来：先把 YSF、DMR、D-STAR 三种数字互通搞定，NRL 只走最稳的 DMR 一路。

## DMR↔YSF 为什么不需要转码

这是 AMBE+2 的功劳。DMR、YSF（DN 模式）、NXDN 6.25k 共用同一个 AMBE+2 语音帧（2450 bps 语音 + 1150 bps 纠错，同一 3600 bps 帧）。XLX 在模块内转发时只做重组帧——把语音帧从 DMR 的容器搬到 YSF 的容器，反之亦然——不解码、不重编。所以：

- **音质逐比特无损**。N6JET 的原话是 "the audio that comes out the far end is bit for bit what your radio produced."
- **零附加延迟**。只做包转发，没有编解码环节。
- **没有 AGC，没有杂音**。语音帧不经过任何 PCM/AGC 环节。

全网的共识一致：DMR↔YSF 根本不应该经过任何一次转码。Pi-Star 的文档也写明："Since YSF uses the same AMBE+2 codec as DMR, no transcoding is needed between them."

唯一要注意的是 YSF 的 VW 模式用的是 IMBE，不是 AMBE+2，所以 VW 过不去——桥只处理 DN（DT=2），VW（DT=3）的帧会被静默丢弃。这点在 28 日凌晨的 YSF 测试里已经反复踩过。

## Opus 桥那条路为什么反而差

我们现在跑的 SvxReflector 桥（audric 那套）走的是另一条路：各模式桥把 AMBE+2 解成 PCM、编成 Opus 上反射器，再由对端桥解 Opus、重编 AMBE+2。两次声码器串联（AMBE+2→PCM→Opus→PCM→AMBE+2）加 Opus 16kbps 中间层加双向 AGC，音质必然有损，延迟单向约 150–500ms。

所以真正要做的不是换方案，而是**把多余的、音质更差的那条 Opus 桥接路径从 DMR↔YSF 链路上拿掉**——停掉 ysf_bridge，让 DMR↔YSF 只走 XLX 原生；NRL 仍只走 DMR 桥。停 ysf_bridge 还有两个顺带好处：NRL 说话时不再有三条桥同时向单模热点发、争用挤断的问题；DMR↔YSF 这条腿不再经过 AGC，AGC 泵动的杂音不会污染它。

其他路线也都比过了：MMDVM 的 YSF2DMR/DMR2YSF 与 XLX 原生同级无损，但是重复建设（XLX 内部已经做了同样的事）；DVSwitch 适合要同时接 AllStar/Echolink/模拟网的多模式大站，对只要 DMR↔YSF 是过度设计；Pi-Star 团队 2025 年底的增强版 xlxd 加了帧内音频电平归一化（YSF→DMR +10dB、DMR→YSF −3dB），是不转码还调音量的唯一解法，但要替换 xlxd 二进制，是远期备选；openSPOT 是单人热点方案，不适合多用户反射器加 NRL 的服务器架构。

## DMR 的电流声：根因是 AGC 把本底噪声放大了

DMR↔NRL 通了之后，11 月 28 日 14:37 确认一个新问题：**只在互相发射时有电流声，说话/发射时才有，静默时没有，双向都有。**

包级取证排除了三个嫌疑：

- **不是 50Hz 工频哼声。** 14:30–14:31 四次发射的 AMBE 帧用 MD380 固件声码器离线解码加 Goertzel 频谱分析，50Hz 成分仅 RMS 的 0.1–0.2%，等于没有；桥内 300Hz 高通滤波器对 50Hz 有约 −31dB 衰减。
- **不是 AGC 削波。** 忠实仿真零触限幅，输出 RMS 1724、峰值 17546。
- **不是 MD380 静音帧伪音。** 静默帧 RMS 仅 0.2–2.1，是真静默，没有直流偏置。

最可能的机制是 **AGC D 方案把电平抬起来后，原本听不见的本底噪声（声码器量化噪声加话筒底噪）在发射期间被一起放大到可闻**。增益轨迹在 1.15 到最高 3.34 之间泵动，字间停顿冻结在 1.79，整段在 1.6×–3.3× 泵动；语音从 RMS 741 抬到 1724（+7dB），噪声同倍放大。修复前音频太轻（RMS 326），噪声淹没在听阈之下；修复后电平正常了，噪声也浮出来了。

还有第二个嫌疑点：**NRL→DMR 输出过热**。RMS 1522–3613，是 DMR→NRL 的 741 的 2–5 倍；14:31:39 那次峰值 23022，是 AGC 目标 0.3×32768=9830 的 2.3 倍。

## 修复 B+C：只改配置，不重编

修复方案是 B+C，最快的一档，只改环境文件配置加重启 DMR 桥，不用重编：

```sh
# /etc/svx-bridges/dmr_bridge.env，追加三行
AGC_EXT_TO_SVX_MAX_GAIN=2.0
AGC_SVX_TO_EXT_MAX_GAIN=2.0
AGC_SVX_TO_EXT_TARGET_LEVEL=0.2
```

- maxGain 4→2.0，双向。噪声放大从 12dB 降到 6dB。
- NRL→DMR 目标电平 0.3→0.2，收 NRL→DMR 的过冲。

回滚是删三行再重启。无效再做 A（AGC 加噪声门，需改代码）：峰值低于语音门限的帧不乘累积增益、改乘小固定增益，只让真语音帧享受 AGC。

11 月 28 日 14:50 按这个执行了：ysf_bridge 已停，DMR 桥 B+C 配置已应用并重启（新 PID 113546，SVX 注册加 DMR 连上 XLX，桥 active），环境文件备份在 `dmr_bridge.env.bak-20261128-bc`。

## 待复验

等我和 BR4SN 分开做，分开验、分开判：

1. **DMR↔YSF 直连**（XLX 原生，不经 NRL）：DMR 电台与 YSF 电台直接对讲各 3–5 秒，双向报音质。预期清晰、无 AGC 泵动杂音、无明显延迟。
2. **YSF→NRL 经 DMR 桥**：YSF 电台发 3–5 秒，我在 NRL 听。
3. **NRL→YSF 经 DMR 桥**：我从 NRL 发 3–5 秒，YSF 电台听。（这步依赖 xlxd 的 DT 修复，已装，见下一篇。）
4. **杂音复测**：对比 B+C 前后 NRL 腿的杂音是否减轻；同时确认 DMR↔YSF 直连全程无杂音。

附带一个未解的异常：桥收到的 DMR 帧源 ID 是 4602468（桥自己的 BD4TWU），不是 BR4SN 的 4601256。`dmr_client.go` 打的是包里字节 5–7 的原始值，不是显示问题。需确认 BR4SN 热点的 DMR ID 配置。这个异常不解释杂音，但要查清——它后来成了另一篇的主角。
