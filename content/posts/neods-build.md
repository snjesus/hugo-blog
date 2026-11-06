---
title: "给 Nintendo DS 编译 2008 年的 NeoDS"
date: 2026-11-06
---

手里有一台 Nintendo DS LL，11 月 2 日想在上面跑 NeoGeo 模拟器 NeoDS。它的源码是 https://github.com/Yardape8000/NeoDS，2008 年的 v0.2.0，十八年前的老代码，要用现代的 devkitARM 和 libnds 编通。

老源码和新工具链之间有不少不兼容的地方。做法是加了一个兼容补丁头文件 common/source/compat-nds.h，在编译时用 `-include` 注入，修掉新旧接口对不上的地方（旧寄存器/函数名映射，如 SOUND_CR、BLEND_CR→REG_BLDCNT、CARD_*→现代 REG_*、powerON→powerOn、irqInit/rtcReset 空操作、touchReadXY 包一层现代 touchRead、strcmpi→strcasecmp、vramSetMainBanks→vramSetPrimaryBanks 等），再配上合适的链接参数：Makefile 改用 calico 的 ds7/ds9 specs、链 -lcalico_ds7/-lcalico_ds9，arm9 INCLUDES 补 ../Cyclone2，objcopy 剥掉 .twl* 段（否则 ARM7 二进制被段间空洞撑到 11MB），去掉现代 GCC 已不支持的 -mno-fpu。内存是真正的硬约束：DS 模式下的可用内存有上限，ARM9 的堆必须缩小才能塞进去。最后把 arm9/source/EmuSystem.c 里的 `g_mainRam` 从 `[1*MB+600*KB]` 缩小到 `[1*MB+128*KB]`，只影响大游戏的精灵缓存，不影响兼容性。

工具链安装也费了一番事：devkitPro 官方 install-devkitpro-pacman 脚本因 apt 源 GPG 密钥缺失（NO_PUBKEY）失败，密钥本身已在带外核实是 devkitPro 官方签名密钥（指纹 ……C9D672AF4128A2F1），只是 apt 验签环节走不通，最后走通的是绕过 apt 验签（`apt-get update -o Acquire::AllowInsecureRepositories=true` + `apt-get install --allow-unauthenticated devkitpro-pacman`），装好 dkp-pacman 后用 `dkp-pacman -S nds-dev` 装齐 devkitARM + libnds + calico 于 /opt/devkitpro。

成品编出来了，NeoDS.nds 共 391,177 字节（ARM9 344,552 字节、ARM7 45,812 字节），在 ~/workspace/your_files/NeoDS.nds，拷到烧录卡的存储卡根目录就能跑。游戏 ROM 得自己准备：用仓库自带的 NeoDsConvert 把 MAME 格式的 NeoGeo ROM 和 neogeo BIOS 转成 .neo 文件，BIOS 和游戏 ROM 都得自己提供。

有一点必须说清楚：这份成品我这边没有真机验证过。编译通过不等于能跑，能进菜单、能列出游戏才算成。等我在 DS LL 上实测了再补结果。
