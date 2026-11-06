---
title: "Mac 系统停在 15.8.2"
date: 2026-11-06
---

11 月 4 日给自己定了一条规矩：这台 Mac 的系统停在当前版本，不再往上升。以后凡是涉及系统升级的建议，都先确认，不默认推荐升级。这台是 snjesus 的 Mac mini，当前系统是 macOS 15.8.2（Sequoia，Build 24H232），与 BD4TWO 那台 Mac mini（macOS 26.5.2）是两台不同的机器，不要混淆。

底线先落实。自动更新设置里，"下载可用的更新"和"安装 macOS 更新"都已关掉，"安装来自 App Store 的应用程序更新"和"安装安全响应和系统文件"留着开。自动更新关了，系统就不会自己升，横幅只是碍眼。

光有关自动更新还不够，系统还会推送新版本横幅。办法是装一个 90 天延迟描述文件，让新版本晚 90 天再出现。文件是 `~/workspace/your_files/DelayMajorUpdates90.mobileconfig`，PayloadIdentifier 是 `com.guo.delay-major-updates`，内层类型 `com.apple.applicationaccess`、`forceDelayedMajorSoftwareUpdates=true`、延迟 90 天。第一版没生效：查安装器列表，新版本那行延迟标志还是 NO。2026-11-04 跑 `softwareupdate --list-full-installers`，全部条目 Deferred: NO，含 macOS 27.2（Golden Gate，Build 26B123，12.93GB）、27.0.1、27.0 及 Tahoe 各版。原因是第一版没写 PayloadScope，默认按用户级装，限制类载荷在用户级不生效。改了一版带 PayloadScope System 的，准备重新装。

这里有个坑：15.8.2 上命令行安装描述文件已被废弃，`sudo profiles install` 被系统拒绝，报错"profiles tool no longer supports installs. Use System Settings Profiles to add configuration profiles."，必须走图形界面——系统设置、通用、设备管理里装。流程是先删掉旧的描述文件，再双击新文件，在设备管理里批准，装完重跑安装器列表，看 27.2 那行延迟标志是否变成 YES；若仍 NO，则手动安装的限制类描述文件在此系统上确实不被采纳，放弃藏横幅、只靠关自动更新。

现状：新版描述文件的图形安装结果还没验证。底线已经守住了，系统不会自己升级，剩下的只是让横幅晚点出现。另外提醒自己：Sequoia 装完一次更新后弹出的"以后将自动为你下载安装更新"不要点继续，点了会把自动更新重新打开；新版本横幅上的"今晚升级/立即升级"按钮也别点。还有个边界要记清：描述文件延迟大版本升级上限 90 天、到期需重装，且该做法在 macOS 26 已 deprecated、27 已移除；`softwareupdate --ignore` 自 Big Sur 起已废弃。
