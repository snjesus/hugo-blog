---
title: "黑苹果装机"
date: 2026-11-06
---

11 月 4 日开始折腾黑苹果装机，先是显卡选型，再是硬盘分区，两个都是决定成败的点。

显卡是 RX 560 对 GTX 1050 Ti，两张都是 4G 显存。玩游戏在 Windows 上，1050 Ti 更强；但黑苹果只能选 RX 560——1050 Ti 自从 Mojave 起就没有 NVIDIA Web Driver 了，而 RX 560 是 macOS 原生免驱。选 RX 560 还有讲究：要 1024 个流处理器的满血版，避开 896 个流处理器的阉割版（560D）；避开 XFX 的卡，有 BIOS 兼容问题；蓝宝石 Pulse 相对稳。

接着是装机盘。目标是一块 500GB 的内置盘 disk1，2026-11-05 00:15 我在安装器终端跑 `diskutil list` 发来照片，盘况是：disk1 是内置 500GB，GUID 分区表但确实没有 EFI 分区——容器直接就是 disk1s1，正常应该是 disk1s1 是约 209.7MB 的 EFI、disk1s2 才是容器。盘上已建空宗卷 MacOS，仅 24.6KB，基本是空的。原因是之前抹盘时只抹了宗卷、没动分区表，所以没生出 EFI。修复办法是整盘重抹：磁盘工具里显示所有设备，选中 disk1 整盘，方案选 GUID 分区图、格式 APFS，抹完再跑 `diskutil list`，确认 disk1s1 出现约 209.7MB 的 EFI。MacOS 宗卷是空的，重抹不丢东西。另外 disk3 是内置 1TB，宗卷名"此硬盘不用"，绝对不动；disk4 是外置 8GB U 盘（MBR/FAT32，卷名"1401"），是 1401 工具做的安装盘，已能引导进安装器，不用管。

挂载 EFI 用一条命令：`diskutil mount disk1s1`，挂载点在 /Volumes/EFI，卸载是 `diskutil unmount disk1s1`。安装器终端是 root 权限（提示符 `#`），不用加 sudo，装好的正常系统里要加 sudo。安装 U 盘没有独立 EFI，引导文件在它的 FAT32 分区根目录，挂载是 `diskutil mount disk4s1`。

现状：等整盘重抹后 EFI 是否出现、挂载是否成功的反馈，再往下走。
