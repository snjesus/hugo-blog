---
title: "Newifi D1 MMDVM 重编译实录：CFLAGS 覆盖、opkg 递归与 D-STAR 呼号"
date: 2026-11-19
---

这篇接着 [Newifi D1 精简 MMDVM 固件](https://blog.bd4two.eu.org/posts/d1-mmdvm-firmware/) 那篇往下写。固件刷进去之后，跑着跑着发现几个不对劲的地方：网关日志的时间戳格式跟别的程序不一样，P25/NXDN/YSF 三个网关的日志干脆连时间戳都没有，MMDVMHost 重启之后 D-STAR 那边老是连不上反射器。于是决定把 feed 拉下来用官方 SDK 重编译一遍，根源其实是一行 Makefile。

这次一共挖出三个层层递进的问题：feed 用 `MAKE_FLAGS` 传 `CFLAGS` 把上游 Makefile 自己那行整条顶掉；自定义 opkg 打包脚本调了 `default_postinst` 造成无限递归，装包直接挂死被 OOM 杀掉；运行时的 ID 库和反射器表被当成普通文件，每次升级都被包内种子覆盖。最后还牵出一个附加谜题：D-STAR 日志里反射器中继显示成 `N0CALL` 而不是 `BR4SN`，真因是 IRC-DDB Gateway 的配置文件从头到尾就没被改过。

设备还是那台 Newifi D1（ramips/mt7621，OpenWrt 24.10.5），呼号 BR4SN，DMR ID 460042，D-STAR 走 XRF055 C 反射器。

## 先说结论：为什么非重编译不可

那套预编译的 ipk 有一个问题，它的 `Log.cpp` 是按"没有 `HAVE_LOG_H`"编译的。日志长这样：

```
Opening DMR network
Loaded 230973 Ids to the DMR callsign lookup table
```

应该长这样：

```
I: 2026-11-19 12:01:33.861 MMDVMHost-20221211-OSTAR is starting
I: 2026-11-19 12:01:33.864     D-Star: enabled
I: 2026-11-19 12:01:33.864     DMR: enabled
```

差别在于 `HAVE_LOG_H` 这个宏决定了 `Log.h` 走哪条路。一眼看就知道，这个宏在编译期被丢了。

一个字节一个字节 patch 预编译二进制当然能糊过去，但那治不了根：只要上游再发一版新包，还得从头 patch 一遍。根治路线就是把 feed 修好，用官方 SDK 重新编译，出自己的包。

## 根因：三层叠加的变量覆盖

这是整个过程里最值得记住的一段。

OpenWrt 的构建系统里，一个包能拿到编译参数的地方有好几层，优先级是这样的：

```
MAKE_VARS   →   MAKE_FLAGS   →   命令行变量   →   makefile 里的普通赋值
             （越靠后优先级越高）
```

而 feed 侧原本写的是：

```make
MAKE_FLAGS +=\
    CFLAGS="$(TARGET_CFLAGS)" \
    LDFLAGS="$(TARGET_LDFLAGS)" \
```

`CFLAGS` 是**命令行变量**。一旦它出现在命令行里，makefile 里的任何普通赋值（包括 `+=`）都会被完全忽略。于是上游 `P25Gateway/Makefile` 里这行：

```make
CFLAGS += -DHAVE_LOG_H
```

就等于从来没存在过。而上游 `MMDVMHost/Makefile` 里这行：

```make
CFLAGS  = -g -O3 -Wall -std=c++0x -pthread
```

同样被顶掉——而且这次丢的是 `-pthread`，那可是**功能性的**：MMDVMHost 是多线程程序（`Thread.o` / `Mutex.o`），编译期没有 `_REENTRANT`，线程语义未定义。

### 正解

feed 侧不再传 CFLAGS，让 OpenWrt 自己那份生效：

```make
MAKE_FLAGS +=\
    LDFLAGS="$(TARGET_LDFLAGS)" \
```

各子目录用 `override` 把自己的参数补回来：

```diff
--- a/Makefile          # 上游 MMDVMHost/Makefile
+++ b/Makefile
-CFLAGS  = -g -O3 -Wall -std=c++0x -pthread
+# 说明：feed 不再传 CFLAGS，所以这里必须用 override 把上游自己的编译参数补回来，
+# 否则会被命令行变量整条顶掉（宏/开关全丢）。
+#   -pthread   功能性修复：MMDVMHost 是多线程程序，缺了编译期 _REENTRANT
+#   -O3 -Wall  恢复上游的优化/告警级别
+# 去掉上游的 -g：本 SDK 未开 CONFIG_STRIP，带调试信息会让二进制从 458KB
+# 涨到数 MB，白占路由器 24MB 的 jffs2。
+override CFLAGS += -O3 -Wall -std=c++0x -pthread
```

两个容易踩的细节。`override` 不可省：命令行变量不受 makefile 普通赋值影响，`CFLAGS +=` 会被无视，只有 `override` 才生效。另外是故意去掉 `-g`：本 SDK 没开 `CONFIG_STRIP`，带调试信息二进制会从 458KB 涨到数 MB，路由器的 jffs2 只有 24MB，白占 flash 不可接受。

### 验收不能看 Makefile，要看真实命令行

这里有个坑值得单独拎出来。构建脚本里我本来打了一行"OK，实际 CFLAGS："，用 `grep -oE 'CFLAGS="[^"]*"' compile.log` 去抓。结果抓出来的是 OpenWrt 自己那份 `TARGET_CFLAGS` 的打印，不是上游 Makefile 里的 `$(CFLAGS)` 变量，完全是两回事。

正确做法是去编译日志里看真实的 g++ 命令行：

```
mipsel-openwrt-linux-musl-g++ -Os -pipe -mno-branch-likely -mips32r2 -mtune=24kc \
  -fno-caller-saves -fno-plt -fhonour-copts -msoft-float \
  -ffile-prefix-map=.../mmdvm-host-1.3.2=mmdvm-host-1.3.2 -mips16 -minterlink-mips16 \
  -Wformat -Werror=format-security -fstack-protector -D_FORTIFY_SOURCE=1 \
  -Wl,-z,now -Wl,-z,relro -std=c++0x -DOPENWRT \
  -I.../staging_dir/toolchain-mipsel_24kc_gcc-13.3.0_musl/usr/include ... \
  -O3 -Wall -std=c++0x -pthread -c -o DMRLookup.o DMRLookup.cpp
```

尾部那 `-O3 -Wall -std=c++0x -pthread` 才是 `override CFLAGS +=` 补回来的东西。数一下就知道：MMDVMHost 共 88 个编译单元，`-O3` / `-Wall` / `-pthread` 各出现 88 次，`-g ` 出现 0 次——这才是可量化的验收判据。

## 构建环境的几个硬事实

官方 SDK 是 `openwrt-sdk-24.10.5-ramips-mt7621_gcc-13.3.0_musl.Linux-x86_64.tar.zst`。Mac 是 arm64，跑不了 x86_64 的交叉工具链，所以用 lima 起一个 x86_64 的 Ubuntu 客户机来编译。

有几个 SDK 事实必须记住。SDK **不带 `.config`**，要用 `make defconfig` 非交互生成。SDK 的 `package/` 里只有 kernel 和 toolchain，base 包（libexpat、zlib、libwxwidgets 这些）都是预编译 ipk、没有源码，所以依赖链上只要碰到它们就编不出来。还有 **OpenWrt 的 ipk 是 tar.gz 不是 ar 归档**，成员是 `./debian-binary`、`./data.tar.gz`、`./control.tar.gz`，用 `ar` 解会失败。另外 `defconfig` 阶段也必须带上裁剪变量，否则 Kconfig 会把包直接取消选中。

### 依赖裁剪：构建期减、打包后补

`mmdvm-host` 上游的 `DEPENDS` 含 `+ircddb-gateway +dapnet-gateway`，它们依赖 `libwxwidgets → libexpat/zlib`，SDK 里编不出来。但 MMDVMHost 不链接任何网关库，只用到 libc / libpthread / libstdcpp，那些对 MMDVMHost 而言只是运行时依赖。所以构建期裁掉，打包后再把 ipk control 里的 `Depends` 改回上游原样，保证 opkg 的依赖语义和已装的旧包完全一致：

```
Depends: libc, libpthread, libstdcpp6, ysf-clients, p25-clients, nxdn-clients, ircddb-gateway, dapnet-gateway
```

## opkg 的无限递归（以及它怎么把路由器搞挂）

第一次装包，**直接挂死了**。

现象是装到一半不动，内存暴涨，最后进程被内核 OOM 杀掉；包状态卡在 `unpacked`，文件留在盘上但包管理器不认，`/var/lock/opkg.lock` 还不释放。dmesg 里一堆同名进程同时在跑：

```
oom-kill:constraint=CONSTRAINT_NONE,...task=p25-clients.pos,pid=16415,uid=0
Out of memory: Killed process 16415 (p25-clients.pos) total-vm:7184kB, anon-rss:5776kB
```

同一个脚本被反复执行。

### 根因

OpenWrt 的打包机制里，你在 `Package/x/postinst` 里写的内容会被写进 ipk 的 **`postinst-pkg`**；SDK 生成的主 **`postinst`** 里有一句 `default_postinst $0 $@`；而 `default_postinst` 做的事是**在子 shell `( . postinst-pkg )` 里 source 你的脚本**。

如果你的 `postinst-pkg` 里又调了一次 `default_postinst`，就成了这样：

```
postinst ──> default_postinst ──> ( . postinst-pkg ) ──> default_postinst ──> ...
```

无限递归。每一层 fork 一个子 shell，内存线性增长，最后触发 OOM。

### 正解

`postinst-pkg` 里绝不能再调 `default_postinst`，直接 `exit 0`：

```sh
#!/bin/sh
# 绝对不要再调 default_postinst！
# default_postinst 会在子 shell `( . postinst-pkg )` 里 source 本文件，
# 这里再调一次就是无限递归 -> 安装挂死 -> opkg 锁不释放。
for f in DMRIds.dat BMMasters.txt; do
	if [ -s /tmp/.mmdvm-$f.save ]; then
		cp /tmp/.mmdvm-$f.save /etc/mmdvm/$f
		rm -f /tmp/.mmdvm-$f.save
		logger -t mmdvm-host "postinst: 已恢复 /etc/mmdvm/$f"
	fi
done
exit 0
```

`default_postinst` 是在子 shell 里被 source 的，所以这里的 `exit 0` 只结束子 shell，不影响后续 `init.d enable/start` 的正常执行。教训很简单：自定义 opkg 脚本前，先把 `/lib/functions.sh` 里的 `default_postinst` 读完，写之前问一句"这段代码会被谁 source 几遍"。

## 运行时数据保护：升级一次就丢一次

MMDVM 的 ID 库和反射器表是**运行时数据**，但它们被塞在 ipk 的普通文件位置：

| 文件 | 现网 | 包内种子（上游旧快照） | 丢了会怎样 |
|---|---|---|---|
| `/etc/mmdvm/DMRIds.dat` | **317,805 条** / 7,785,099 B | 230,972 条 / 7,142,836 B | **回退 8 万多条 ID** |
| `/etc/mmdvm/BMMasters.txt` | **42 条**（wiki+DNS 存活验证版） | 旧版 | BM 主站表失准 |
| `/etc/mmdvm/YSFHosts.txt` | **1,560 条** | 旧版 | YSF 反射器表缩水 |
| `/etc/MMDVM.ini` | 含 BM 密码 | 上游默认 | **密码丢失 → 登录不了 BM** |

保护用双保险。**A 是声明成 conffile**：

```make
define Package/${PKG_NAME}/conffiles
/etc/MMDVM.ini
/etc/mmdvm/BMMasters.txt
/etc/mmdvm/DMRIds.dat
endef
```

**B 是 preinst 备份 + postinst-pkg 恢复**：

```make
define Package/${PKG_NAME}/preinst
#!/bin/sh
for f in DMRIds.dat BMMasters.txt; do
	if [ -s /etc/mmdvm/$$f ]; then
		cp /etc/mmdvm/$$f /tmp/.mmdvm-$$f.save
		logger -t mmdvm-host "preinst: 已备份 /etc/mmdvm/$$f"
	fi
done
exit 0
endef
```

备份放 `/tmp`（tmpfs，119MB 可用），不放 jffs2——7.7MB 的 ID 库白占 24MB flash 不可接受。

### 第一次升级仍会丢一次

这是踩过的坑，规律很硬：**opkg 的 `--force-reinstall` 是「先卸载旧包，再执行新包 preinst」。** 旧包 r1 的 conffiles 里只有 `/etc/MMDVM.ini` 一个文件，`DMRIds.dat` 和 `BMMasters.txt` 在旧包眼里是普通文件，卸载时就被删了。等新包的 preinst 跑起来时，已经没东西可备份。

所以第一次升级必须手工备份加手工恢复，从 r2 起才自动生效。ysf-clients r1 → r2 首次安装时就实测到了：`YSFHosts.txt` 1562 条 → 1325 条，preinst 埋点完全没出现。靠更新脚本自己留的 `$DST.prev` 备份救回，再重拉刷新。mmdvm-host 第一次装也一样，只能手工恢复。

装完三个运行时文件依旧逐字一致（`DMRIds.dat` 317,805 条、`BMMasters.txt` 42 条、`/etc/MMDVM.ini` 的 md5 装包前后完全相同），BM 密码文件零改动。

**第二次安装同一个包，埋点终于出现了**：

```
12:20:04 mmdvm-host: preinst: 已备份 /etc/mmdvm/DMRIds.dat
12:20:04 mmdvm-host: preinst: 已备份 /etc/mmdvm/BMMasters.txt
12:22:34 mmdvm-host: postinst: 已恢复 /etc/mmdvm/DMRIds.dat
12:22:34 mmdvm-host: postinst: 已恢复 /etc/mmdvm/BMMasters.txt
```

闭环了。规律是：**给运行时数据加 conffile 保护，第一次升级仍可能丢一次，之后才长期有效**。所以升级后要立刻核对条目数，别等到几天后才发现列表变短了。

## 上机前预检闸：以及它自己翻的三次车

这类问题在 Mac 上 3 秒能查出来，在路由器上要靠一次 OOM 才能暴露。所以写了个 `check-ipk.sh`，装包前把 ipk 拆开检查：conffiles 有没有带全运行时数据文件、`postinst-pkg` 里有没有 `default_postinst`（递归风险）、主 `postinst` 里有没有 `default_postinst`（没有的话 init.d 不会被 enable/start）、有没有残留 `${VAR}`、有没有变量展开为空的痕迹、`$(grep` 的 `$$` 转义对不对。

它当场抓出两个真问题。其一是 Makefile 里 `$(grep …)` 被构建期求值，应该写 `$$(grep …)` 才是运行时求值。其二是 `$f` 被 Make 展成空：

```make
	for f in DMRIds.dat BMMasters.txt; do
		if [ -s /etc/mmdvm/$f ]; then     # ← 错了
```

`$f` 在 Make 眼里是变量 `f`（未定义就是空），打包出来成了 `/etc/mmdvm/`，少一个文件名，**整个保护逻辑静默失效**。必须写 `$$f`。

### 预检脚本自己也翻车

修完上面两个，脚本在 build9 产物上**报失败**。但看命中行是：

```
postinst-pkg:7:# （打包后脚本里就变成 "/etc/mmdvm/" 这种路径，少一个文件名）——
```

命中的是**注释行**。根因是过滤注释用的 `grep -v '^\s*#'`，而 macOS 自带的是 BSD grep，**不认 `\s`**（GNU grep 认），过滤静默失效。修法是用 POSIX 字符类 `grep -vE '^[[:space:]]*#'`。

但更重要的是：**改检测规则必须做双向回归**，否则就成了"把检测放宽到能通过"。旧的真坏产物仍然要 FAIL，新的好产物要 PASS。顺带还得到一条强证据：build8 与 build9 的 `MMDVMHost` 二进制 md5 完全相同（`25ab977ca106b0a8728c357b229cfc6f`），说明 `$f` 修复只动了打包脚本，代码零变化。

## 三个网关的验收

新二进制与旧包的关键差异：

| 二进制 | 旧 md5 | 新 md5 |
|---|---|---|
| `P25Gateway` | 预编译版（无 `HAVE_LOG_H`） | `1c40ddb03efac7f937d94e8ca4306cb4` |
| `NXDNGateway` | 同上 | `687283cda88b6243d91c6cd8c4ce57c4` |
| `YSFGateway` | 同上 | `997901c816a75455d8215cbd970a077f` |
| `YSFParrot` | 同上 | `b445f688485a9ebd9f53e5c42b722d76` |

验收判据要跟着"修复了什么"换。字节补丁时代的判据是固定偏移，重编译后布局全变，必须换成行为判据：双 socket（每个端口两条 `Opening UDP port`，一条 IPv4 一条 IPv6，且带 `I: 时间戳` 前缀）、反射器条数（P25 433 / NXDN 392 / YSF 1560）、dmesg 里 SIGSEGV 不增长。

```
udp 0 0 0.0.0.0:42010 0.0.0.0:* 1752/P25Gateway
udp 0 0 :::42010    :::*    1752/P25Gateway
udp 0 0 0.0.0.0:14050 0.0.0.0:* 1933/NXDNGateway
udp 0 0 :::14050    :::*    1933/NXDNGateway
```

光看进程在不在不算数，做的是**决定性故障注入**——注入错误流量看链路是否真的通，网关向 MMDVM 侧发 poll、MMDVM 驱动建链、建链后状态是否为 `p25:conn`、上行语音帧到没到反射器、下行语音帧回没回 MMDVM 侧、链路保持、host 指向假反射器时是否正确拒绝。七项全过，40 次 soak 采样全清。

## mmdvm-host：最后一个漏网的包

三个网关修完后，我把整个 feed（**11 个包**）全量重扫了一遍，只认真正存在的 `MAKE_FLAGS` 块——**只剩 `mmdvm-host` 还在覆盖 CFLAGS**。

它丢的不是日志宏，而是 `-pthread`，**功能性参数**。而 D-STAR 恰好就是 `mmdvm-host` 自己实现的（`DStarControl.o` / `DStarHeader.o` / `DStarNetwork.o` / `DStarSlowData.o`），所以这次修复对 D-STAR 属于必需修复，不只是优化。

装包前后：

```
Removing package mmdvm-host from root...
mmdvm: Stoping MMDVMHost Service...
mmdvm: Starting MMDVMHost Service...
Not deleting modified conffile /etc/MMDVM.ini.
Installing mmdvm-host (1.3.2-r2) to root...
Configuring mmdvm-host.
```

| 项 | 旧（上游 r1） | 新（重编译 r2） |
|---|---|---|
| `/usr/sbin/MMDVMHost` md5 | `7e77451fcf8f5c0a725dc733257fce44` | `25ab977ca106b0a8728c357b229cfc6f` |
| 大小 | 458,856 B | 524,392 B |
| `/etc/init.d/mmdvmhost` md5 | `6ad470e6…` | `25330714…`（feed 新版） |
| `Conffiles` | 只有 `/etc/MMDVM.ini` | 三个（含 DMRIds.dat / BMMasters.txt） |

装完包状态 `install user installed`，opkg 锁已释放，残留 opkg/postinst 进程 0，BM 登录成功（`DMR, Logged into the master successfully`），dmesg SIGSEGV **0**、segfault **0**，当日 MMDVM 日志 `E:` 级错误 **0** 行。

这里插一句判断方法：dmesg 里确实还有 OOM 记录，但**全部**是 `p25-clients.pos`，时间戳 `1117214–1117224`，而这次安装发生在 uptime `1182100+`——差 18 小时，是前一天那场递归风暴的旧账。**判断"是不是这次搞的"必须看任务名加时间戳两个维度，光看计数会误判。**

## D-STAR 收发复核：差点误判

新实例起来后，日志里只有：

```
I: 2026-11-19 12:01:33.863 D-Star: enabled
M: 2026-11-19 12:01:35.938 Opening D-Star network connection
```

**没有** `D-Star link status`，**没有** `received network header`。而旧实例（上游 r1）是有完整链路的：

```
10:29:48.919 D-Star link status set to "Linking to XRF055 C "
10:29:52.847 D-Star link status set to "Linked to XRF055 C "
10:29:54.889 D-Star, received network header from N0CALL /INFO to CQCQCQ via XRF055 C
10:30:02.050 D-Star, received network end of transmission from N0CALL /INFO to CQCQCQ , 7.2 seconds, 0% packet loss, BER: 0.0%
```

差点判成"重编译把 D-STAR 弄坏了"。

但先证伪再下结论：查 `ircddbgatewayd` 的日志，发现它**只在自身启动时**推一次 link 状态（`Linking N0CALL B at startup to XRF055 C` → `D-Plus link to XRF055 C established`），MMDVMHost 重启后它**不会补推**。旧实例之所以有，是因为当时 `ircddbgatewayd` 恰好也重启过。

于是做了决定性故障注入：重启 `ircddbgatewayd`，逼它重新推状态。新实例立刻响应：

```
M: 2026-11-19 12:06:18.111 D-Star link status set to "Linking to XRF055 C "
M: 2026-11-19 12:06:18.507 D-Star link status set to "Linked to XRF055 C "
M: 2026-11-19 12:06:20.562 D-Star, received network header from N0CALL /INFO to CQCQCQ via XRF055 C
M: 2026-11-19 12:06:27.724 D-Star, received network end of transmission from N0CALL /INFO to CQCQCQ , 7.2 seconds, 0% packet loss, BER: 0.0%
```

与旧实例**逐行同构**，连时长都一样（7.2 秒 / 0% 丢包 / BER 0.0%）。所以要记住：**每次重启 MMDVMHost 后，若 D-Star 没有 link status，顺手重启一次 `/etc/init.d/ircddbgateway`**。这是既有架构行为（单次推送），不是 bug，但极容易被误读成"D-STAR 挂了"。

## 附加谜题：为什么反射器上显示的是 N0CALL

上面每一条 D-Star 日志的源呼号都是 `N0CALL`：

```
D-Star, received network header from N0CALL /INFO to CQCQCQ via XRF055 C
```

但 MMDVM 自己的呼号明明是 `BR4SN`：

```
I: 2026-11-19 12:01:33.862     Callsign: BR4SN
```

两边身份不一致。

### 定位

`ircddbgatewayd` 的日志里，`BR4SN` 出现 **0 次**，`N0CALL` 出现 **52 次**：

```
M: Connected 'MSG' to N0CALL using file - /message.dvtool
M: Adding N0CALL B to the cache as a local repeater
M: Linking N0CALL B at startup to XRF055 C
M: Registered with dns.xreflector.net using callsign N0CALL
```

再找配置文件。`ircddbgatewayd` 的启动命令是 `/usr/bin/ircddbgatewayd`（不带参数），所以它读的是当前工作目录下的相对路径配置，而它的 cwd 是 `/`。有个 `/etc/ircddbgateway` 目录看着像，但 `ls` 出来是空的——因为它根本不是目录，是**一个文件**：

```
-rw-r-----    1 root     root          2951 Jan  1  1970 /etc/ircddbgateway
```

打开一看全是上游默认的占位符：

```ini
[default]
gatewayType=0
gatewayCallsign=N0CALL          ← 网关自身呼号
icomAddress=127.0.0.1
icomPort=20000
hbAddress=127.0.0.1
hbPort=20010
url=http://www.qrz.com/db/N0CALL
repeaterCall1=N0CALL            ← 本地中继（反射器上显示的那个）
repeaterBand1=B
repeaterType1=0
repeaterAddress1=127.0.0.1
repeaterPort1=20011
reflector1=XRF055 C
atStartup1=1
reconnect1=1
frequency1=439.10000
...
ircddbUsername=N0CALL
aprsEnabled=1
aprsHostname=asia.aprs2.net
dplusEnabled=1
dplusLogin=N0CALL              ← 登录反射器用的名字
```

`N0CALL` 是 IRC-DDB 的默认占位呼号。这套东西装完从来没被改过，所以反射器上我们的中继一直叫 "N0CALL B"。

### 修正

```sh
cp /etc/ircddbgateway /etc/ircddbgateway.bak-n0call
sed -i "s/N0CALL/BR4SN/g" /etc/ircddbgateway
chmod 0640 /etc/ircddbgateway
/etc/init.d/ircddbgateway restart
```

5 处占位符全替换。生效后：

```
14:54:27 Adding BR4SN B to the cache as a local repeater
14:54:27 Linking BR4SN B at startup to XRF055 C
14:54:28 D-Plus ACK message received from XRF055 C
14:54:28 D-Plus link to XRF055 C established
14:54:28 Registered with dns.xreflector.net using callsign BR4SN
14:54:28 Registered with auth.dstargateway.org using callsign BR4SN
14:54:29 CCS: BR4SN B connected to server ccs706.xreflector.net
```

`Links.log` 现在记的是 `Rptr: BR4SN  B Refl: XRF055 C`（UTC 时间戳，+8 是 14:54），原来是 `N0CALL B`。反射器上我们的身份算是正过来了。

### 一个附带收获：反射器自测帧的"残留表项"

改完呼号后，15:00 整点反射器的自测帧（`/TIME`）却没收到，网关日志里是：

```
M: 2026-11-19 15:00:00: Incoming G2 header from N0CALL to unknown repeater - N0CALL B
```

`unknown repeater - N0CALL B`，反射器端还留着一条叫 N0CALL B 的表项，它的自测帧仍发给 N0CALL B，而我们只认 BR4SN B，帧被丢弃。我一度以为是改名前自己的残留，等了半小时以上也没消失，16:00 整点又来一轮还是一样，而 `BR4SN B` 的计数始终是 0——说明 XRF055 C 上那个 N0CALL B 是**另一台设备**（别人家没改呼号的默认配置），不是我们的残留。

但**真实电台流量完全不受影响**：

```
15:50:14.935 D-Star, received network header from BI1JVA /MDV to CQCQCQ
15:50:27.854 D-Star, received network end of transmission from BI1JVA /MDV to CQCQCQ , 12.9 seconds, 0% packet loss, BER: 0.0%
15:50:39.100 D-Star, received network header from BI1KQL / to CQCQCQ
```

注意 `via XRF055 C` 的后缀**消失了**，说明反射器是直接匹配到 BR4SN B 这个中继转发的，而不是像以前那样"经由某个 repeater 转发"。整个 15:50 到 16:00 之间 BI1KQL、BI1JVA 的通信基本 0% 丢包。

顺带澄清一个误解：日志里那个 `from N0CALL` **全部来自反射器每小时一次的自测帧**（`/TIME` 和启动时的 `/INFO`），真实电台流量显示的永远是原电台呼号（BI1KQL、BI1JVA、BG3CGC、LX6FB 这些）。所以这个显示问题**不影响实际收发**，只影响"我们自己在反射器上自测时的身份显示"。既然真实身份已经改成 BR4SN，那个 `from N0CALL` 也就无所谓了——硬要改回去反而会和别人的 N0CALL 重名。

## 经验总结

1. **"覆盖顺序"是构建系统里最隐蔽的一类 bug。** 三层变量叠加，宏就这么没了。症状还特别安静：程序能跑，只是日志里少了几个字。
2. **判据要跟着"修复了什么"换。** 字节补丁时代看偏移，重编译后必须看行为。
3. **失败模式不是"装失败"，而是"装到一半挂死"。** 这类静默失败比崩溃危险——网关不工作、装不上新包、不知道哪一步坏的，三重叠加。
4. **上机前加一道"拆开看"的预检闸。** Mac 上 3 秒能查出来的问题，路由器上要靠一次 OOM 才能暴露。
5. **给运行时数据加 conffile 保护，第一次升级仍可能丢一次。** 装完立刻核对条目数。
6. **"没有独立的包"不等于"这个模式不受影响"**——要看实现落在谁身上。D-STAR 没有 `dstar-clients` 包，它由 `mmdvm-host` 自己实现。
7. **全量扫一遍，别只查你正在修的那几个包。** 11 个包里最后一个漏网的，丢的还是功能性参数。
8. **自定义 opkg 脚本前，先把 `/lib/functions.sh` 里的 `default_postinst` 读完。**
9. **判"某个方案不行"之前，先确认你测的那份产物里真的有那个改动。** 本次一度把 conffile 方案判成"被证伪"，其实那份 ipk 根本没带 `Conffiles` 字段。**假证伪比没验证更危险。**
10. **工具自己也会骗你，而且骗你的方式是"假通过"或"假失败"。** macOS 的 BSD grep 不认 `\s`，过滤注释行静默失效。改检测规则后**必须做双向回归**。
11. **"看起来功能退化"先证伪，别急着归因到刚做的改动。** 解法不是猜，是做一次能定性的故障注入——它能把两种解释劈开。
12. **版本串、构建时间戳这类"自报家门"的字段不能当版本证据，只有二进制 md5 能。** 上游 feed 锁的 commit `81ad9f0` 里 `Version.h` 写的是 `20200809`，可上游发布的 r1 ipk 打出来却是 `20221211-OSTAR`，两边的版本号根本不能用来区分新旧。

## 常用命令速查

```sh
# —— 免密 SSH ——
ssh -o BatchMode=yes -o StrictHostKeyChecking=no root@192.168.1.1 '<cmd>'

# —— 推文件（Mac 的 scp 走 SFTP，路由器没有 sftp-server）——
ssh root@192.168.1.1 'cat > /path/file' < local_file
ssh root@192.168.1.1 'chmod 0755 /path/file'

# —— 段内安全改 ini（不打印密码）——
awk '/^\[D-Star/{f=1;print;next} /^\[/{f=0} f' /etc/MMDVM.ini | grep -viE 'password|passwd'

# —— 崩溃与内存 ——
dmesg | grep -c 'sending SIGSEGV'
dmesg | grep -iE 'out of memory|oom-kill'

# —— 确认某个文件被谁读 ——
ls -l /proc/$(pidof ircddbgatewayd)/cwd
ls -l /proc/$(pidof ircddbgatewayd)/fd

# —— 包管理 ——
opkg list-installed | grep mmdvm
cat /usr/lib/opkg/info/mmdvm-host.conffiles
```

几个 OpenWrt 小坑收在这里：路由器**没有 `stat`**，取 mtime 用 `date -r file +%s`；busybox `wget` 不支持 `-S`；ipk 是 **tar.gz** 不是 ar；`MAKE_VARS` 在 `$(MAKE)` 前、`MAKE_FLAGS` 在后，后者覆盖前者；Makefile 里 shell 变量必须写 `$$var` 否则被 Make 求值成空；`Package/x/postinst` 写进的是 **`postinst-pkg`**，主 `postinst` 里的 `default_postinst` 会 source 它，**别再调一次**。

---

下一篇大概会写写 Newifi D1 上这套 MMDVM 的日常运维：ID 库和反射器表怎么自动更新、D-STAR 中继怎么切、BM 主站挂了怎么自愈，以及那台 VPS 上跑的转发链路。
