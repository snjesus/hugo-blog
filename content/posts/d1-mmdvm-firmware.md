---
title: "Newifi D1 精简 MMDVM 固件"
date: 2026-11-06
---

手里这台 Newifi D1 是我的 MMDVM 数字热点用机，联想 Newifi D1，MT7621AT 双核 880MHz、256MB 内存、32MB 闪存加 128MB NAND、USB 2.0 加 USB 3.0、TF 卡槽。现用固件是定制版"江苏曈宇 2026.10 / MMDVM数字热点"，内核 5.4.255，截图显示内存总量 245.13MiB、可用 156.78MiB。现用固件里塞了不少用不上的插件，11 月 5 日决定自己编一套精简版，只留要用的东西。

规格先定死：全模式都要（D-Star、DMR、YSF、P25、NXDN、POCSAG、FM），热点板走 /dev/ttyS0（板载 UART，115200），配置由我手动填、服务默认不启用。呼号 BD4TWO 预填进 MMDVM.ini 及各网关配置模板的 Callsign 栏，DMR ID、频率、服务器密码留空占位，绝不编造。型号栏显示"Newifi-D1 by BD4TWO"（现固件显示的是 Newifi-D1 by BI4PCS），仪表盘也要带上（DG9VH MMDVMHost-Dashboard PHP 版）。底座用官方 OpenWrt 24.10.5 Image Builder（profile lenovo_newifi-d1），镜像上限 IMAGE_SIZE 32448k，刷机走 Breed，不保留旧配置。固件默认 LAN 地址沿用官方 OpenWrt 的 192.168.1.1（LuCI http://192.168.1.1，SSH root@192.168.1.1，首次无密码）。

第一版很快编出来了，13,894,743 字节，SHA256 是 f8e1148ec6e3a1b796073076d228c7ede0733f8a9502f9d56e478208dde0820c，刷上去却起不来：蓝灯一直闪，Mac 直连拿到 169.254.13.87 自分配地址（子网 255.255.0.0，无路由器），说明未获 DHCP，打不开 192.168.1.1。蓝灯是 gpio16 系统灯，正常启动完成应常亮，一直闪就是未启动完、安全模式或启动循环。查编译日志没报错，镜像结构也对得上。起初怀疑网线插错了口——这台机器一共只有 3 个网口（1 个 WAN 加 2 个 LAN），我之前还错说成 4 个 LAN 口。Breed 重刷还是一样。

第二版改按 guo 给的 lazywalker/mmdvm-openwrt feed 正规打包重编（ostar 那个仓库是 BD7MQB 的 19.07 完整 buildroot、无 D1 配置，不适合），同时编了一个纯净官方对照版 clean-v2（官方默认 126 包、无 LuCI 只能 SSH，7,078,999 字节，SHA256 ff3aa7d4697fdda85976db433a6216c399b8e2ffae7b28bf71d159b2d35b542d）。纯净版一刷就起了（2026-11-05 15:47，Mac 能拿到 DHCP），证明硬件没问题；再刷第二版 v2（12,715,095 字节，SHA256 ec54f6eaf3275633612897f9e293adac340c9bdec5a1f32672f6cb0ff279ec76），还是蓝灯闪、拿不到地址。v2 的 MMDVMHost 是 1.3.2 的 2022 旧版，DMR ID 库也是 2022 年的，各网关 ini 仍是默认呼号 N0CALL，刷完要在 LuCI 里改。

接着查到两个可疑点：feed 的 11 个 init 脚本都在 start_service() 里跑 `ubus -t 60 wait_for network.wireless`，WiFi 默认关闭，network.wireless 永不出现，每个启动的服务白等 60 秒，最坏串行 11 分钟；几条首次启动脚本（99_*_init）在停用服务前先调 stop 命令，首次启动时服务根本没跑过。第三版 v3 把两处都修了（99_*_init 删 stop 只留 disable、11 个 init 脚本删掉等无线的等待、补 99_dmrid-init 默认禁用，12,715,095 字节，SHA256 cc9afbc1…741f7b），仍然不行。

于是做二分对照：bisect-A（干净底子加整份 v3 的 FILES 覆盖层，7,078,999 字节，SHA256 f52befe3…0c5357a）起不来，根因在 FILES 覆盖层里；bisect-C（干净底子只改 inittab 删 askconsole 一行，7,078,999 字节，SHA256 379e083a…102b2b5）也起不来。我用 unsquashfs 把 clean-v2 和 bisect-C 的 squashfs（偏移 3,189,062，magic hsqs）解开逐项对比，两者唯一差别就是 inittab 少了 `::askconsole:/usr/libexec/login.sh` 这一行。按这个思路编了第四版 v4，把 inittab 换回官方原版（12,715,095 字节，SHA256 01220e10…e9d55，我亲自解包核验与 clean-v2 逐字节相同），2026-11-05 16:56 guo 回报"还是不行"。

两次结论互相矛盾，说明有混杂因素：可能是纯净版能起只是当时的状态，之后设备、刷机或线路侧变了；也可能是后续刷写没生效，或电脑侧还留着手动地址没切回自动获取。ping 测试一直无效：guo 的 Mac 当时在 192.168.1.0/24 没地址（静态 IP 没设上），ping -c 4 192.168.1.1 是 100% 丢包，nc -vz 192.168.1.1 23 报 No route to host，始终没真正测到设备。继续盲刷没有意义。

现状：停在重刷纯净对照版 clean-v2 做决定性验证。能起，问题还在镜像里；起不来，问题就不在固件，改查设备、刷机和线路。教训是反复失败时先做一次决定性对照再编下一个版本，指示灯和地址分配说了算。
