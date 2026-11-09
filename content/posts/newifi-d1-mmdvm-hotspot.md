---
title: "把 MMDVM 数字语音热点塞进一台 32MB 的路由器"
date: 2026-11-09
---

# 把 MMDVM 数字语音热点塞进一台 32MB 的路由器

> newifi d1（联想 MT7621AT）刷 OpenWrt 24.10.5 + MMDVM 双工热点实战实录
>
> 从"固件刷不进去"到"Brandmeister 中国主站上线并跑出真实通联"，全程约一周。真正卡住我的不是硬件，也不是串口，而是三个**看起来毫不相干、实际上都源自同一类 bug** 的问题。

---

## 最终状态速览

| 项目 | 值 |
|---|---|
| 硬件 | newifi d1（联想），MT7621AT，32MB NOR，256MB RAM |
| 固件 | OpenWrt 24.10.5 (r29087-d9c5716d1d)，内核 6.6.119 |
| 调制解调器 | MMDVM HS Dual Hat v1.5.2，双 ADF7021，14.7456MHz 参考 |
| 串口 | `/dev/ttyS0`（板上**唯一**硬件串口） |
| 呼号 / DMR ID | BR4SN / 460042 |
| 频率 | RX 434.750 MHz / TX 439.750 MHz，双工 |
| 网络 | Brandmeister 主站 4602（中国，43.129.83.124） |
| YSF 反射器表 | 1,508 条（refcheck 官方源，每日自动更新） |
| 状态 | `Logged into the master successfully`，实测 0% 丢包 |

---

## 一、先把问题拆清楚

拿到手时设备处于一个半成品状态：刷过自定义 MMDVM 镜像，开机日志里有一行刺眼的

```
E: Received a NAK to the SET_FREQ command from the modem
```

直觉是"调制解调器拒绝了频率设置"，于是顺着频率、串口、OLED 方向查了一圈，全是死胡同。

实际上这句话**只是症状**。要定位根因，必须先把三件事分开：

1. **本地配置**（ini）到底被写进去没有？
2. **调制解调器**到底收到了什么？
3. **BM 网络**到底认不认我？

而其中第 1 件事本身就是坏的——**LuCI 页面上显示的值和真正生效的值不是一回事**。这是整个排查最关键的一步。

---

## 二、固件路线：官方底包 + 离线 opkg

### 2.1 为什么不用定制镜像

一开始刷的是 BD7MQB 的 MMDVM 定制镜像，失败后换了思路：**官方 OpenWrt 底包 + 离线 opkg 补装 MMDVM**。理由很简单——自包含、零联网依赖、可控。

分区表（`cat /proc/mtd`）：

```
mtd0  u-boot      192K
mtd1  u-boot-env   64K
mtd2  factory      64K
mtd3  firmware  32448K     ← 32MB 整片
```

`/overlay` 挂在 `/dev/mtdblock6` 上，**jffs2，24.6M**，可写可持久。这意味着配置重启不丢，但也意味着**存储极其紧张**（后面会看到这个紧张点差点酿成事故）。

### 2.2 自制 .ipk 的格式坑

为了离线补装，我需要自己打 39 个 .ipk。官方 opkg 的 `.ipk` 其实是 **ar 归档**（Debian 风格），但 OpenWrt 实际能吃的是 **tar.gz 容器**：

```
./debian-binary
./data.tar.gz
./control.tar.gz
```

几个必须注意的细节：

- `control.tar.gz` 里的成员名要带 `./` 前缀
- `data.tar.gz` 里要**包含目录条目**（`./usr/`、`./usr/sbin/`…），否则解压后目录不存在
- **符号链接必须原样保留**（`tar -h` 会把链接变成实体文件，直接破坏 `ln -s` 语义）

最终打出来的包集约 7.0MB（`mmdvm-ipk.tar.gz`）。

### 2.3 那个"架构不兼容"的假象

装包时报：

```
cannot find dependency libstdcpp6 for libmmdvm
```

同时还伴随一句极具误导性的：

```
Package libstdcpp6 is not compatible with the architectures configured
```

**架构根本不是问题**——官方 24.10.5 底包**不带 `libstdcpp6`**（C++ 运行时），而 MMDVM 是 C++ 程序。补进版本完全一致的 `libstdcpp6_13.3.0-r4`（target packages feed）、`libexpat_2.8.4-r1`、`zlib_1.3.1-r1` 后，依赖链闭合，装机成功。

> **教训**：opkg 报"架构不兼容"时，先去 target feed 里确认这个包在你的 target 上是否存在，别去改 `arch` 配置。

---

## 三、串口：这块板只有 ttyS0

我一度以为是 `ttyS1`，被用户直接否定。查 OpenWrt 的 DTS 后确认用户是对的：

```dts
&state_default {
    gpio { groups = "jtag", "uart2", "wdt"; function = "gpio"; };
};
```

**UART2 的引脚被改成 GPIO 了**，UART1 也没启用。所以板上唯一的硬件串口就是 `ttyS0`：

```
1e000c00.uartlite: ttyS0 at MMIO 0x1e000c00 (irq 18) is a 16550A
console=ttyS0,115200
```

内核控制台默认也占着 ttyS0，处理办法是：

```sh
sed -i 's|^::askconsole:|#::askconsole:|' /etc/inittab
echo "0 0 0 0" > /proc/sys/kernel/printk     # 关掉内核回显
```

注意 `stty` 在 busybox 上要写成 `stty ... < /dev/ttyS0`，否则错误会被吞掉、流控问题查不出来。

---

## 四、核心：三个看起来无关、实则同源的 Bug

### 4.1 Bug #1：LuCI 的 ini 解析器会丢弃"空值键"

现象是 LuCI 的 General 页面里，呼号、ID、频率都显示成了我填的值，点了"保存并应用"，**ini 文件里却纹丝不动**。

`/usr/lib/lua/luci/ini.lua` 的解析正则：

```lua
local key, value = line:match("^([%w_]+)%s-=%s-(.+)$")
```

`(.+)` 要求 `=` 后面**至少有一个字符**。而 MMDVM 的 ini 里长这样：

```ini
[General]
Id=
[Info]
RXFrequency=
TXFrequency=
[DMR Network]
Password=
```

**这些空值键在解析时被整条丢掉**，解析结果里根本不存在这些键。

再看 `/usr/lib/lua/luci/model/mmdvm.lua` 里 CBI 提交时的回写逻辑：

```lua
local mmdvmhost_conf = ini_load(MMDVMHOST_CONFFILE)
if mmdvmhost_conf[section] and mmdvmhost_conf[section][option] then   -- ← 键必须已存在
    mmdvmhost_conf[section][option] = value
    ini_save(MMDVMHOST_CONFFILE, mmdvmhost_conf)
end
```

**只写"ini 里已经存在的键"**。于是形成死锁：

> 空值键 → 解析时被丢 → 键不存在 → 回写时跳过 → 永远写不进去

而页面上为什么显示对？因为 LuCI 页面读的是 **UCI**（`/etc/config/mmdvm`），不是 ini。UCI 里存着用户填的值，页面自然显示正确——**"看起来保存成功了，其实只写进了 UCI，ini 一字未改**。

MMDVM 本体读的是 ini。所以：

- 页面显示 `RXFrequency=434750000`
- ini 里 `RXFrequency` **这一行根本不存在**
- MMDVM 启动 → `TX Frequency: 0Hz` → 拿 0Hz 去 SET_FREQ → 调制解调器 NAK

**修复**（一行）：

```lua
-- 改前
local key, value = line:match("^([%w_]+)%s-=%s-(.+)$")
-- 改后
local key, value = line:match("^([%w_]+)%s-=%s-(.*)$")
```

改完实测：空值键保留（`t.General.Id ~= nil` 为真），`uci2ini` 回写立刻生效。

### 4.2 Bug #2：一个无节区限定的 sed 污染了整份配置

安装脚本里有一行：

```sh
sed -i 's#^Port=.*#Port=/dev/ttyS0#' /etc/MMDVM.ini
```

`^Port=` 会命中**全文件所有段**的 Port 行。结果：

| 段 | 原始值 | 被改成 |
|---|---|---|
| `[Modem]` | `/dev/ttyS0` | `/dev/ttyS0`（正常） |
| `[UMP]` | `/dev/ttyACM1` | ❌ `/dev/ttyS0` |
| `[DMR Network]` | `62031` | ❌ `/dev/ttyS0` |
| `[Nextion]` | `modem` | ❌ `/dev/ttyS0` |
| `[LCDproc]` | `13666` | ❌ `/dev/ttyS0` |
| `[GPSD]` | `2947` | ❌ `/dev/ttyS0` |
| `[Remote Control]` | `7642` | ❌ `/dev/ttyS0` |

紧接着的第二行 `sed -i 's#^; *Port=/dev/ttyS0#Port=/dev/ttyS0#'`（解除注释）又在 `[Modem]` 里造出了**两行重复的 Port**。

修复：改成段内限定的 awk。

```sh
awk '/^\[/ { sec=$0 }
     /^;? *Port=/ { if (sec == "[Modem]") { print "Port=/dev/ttyS0"; next } }
     { print }' /etc/MMDVM.ini > /tmp/MMDVM.ini.new && cat /tmp/MMDVM.ini.new > /etc/MMDVM.ini
```

### 4.3 Bug #3：Brandmeister 密码为空 → 启动即断言中止

修好频率后，服务**还是**起不来。procd 报 `crash loop 6 crashes`，日志一片安静。直接把二进制放到前台跑，抓到了真凶：

```
Assertion failed: !password.empty() (DMRNetwork.cpp: CDMRNetwork: 75)
Aborted
```

`[DMR Network] Password=` 在原版 ini 里就是空的。而 `DMRNetwork.cpp` 里有一句硬断言——密码为空直接 abort。

讽刺的是，**密码明明就在 UCI 里**（`mmdvm.DMR_Network.Password`），又是被 Bug #1 那套"空值键丢弃"机制挡在门外。

写入密码后：

```
M: DMR, Opening DMR Network
I: Loaded 230973 Ids to the DMR callsign lookup table
M: MMDVMHost-20221211-OSTAR is running
```

### 4.4 附：BM 主站选错了

前两个 bug 修完，`2001.master.brandmeister.network`（欧洲主站）仍然超时。查 BM 官方 wiki 的中国页面：

```
Master Server Information
Master   IP Address      Location
4602     43.129.83.124   HK
```

并确认 **4601 已被踢出 Brandmeister 集群**，官方用 4602 顶替。中文玩家手册也写明：

> 国内通常选择 ~~BM_4601_China~~（由于接任 4601 服务器管理的团队并没有好好运营，因此 4601 服务器已被踢出 Brandmeister 集群并由 BM 官方团队设置了 **BM_4602_China** 作为代替）

换成 `4602.master.brandmeister.network` 后立刻：

```
M: DMR, Logged into the master successfully
```

> **坑**：镜像自带的 `BMMasters.txt` 里 `5151` 被标成"菲律宾"，而镜像默认用的就是 5151。用户 IP 段（460 开头）对应的是 4602。**别信设备里预置的默认值，去 wiki 查当期主站。**

---

## 五、SET_FREQ NAK 到底意味着什么

MMDVM 二进制里有两种完全不同的措辞：

```
Received a NAK to the SET_FREQ command from the modem    ← 板子明确拒绝了
The MMDVM is not responding to the SET_FREQ command      ← 超时，板子没回应
```

前者说明**通信是通的**，只是参数被拒。所以不用怀疑串口、电平、波特率——那些一旦有问题，板子根本不会回版本串。

判定链路应该这样走：

```
MMDVM 能否读到调制解调器版本串？
  ├─ 不能 → 串口/电平/波特率/占用问题
  └─ 能   → 报文收发了 → 看 NAK 还是超时
                ├─ 超时 → 板子没收到（检查 TX 方向）
                └─ NAK → 板子收到了但拒绝（查参数）
```

同样，`Display=OLED` 时报的 `Error returned from write(), errno=%d` 也不是串口错误——那条字符串属于 `I2CController.cpp`（CI2C 类），串口类 `CSerialController` 根本没有 write 错误日志。**没有 I²C OLED 就要 `Display=None`**，否则同样会打断启动。

---

## 六、DMR ID 库：更新 + country 列合并

### 6.1 数据源选择

设备自带的 `DMRIds.dat` 是 230,972 条，来自镜像打包时刻。

现役的每日自动生成源是 Pi-Star 同源的 GitHub 仓库：

```
https://raw.githubusercontent.com/DMR-Database/database-beta/master/DMRIds.dat
```

**317,967 条**，实测每天更新一次。

### 6.2 3 列 vs 4 列

新源是 3 列（ID / 呼号 / 名字），设备用的格式是 4 列（多一个 country）。而 LuCI 的 Last Heard 详情页会显示国名：

```lua
<p><%=user.country or "Somewhere" %></p>
```

**Lua 里空串是 truthy**，`"" or "Somewhere"` 返回空串——直接换 3 列库会让这一行变成**空白**而不是"Somewhere"。

解法：用旧库做 ID→country 映射，把第 4 列补回去。

```python
cc = {}                                    # 旧库建 ID->country 映射
for line in open(old):
    p = line.rstrip("\n").split("\t")
    if len(p) >= 4:
        cc[p[0]] = p[3]

with open(new) as fi, open(out, "w") as fo:  # 新库逐行补列
    for line in fi:
        p = line.rstrip("\n").split("\t")
        if len(p) >= 3:
            fo.write("%s\t%s\t%s\t%s\n" % (p[0], p[1], p[2], cc.get(p[0], "")))
```

结果：317,964 条有效记录，221,954 条带国名，96,010 条新 ID 暂空。**文件反而比原版小**（7.57MB vs 7.14MB 略大，但比 3 列版小得多）。

### 6.3 别拿 `usrmd2017.csv` 当全量源

同一仓库里还有 `usermd2017.csv`（带 City/State/Country），看着很美，但它是**截断的 10 万行**，只覆盖到 ID 2629183（德国段）。拿它当全量会丢掉大半数据。

---

## 七、服务器列表：一次源站考古

`/etc/mmdvm/` 下那一堆列表文件，得先搞清楚谁在读：

| 文件 | 读取方 | 处置 |
|---|---|---|
| `BMMasters.txt` | LuCI `mmdvm.lua`（主站下拉框） | **重建** |
| `FCSRooms.txt` | 仅 `FCSRoomsUpdate.sh` | **更新** |
| `YSFHosts.txt` / `P25Hosts.txt` / `NXDNHosts.txt` | LuCI `mmdvm.lua` | 保持原样 |
| `CountryCode.txt` | **无人读取** | 保持原样 |
| `RSSI.dat` | MMDVM 本体（射频标定） | **绝不覆盖** |

### 7.1 为什么 MMDVM 自带的更新脚本一直没生效

包里有四个 `*Update.sh`，内容是：

```sh
wget -O /etc/mmdvm/BMMasters.txt -q "http://downloads.ostar.me/hosts/BMMasters.txt"
```

这个路径**从 2023 年起就全是 404**——作者把目录改名成了 `hosts.2022.bak/`：

```
Index of /
hosts.2022.bak/     2022-Dec-31
radioid.2022.bak/   2022-Nov-22
releases/            2022-Nov-22
```

所以这些列表**从 2022 年起再没更新过**。

### 7.2 BMMasters.txt 重建：wiki + DNS 存活验证

官方 wiki 只有部分国家页有主站表（69 个国家页里只有 2 页有），所以纯靠抓 wiki 不够。改用**实证法**：

1. 扫 69 个国家页 wikitext，提取正文里与 "master" 同现的 4 位 ID
2. 在**路由器上**逐个 `nslookup` 验证域名是否还解析
3. 死的剔除，活的保留，再从 wiki 补上新的

结果（42 个，全部通过 DNS 实测）：

- **剔除**：`2001-HAMNET`、`2021`、`2321`、`2381`、`4601`（已下线）
- **新增**：`2022`、`2322`、`2382`、`2681`、`4602`

这套"**以 DNS 存活为唯一真值**"的做法，比任何静态列表都可靠。

### 7.3 那几个不更新的列表为什么不动

| 文件 | 设备现值 | pistar 现值 | 决定 |
|---|---|---|---|
| `P25Hosts.txt` | **674 行** | 231 行 | 设备更全，保持 |
| `NXDNHosts.txt` | **522 行** | 162 行 | 设备更全，保持 |
| `YSFHosts.txt` | 1325 行（**8 字段**） | 1384 行（6 字段） | 已更新，详见 7.5 |

P25 和 NXDN 保持不动：设备现值条目明显更多，覆盖会丢东西。

YSF 那份当时被我判为"格式不兼容，保持"——**这个结论后来被推翻了**，原因见 7.5。真正需要小心的不是字段数，而是这个文件到底被谁读。

它们的现役源 `hostfiles.refcheck.radio` 现在改成了**凭呼号换 token 的门禁**（准确的接口形态见 7.5）：

```
POST /api/hostfile-gate/open/  {"callsign":"BR4SN"}  →  {"token":"..."}
GET  /api/hostfile-gate/fetch/<mode>/                Authorization: Token <token>
```

条款里写明「每个 hostfile 每小时只能取一次」「禁止大规模客户端软件拉取」。个人单机热点属于允许用途，但受限于限流——我探测时直接吃到了 429。

### 7.4 顺带纠正一个我自己的误判

中途我一度断定"路由器连不上外网，只能从 Mac 推送"，依据是 `uclient-fetch` 全部失败。**这个结论是错的**——

```
uclient-fetch → 连 baidu 都失败
busybox wget  → https://www.baidu.com/  OK
                https://raw.githubusercontent.com/...  OK (160s / 6.7MB)
```

**这台设备的 `uclient-fetch` 不支持 TLS，busybox `wget` 支持。** 一个工具的参数差异，差点让我把整个方案建立在错误前提上。

> MMDVM 包里的更新脚本用的正是 `wget`——如果当初先读包里的脚本，可能早就发现这点了。

### 7.5 更新 YSF 列表：先搞清楚"谁在读这个文件"

上一节我把 YSF 列表判成"格式不兼容，保持"。隔了一段时间决定更新它，重做了一遍判断——**结论变了，但不是因为找到了新源，而是因为我搞错了判断依据。**

#### 第一步：全量扫二进制，确认谁真的读它

```sh
for b in /usr/sbin/* /usr/bin/*; do
  [ -f "$b" ] || continue
  s=$(strings "$b" 2>/dev/null | grep -icE "ysfhosts")
  [ "$s" != "0" ] && echo "$b -> $s hits"
done
```

结果只有一条：

```
/usr/sbin/YSFHostsUpdate.sh -> 4 hits
```

**除下载脚本本身外，没有任何 C++ 程序引用过 `YSFHosts.txt`**——`MMDVMHost`、`YSFGateway`、`YSFParrot` 全部零命中。这直接推翻了"换过去会破坏 YSFGateway 解析"的说法：YSFGateway 根本不读它。

但它也不是死数据。它是 LuCI 里 **YSFGateway → Startup Reflector** 下拉框的数据源：

```lua
-- /usr/lib/lua/luci/model/cbi/mmdvm/config_dvmode.lua
o = s:option(ListValue, "Startup", translate("Startup Reflector"))
for _, r in ipairs(mmdvm.get_ysf_list()) do
    o:value(r[2], r[1] .. " - " .. r[2])
end
```

```lua
-- /usr/lib/lua/luci/model/mmdvm.lua
function get_ysf_list()
    local hostfile = "/etc/mmdvm/YSFHosts.txt"
    local file = assert(io.open(hostfile, 'r'), ...)
    local data = {}
    for line in file:lines() do
        local tokens = line:split(";")
        table.insert(data, {tokens[1], tokens[2]})
    end
    return data
end
```

注意这个解析器**不跳过 `#` 注释行，也不跳过空行**——和同文件里的 `_get_p25_nxdn_list()` 不同（后者有 `line:byte(1) ~= 35` 的判断）。所以上游文件如果带注释头，直接塞进去会在下拉框里多出 6 条垃圾项。**这是更新这个文件的第一约束。**

#### 第二步：接口形态踩了三个坑

`hostfiles.refcheck.radio/YSFHosts.json`（g4klx 脚本里写的那个地址）已经 **404**，现役接口是：

```
POST /api/hostfile-gate/open/  {"callsign":"BR4SN"}  →  {"token":"bcf28183..."}
GET  /api/hostfile-gate/fetch/<mode>/   Authorization: Token <token>
```

三个坑，每个都卡住过一轮：

| 坑 | 现象 | 解法 |
|---|---|---|
| 证书 CN 不含 `www` | `curl https://www.refcheck.radio/...` → `SSL: no alternative certificate subject name matches` | 用 `refcheck.radio`（不带 www） |
| 认证头前缀不是 `Bearer` | `Authorization: Bearer <tok>` → 401 | 是 `Authorization: **Token** <tok>` |
| fetch 末段是 **mode 不是文件名** | 传 `YSFHosts.txt` → 400 `Invalid mode 'ysfhosts.txt'` | 合法值：`dcs dextra dmr dplus fcs m17 nxdn p25 xlx ysf` |

另外这台机器上 **`curl` 的 LibreSSL 握手失败而 Python 的 `ssl` 正常**，路由器上的 busybox `wget` 也正常。所以「本机能 curl 就等于路由器也能 wget」的直觉在这里不成立，得分工具验证。

#### 第三步：格式对齐

上游是 7 段（末尾 URL 为空），设备上是 8 段（多一列 `Location`）。上游缺的这列在设备现值里 1324 条是 `0`、1 条是 `1`，属于可选字段。我补成 `0` 以保持与历史格式一致：

```awk
awk -F';' '{
    sub(/\r$/, "");
    if ($0 ~ /^#/ || $0 ~ /^[[:space:]]*$/) next;
    if (NF < 5) next;
    for (i = 1; i <= NF; i++) { gsub(/^[ \t]+/, "", $i); gsub(/[ \t]+$/, "", $i); }
    if (NF < 8) { for (i = NF + 1; i <= 8; i++) $i = ""; }
    if ($8 == "") $8 = "0";
    print $1";"$2";"$3";"$4";"$5";"$6";"$7";"$8;
}' input > output
```

这段 awk 在 **Mac 的 awk 和路由器的 busybox awk 上输出逐字节一致**（md5 均为 `88173b781abfef08483b589f4f9ba7cf`，1508 行）——跨实现一致性是必须验的，否则"在我这跑得好好的"上线就变形。

#### 第四步：用连通性证明新列表更好，而不是数行数

新旧两份列表的条目**几乎不重叠**（`host:port` 交集仅 253 条，主机 IP 交集 209/1004），所以"1325 vs 1508 谁多"这个问题本身就没有意义。唯一有意义的判据是：**里面的主机还活着吗。**

这里又踩了一个坑。我的 Mac 上有透明代理：

```
对照组(随机IP + 随机端口)  可连通 49/50   ← 随机 IP 都能连上
```

**Mac 侧所有 TCP 连通性测试全部作废**，连真活着的和随手编的 IP 结果一样。改到路由器侧测（路由器出口没有代理）：

```sh
ping -c 1 -W 3 <host>   # busybox 没有 timeout applet，nc 也不支持 -w，ping 是唯一可靠手段
```

各抽样 60 条：

| 列表 | 来源 | ping 存活 |
|---|---|---|
| 设备现值 | ostar 老列表 | **23/60（38%）** |
| 新版 | refcheck 官方 | **37/60（62%）** |

老列表里大量 `100.x.x.x` 这类 CGNAT 段地址，明显是历史垃圾数据。新列表明显更活跃，**替换是净收益**。

#### 第五步：限流与自动化

```
HTTP 429  {"detail":"Request was throttled. Expected available in 3129 seconds."}
```

限流是 **按呼号共享额度、每文件每小时一次**——Mac 上取过一次，路由器用同一个 token 再取同样被拒（token 由呼号决定，Mac 和路由器拿到的是同一个）。所以：

- 每天 cron 跑一次完全安全（间隔 24 小时 >> 1 小时）
- 想手动重跑，必须等窗口

脚本 `/usr/sbin/mmdvm-ysfhosts-update.sh` 做三件事：**换 token → 带 `Authorization: Token` 拉取 → 规范化后校验再替换**。校验含两条硬保护，任一不满足就保留旧文件：

1. 有效条目 < 500 → 上游异常，放弃替换
2. ID/端口非数字、或频率段不是 3 位数字 → 放弃替换

第一次实跑时它正确地**拒绝替换并保住了旧文件**（当时还在 429 窗口内）：

```
[2026-11-09 10:45:50] === YSFHosts update start ===
[2026-11-09 10:45:52] token acquired
[2026-11-09 10:45:54] ERROR: fetch failed
```

这也是为什么更新脚本必须**先校验后替换**，而不是直接 `wget -O` 覆盖目标文件——前者在上游抽风时保住现场，后者直接把你的可用列表清零。

cron 加在 04:57（排在日志清理之后，避免和另外两条撞车）：

```
57 4 * * * /usr/sbin/mmdvm-ysfhosts-update.sh >/dev/null 2>&1
```

---

## 八、让它自己会修：cron 自维护

确认路由器能直连 GitHub 后，把更新做成了**设备自己跑**，而不是依赖 Mac。

`/usr/sbin/mmdvm-dmrid-update.sh`：

```sh
URL="https://raw.githubusercontent.com/DMR-Database/database-beta/master/DMRIds.dat"
wget -q -T 900 -O "$TMP" "$URL"

# 条数明显减少说明源出了问题，宁可不动
[ "$NEWLINES" -lt $((OLDLINES - 1000)) ] && exit 1

# 用旧库做 country 映射，补回第 4 列
awk -F'\t' 'BEGIN{OFS="\t"}
    NR==FNR { if (NF>=4) cc[$1]=$4; next }
    { if (NF>=3) { print $1,$2,$3,(($1 in cc) ? cc[$1] : "") } else { print } }
' "$OLD" "$TMP" > "$MERGED"

cat "$MERGED" > "$OLD"
/etc/init.d/dmrid restart
/etc/init.d/mmdvmhost restart
```

`/etc/crontabs/root`：

```
17 4 * * * /usr/sbin/mmdvm-dmrid-update.sh
37 4 * * * /usr/sbin/mmdvm-log-clean.sh
```

### 这个保护机制真的救过场

连续四天的运行日志：

```
11-06 04:17  317967 行 → 更新
11-07 04:17  318046 行 → 更新   (+79)
11-08 04:17  318130 行 → 更新   (+84)
11-09 04:17  316422 行 → 条数异常减少，放弃替换   ← 保护生效
```

**11-09 那天上游源返回了残缺数据**（比前一天少 1700 条），条数校验拦住了它，完整的 318,130 条库被保留，服务照常运行。

另外注意路由器上 **jffs2 写 7.5MB 要约 1 分钟**，我一度以为文件写坏了——其实只是读到写入中途。

---

## 九、存储治理：日志差点撑爆 flash

设备只有 24.6MB 的 jffs2 overlay，而 MMDVM 日志**每天涨 0.9~1.7MB**（按日期滚动，一天一个文件）。

```
16.6M 可用 ÷ 1.4M/天 ≈ 12 天
```

**12 天后 flash 写满 → LuCI 存不下配置 → 路由器行为异常。** 这不是理论风险，是必然发生。

设备上**没有 logrotate**。解决方案是一个极简清理脚本：

```sh
LOGDIR=/var/log/mmdvm
KEEP=$(date +%Y-%m-%d)
for f in "$LOGDIR"/*.log "$LOGDIR"/*.log.gz; do
    [ -f "$f" ] || continue
    case "$f" in *"$KEEP"*) continue ;; esac
    rm -f "$f"
done
```

**为什么当天那份不能删**：MMDVMHost 还持有该文件的 fd，`rm` 掉只是摘掉目录项，**空间并不会真正释放**（要等进程退出才回收），还会丢掉当天日志。所以规则是「只保留当天」，最多占用 1~2MB。

实测效果：

```
清理前  /var/log/mmdvm  1240 KB
清理后  /var/log/mmdvm    76 KB
```

并且验证了 jffs2 删掉的块**能立刻复用**：连续写两次 3MB 都成功，`df` 一直稳在 16.5M 可用。

---

## 十、验证结果

配置落地后的日志：

```
I:     Callsign: BR4SN
I:     Id: 460042
I:     Duplex: yes
I:     TX Frequency: 439750000Hz (439750000Hz)     ← 括号内是调制解调器回读值
M:     Opening the MMDVM
I:     MMDVM protocol version: 1, description:
       MMDVM_HS_Dual_Hat-v1.5.2 20201108 14.7456MHz dual ADF7021 FW by CA6JAU
M:     DMR, Logged into the master successfully
```

括号里那个 439750000Hz 是**从调制解调器读回来的值**，和配置一致——这才是 SET_FREQ 真正被 ACK 的铁证。

真实通联记录：

```
M: DMR Slot 1, received network voice header from BH6IYI to TG 46001
M: DMR Slot 1, received network end of voice transmission from BG3IZE to TG 46001,
              0.1 seconds, 0% packet loss, BER: 0.0%
```

**0% 丢包、BER 0.0**——双工收发、频率、BM 注册、ID 库解码，整条链路都通了。

---

## 十一、踩坑速查表

| 现象 | 真因 | 处理 |
|---|---|---|
| opkg 报"架构不兼容" | 官方底包缺 `libstdcpp6` | 从 target feed 补同版本包 |
| LuCI 页面显示已改，ini 却没变 | `luci/ini.lua` 丢弃空值键 | 正则 `(.+)` → `(.*)` |
| MMDVM 启动无日志就退出 | `DMRNetwork.cpp` 断言密码非空 | 填 `[DMR Network] Password` |
| 多个段的 Port 全变成 `/dev/ttyS0` | sed 未限定段 | 改用段内限定的 awk |
| SET_FREQ 一直 NAK | 频率 0Hz（ini 没写进去） | 先确认 ini 真的变了 |
| BM 注册超时 | 主站选成欧洲 2001 | 换 4602（中国） |
| `uclient-fetch` 全失败 | 该工具不支持 TLS | 用 busybox `wget` |
| `scp` 传文件失败 | Mac 的 scp 默认走 SFTP，路由器无 sftp-server | `ssh 'cat > file'` 或 tar 流 |
| 换固件后 ssh 报主机密钥变了 | 正常 | `ssh-keygen -R 192.168.1.1` |
| 换了固件后 LuCI 登录不了 | rpcd 密码未同步 | `passwd` 重设后 `/etc/init.d/rpcd restart` |
| 文件写完看着"少了一半" | jffs2 写入 7.5MB 要约 1 分钟 | 等，别慌 |
| 本机 curl 报证书主机名不符 | 证书 CN 不含 `www` | 用不带 `www` 的域名 |
| `Authorization: Bearer` 拿 401 | 该站用的是 `Token` 前缀 | `Authorization: Token <tok>` |
| fetch 报 `Invalid mode 'ysfhosts.txt'` | 末段是 mode 不是文件名 | 用 `ysf`/`p25`/`nxdn` 等 |
| 随机 IP + 随机端口都"连得上" | 本机有透明代理 | 连通性测试必须换出口（路由器侧） |
| 换文件后下拉框多出几行空白项 | 上游带 `#` 注释头，而 LuCI 不跳过 `#` 行 | 入库前剥注释 |

---

## 十二、几个可复用的经验

1. **"页面显示正确"不等于"配置生效"。** 配置有多个存储层（UCI / ini / 硬件）时，务必**直接读最终生效的那一层**去验证，别信 UI。

2. **正则里的 `(.+)` 和 `(.*)` 差一个字符，语义完全不同。** 前者要求值非空——很多 ini 靠空值表达"未设置"，解析器一丢，后面所有"只更新已存在的键"逻辑就全瘫了。

3. **用 DNS 存活做真值。** 服务器列表、节点列表这类会自然失效的东西，与其相信任何静态清单，不如**逐条解析一次**。

4. **别信设备里预置的默认值。** 镜像里的 BM 主站、密码、频率，很可能对应的是 2022 年的世界。

5. **工具的失败模式要先证伪再下结论。** 我差点因为 `uclient-fetch` 不支持 TLS，就得出"设备没有外网"的错误结论，进而把整个方案建在 Mac 推送上。**换一个工具再试一次**，成本极低，收益可能是整个架构。

6. **24MB 的 flash 上，日志和数据库才是长期住户。** 装得下不等于活得下去。

7. **写文件前想清楚"删掉正在被写的文件"会发生什么。** fd 还开着时删除，省不下空间，还丢数据。

8. **判断"这份数据该不该更新"之前，先搞清楚它被谁读。** 我一度用"字段数不兼容"否决了 YSF 列表的更新，理由其实站不住——`strings` 全量扫一遍二进制，会发现除了下载脚本没有任何程序引用它。"格式不兼容"这种结论，得出结论的**依据**比结论本身更该被复查。

9. **连通性测试先跑对照组。** 本机透明代理会让"随机 IP + 随机端口"都返回连接成功，一组测试全绿但毫无信息量。加一行对照（随机 IP 是否也连得上）就能识破，成本是几秒钟。

---

## 附录 A：常用命令速查

```sh
# —— 免密 SSH（已装公钥）——
ssh root@192.168.1.1 'cat /etc/MMDVM.ini'

# —— 推送文件（Mac 的 scp 走 SFTP，路由器没有 sftp-server）——
ssh root@192.168.1.1 'cat > /tmp/x.bin' < local.bin
tar czf - -C ~/Downloads mmdvm-ipk | ssh root@192.168.1.1 'cd /tmp && tar xzf -'

# —— 段内安全改 ini ——
awk '/^\[/ {sec=$0}
     /^Id=/ {if (sec=="[General]"){print "Id=460042";next}}
     {print}' /etc/MMDVM.ini > /tmp/i.new && cat /tmp/i.new > /etc/MMDVM.ini

# —— 看 MMDVM 日志 ——
tail -f /var/log/mmdvm/MMDVM-$(date +%Y-%m-%d).log

# —— 验证主站域名是否还活着（在路由器上测，DNS 才是真的）——
nslookup 4602.master.brandmeister.network

# —— 查询 ID 库 ——
ubus -t3 call dmrid get_by_callsign '{"callsign":"BG2IWK"}'

# —— 手动更新 YSF 反射器表（refcheck 每呼号每文件每小时限一次，别乱试）——
sh /usr/sbin/mmdvm-ysfhosts-update.sh
tail -5 /var/log/mmdvm/ysfhosts-update.log

# —— 确认这个文件到底被谁读（结果：只有下载脚本自己命中）——
for b in /usr/sbin/* /usr/bin/*; do
  [ -f "$b" ] || continue
  s=$(strings "$b" 2>/dev/null | grep -icE "ysfhosts")
  [ "$s" != "0" ] && echo "$b -> $s hits"
done
```

## 附录 B：文件清单

**路由器**

```
/etc/MMDVM.ini                        主配置
/etc/config/mmdvm                     UCI 层配置
/etc/mmdvm/DMRIds.dat                 ID 库（318,130 条，每日自动更新）
/etc/mmdvm/BMMasters.txt              BM 主站表（42 个，DNS 实测存活）
/etc/mmdvm/FCSRooms.txt               FCS 房间表（3,004 条）
/etc/mmdvm/{P25,NXDN}Hosts.txt         镜像自带，保持原样
/etc/mmdvm/YSFHosts.txt               YSF 反射器表（1,508 条，每日自动更新）
/etc/mmdvm/RSSI.dat                   射频标定，未动
/usr/sbin/mmdvm-dmrid-update.sh       ID 库自动更新
/usr/sbin/mmdvm-ysfhosts-update.sh    YSF 反射器表自动更新
/usr/sbin/mmdvm-log-clean.sh          日志自动清理
/etc/crontabs/root                    三条定时任务
/usr/lib/lua/luci/ini.lua             已打补丁（.orig 为原版备份）
/usr/lib/lua/luci/model/cbi/mmdvm/config_general.lua  已打补丁
/var/log/mmdvm/                       日志（只保留当天）
```

**Mac**

```
~/Downloads/MMDVM.ini.br4sn.final     最终配置备份
~/Downloads/DMRIds.dat.latest-4col    4 列 ID 库快照
~/Downloads/mmdvm-ipk/                39 个 ipk + 4 个脚本
~/Downloads/mmdvm-ipk.tar.gz          包集归档
~/Downloads/mmdvm-dmrid-update.sh     Mac 端 ID 库更新
~/Downloads/mmdvm-hosts-update.sh     Mac 端服务器列表更新
~/Downloads/mmdvm-ysfhosts-update.sh  YSF 反射器表更新脚本源码
~/Downloads/mmdvm-log-clean.sh        日志清理脚本源码
~/Downloads/YSFHosts.refcheck-20261109.txt  新版 YSF 表快照（1,508 条）
```

---

*本文所述配置与固件均已实测运行正常。文中所有日志片段与命令均来自实际调试过程。*
