---
title: "Muse Terminal AI Passport"
date: 2026-11-06
---

10 月 10 日开始在 Mac mini 上装一套硬件桥接：把 AI 会话接到一个终端设备上，项目是 muse-terminal-ai-passport（github.com/jonwong0882-source/muse-terminal-ai-passport），把 FoloToy AI Passport 改成访问我自己 Muse 会话的语音终端。装在 BD4TWO 的 Mac mini 上，主目录 /Users/snjesus，macOS 26.5.2，Python 3.13（/opt/homebrew/bin/python3.13）。包和固件都按校验和验过，没问题：最新 Release 是 v1.0.0（唯一发布），ZIP 完整性 OK，SHA256SUMS.json 全部 170 条哈希逐一核对一致，固件 FoloToy-AI-Passport-full.bin 的 SHA-256 是 ef16688d7de00035e94ddc607ec5265f4f1fb8105ae0141e3d4e1fb7b3714fc9，与文档要求一致，是从 0x0 刷写的合并镜像。

装的过程中遇到一个小插曲：桥接代码写死走 Chrome，但我不想装 Chrome（这台只装了 Edge 和 Safari）。解决办法是改一行代码，让它走我已有的 Edge：source/companion/muse_web.py 里 `channel='chrome'` 改成 `channel='msedge'`，改之前原文件做了备份（muse_web.py.orig）。虚拟环境在 `~/Library/Application Support/MuseTerminal/venv`，esptool 5.5.0、aiohttp 3.13.3、faster-whisper 1.2.1、playwright 1.63.0、pyserial 3.5 均在位。

真正卡住的地方在健康检查。桥要找会话页面的输入框来确认就绪，但我给它的地址是个侧边对话（side chat），页面结构和普通主会话不一样，输入框根本不存在，健康检查一直报 muse_ready=false、no_composer（speech_ready=true）。串口和刷固件的步骤必须等桥健康之后再做，顺序不能乱。串口候选有两个：/dev/cu.usbmodem21201 是当日新出现、疑似 Passport，/dev/cu.usbmodem2020_12_222 是既有的，尚未经插拔比对确认，禁止凭猜测刷机。

下一步很具体：在桥专用的浏览器窗口里打开一个普通主会话，发一句"hello"，把新的 muse.ai/thread/ 地址给我，重启桥再验健康。

现状：等这条普通主会话链接，桥接停在我这边。
