---
title: "Ubuntu 上 Steam 打不开"
date: 2026-11-06
---

Steam 装在一台 Ubuntu 机器上，10 月 28 日启动失败。这台是联想 ThinkCentre E77（bd4two@bd4two-ThinkCentre-E77），系统是 Ubuntu 24.04 64-bit，不是 Mac mini，跨机器排查必须先写死执行位置，之前就因为默认成 Mac mini 给错过办法。

客户端能打开，但卡在下载更新清单那一步：从 client-update.steamstatic.com 下载更新清单失败，http error 0，报错提示需要联网更新——可机器明明是联网的。

有意思的是，崩溃上报能上传成功，说明一般出网是通的。问题收窄到 Steam 更新服务器这一条线：客户端连不上更新服务器，但别的出网正常。

下一步是查两样东西：终端里的代理环境变量（`env | grep -i proxy`），和对更新服务器地址做一次带超时的连通性检查（`curl -sI -m 10 https://client-update.steamstatic.com/steam_client_ubuntu12`）。代理如果配错了，Steam 这类程序可能走了错误的出口；连通性检查能区分是域名解析、连接还是传输层面的问题。

现状：等这两项检查结果再判。顺带给自己立个规矩：在 Steam 修好之前，商店打折先别买，买了也玩不了。
