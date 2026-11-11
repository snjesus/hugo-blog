---
title: "把桌面调到「刚刚好」：Edge 缩放、微信 QQ、屏保与 GitHub 的一次实录"
date: 2026-11-11
tags: [Omarchy, Arch, Linux, Edge, 缩放, AUR, 微信, QQ, Wayland, 电源管理, GitHub]
description: "一天里连着调四样东西：把 Edge 界面和网页整体缩小一档、在 Omarchy 上装微信和 QQ、把屏保从 2.5 分钟延到 15 分钟、再把 GitHub 接上。中间踩了五个关于『Linux 桌面字号』的坑，最后发现真正的开关只有一个。"
---

# 把桌面调到「刚刚好」

> 接着前两篇《在 Omarchy 上折腾一整天》《中文文档在 Linux 上字体总是不对？》
> 时间：2026-11-11 08:23 — 15:21（HKT）
> 目标：把 Edge 字号调小一档、装微信和 QQ、屏保改 15 分钟、接上 GitHub

这一天基本都在跟「桌面到底怎么才算顺手」较劲。四件事听起来互不相干，实际串在一条线上：**Omarchy 是 Wayland 纯环境，很多桌面软件的默认行为都要重新校准一遍。**

先把四条结论放前面：

| 目标 | 最终方案 | 关键数字 |
|---|---|---|
| Edge 字号 | `--force-device-scale-factor=0.9` | 界面 91.7%、正文 90.0% |
| 微信 + QQ | `omarchy pkg aur add` + Wayland 修复包 | 体检 20 项通过 |
| 屏保 | `shell.json` 里 `idle.screensaver` 改 900 秒 | 15 分钟息屏、17.5 分钟锁屏 |
| GitHub | GitHub App 连接器 | 身份 `snjesus`，可见仓库 2 个 |

---

## 一、Edge 字号：五个错路，一条正路

### 1.1 需求在变，路也就变了

一开始用户只说「Edge 浏览器整体字号偏大」，我按常规做法准备改**页面缩放**（100% → 90%）。改完用户补了一句：

> edge 菜单栏的字体偏大

于是需求从「网页内容」变成了「界面 + 网页都要小一档」。这一句直接把技术路线从「改一个 pref」推到了「改渲染缩放」。

### 1.2 环境勘察

```console
$ which microsoft-edge-stable
/usr/bin/microsoft-edge-stable          # shell 脚本

$ head -3 /usr/bin/microsoft-edge-stable
#!/usr/bin/env sh
# Launches MS Edge with flags specified in $XDG_CONFIG_HOME/microsoft-edge-beta-flags.conf

$ ls ~/.config/microsoft-edge/Default/Preferences
~/.config/microsoft-edge/Default/Preferences    # 唯一 profile：Default
```

从 Edge 二进制里 `strings` 出缩放相关的 pref 名：

```console
$ strings /opt/microsoft/msedge/msedge | grep -i "default_zoom_level\|per_host_zoom"
partition.default_zoom_level
partition.per_host_zoom_levels
```

按 Chromium 的约定，缩放 level = `log(factor) / log(1.2)`，所以 90% 对应：

```console
$ python3 -c "import math; print(math.log(0.9)/math.log(1.2))"
-0.5778829311823857
```

用文本级插入（而不是整体重写 JSON）写进 `Preferences`，`json.loads` 校验通过：

```json
"partition":{"default_zoom_level":{"x":-0.5778829311823857}}
```

**这一条是生效的**——三组对照实验（A: GDK 0.9 + 缩放 90%、B: 无 GDK + 缩放 90%、C: 无 GDK + 无缩放）里，B vs C 的网页区域像素差分 6373（1.1%），确认 90% 缩放确实改变了页面文字大小。

但它**只管网页，不管界面**。菜单栏还是那个字号。

### 1.3 错路一：fontconfig 规则

Chromium 在 Linux 上走 Skia + fontconfig，理论上可以用 fontconfig 规则改界面字体。建了 `~/.config/fontconfig/conf.d/99-edge-ui-font.conf`，用 `prgname` 匹配进程：

```xml
<match target="pattern">
  <test name="prgname"><string>msedge</string></test>
  <edit name="pixelsize" mode="assign_replace"><double>13</double></edit>
</match>
```

诊断确认规则**命中了**（`FC_DEBUG=4` 能看到 `prgname=msedge` 进了 `FcConfigSubstitute(Font) Pattern`）。但视觉像素差分只有 **0.7%**——噪声级别，等于没变。

原因很硬：Skia 不采用 fontconfig 回填的 `pixelsize` 和 `matrix`，它按自己的像素尺寸渲染。顺带两个坑：

- `mode="assign"` 只是**追加**不是替换，要真替换得用 `assign_replace`
- `<matrix>` 语法在 fontconfig 2.18.3 下直接报 `Missing values in matrix element`

### 1.4 错路二：GTK 主题字号

```ini
# ~/.config/gtk-3.0/settings.ini
[Settings]
gtk-font-name=Adwaita Sans 9
```

Chromium 不用 GTK 渲染界面，这行只影响真正的 GTK 程序。实测差分同样 0.7%，无效。这个文件后来删掉了。

### 1.5 错路三：`GDK_DPI_SCALE=0.9`

`GDK_DPI_SCALE` 是 GTK3 的缩放环境变量，看起来正好对症。做了个严格的对照实验：固定窗口几何 `--window-size=1400,900 --window-position=600,60`，页面用一张带品红标记条的本地 HTML，两组截图用**完全相同的裁剪坐标**做标记条 bbox 测量：

```console
base : 452 27 +7 +120
gdk  : 452 27 +7 +120      ← 逐像素一致
```

而且去主进程环境里翻：

```console
$ tr '\0' '\n' < /proc/<pid>/environ | grep GDK
（什么都没有）
```

Chromium 根本不消费这个变量。**这条路是死的**，之前基于它做的 `~/.local/bin` 包装脚本、`~/.bashrc` 别名、桌面文件 Exec 覆盖全部无效，后来全部撤销了。

### 1.6 错路四：`--force-device-scale-factor=1.35`

换 Chromium 官方开关试试。1.35 是按「输出缩放 1.5 × 0.9」算出来的，理论上该缩小 10%。实测：

```console
f135 : 标记条 362 35 +10 +163
```

标记条从 27px 变成 35px，界面上方高度从 120px 涨到 163px——**界面反而变大了**。方向反了。

### 1.7 正路：`--force-device-scale-factor=0.9`

```console
f09  : 标记条 496 22 +8 +110
```

界面 120px → 110px，网页标记条 27px → 22px，两个都小了。

关键在于这个开关怎么送达每一个启动入口。读 `/usr/bin/microsoft-edge-stable` 发现它本来就会读一个配置文件：

```bash
XDG_CONFIG_HOME="${XDG_CONFIG_HOME:-${HOME}/.config}"
if [ -r "${XDG_CONFIG_HOME}/microsoft-edge-stable-flags.conf" ]; then
  EDGE_USER_FLAGS="$(cat "${XDG_CONFIG_HOME}/microsoft-edge-stable-flags.conf")"
fi
exec /opt/microsoft/msedge/microsoft-edge $EDGE_USER_FLAGS "$@"
```

而桌面启动器的三行 Exec 也全走这个脚本：

```console
$ grep -E "^Exec=" /usr/share/applications/microsoft-edge.desktop
Exec=/usr/bin/microsoft-edge-stable %U
Exec=/usr/bin/microsoft-edge-stable
Exec=/usr/bin/microsoft-edge-stable --inprivate
```

所以只要写一个文件，终端、桌面图标、应用启动器三条路全部生效：

```bash
# ~/.config/microsoft-edge-stable-flags.conf
--force-device-scale-factor=0.9
```

实测启动命令行确实带上了：

```console
$ tr '\0' ' ' < /proc/<pid>/cmdline
/opt/microsoft/msedge/msedge --force-device-scale-factor=0.9 --no-first-run ...
```

### 1.8 最终数据

界面高度取「窗口顶部到网页内容起始」的物理像素，正文字号取文字像素包围盒：

| 指标 | 原始 | 调整后 | 比例 |
|---|---|---|---|
| 界面（标签栏+工具栏）高度 | 120 px | 110 px | 91.7% |
| 网页正文（宽） | 297 px | 267 px | 89.9% |
| 网页正文（高） | 22 px | 20 px | 90.9% |

正好小一档。

### 1.9 一个值得记的坑：Edge 会把旧设置写回来

改完 pref 之后重启 Edge 验证，发现 `partition` 又回来了：

```console
partition = {'default_zoom_level': {'x': -0.5778829311823857}}
```

原因是：**Edge 退出时会把内存里的旧配置整个写回磁盘**。我是在 Edge 运行时改的文件，它退出时用内存快照覆盖了我的修改。

正确顺序是：**先关 Edge → 再改 Preferences → 再启动验证**。改完再关一次，验证 `partition = None` 且没有回弹，这才算收干净。

同时删掉的 90% 页面缩放也是为了这个：它和 `--force-device-scale-factor=0.9` 会叠加，内容就变成 81% 了。统一由缩放开关控制，一个来源。

### 1.10 回滚

```bash
rm ~/.config/microsoft-edge-stable-flags.conf
```

---

## 二、装微信和 QQ

### 2.1 先查 AUR，别猜包名

```bash
omarchy pkg --help
#   omarchy pkg add <packages...>            安装缺失的 Arch 包
#   omarchy pkg aur add <packages...>        从 AUR 安装
```

`omarchy pkg aur add` 内部就是 `yay -S --noconfirm --needed`。

搜 AUR 时微信的候选有 18 个——`wechat-universal-bwrap`(149 票)、`wechat-bin`(28)、`wechat`(51)、`wechat-appimage`(23)、`tencent-wechat`(1)。QQ 那边 `linuxqq`(113 票)、`linuxqq-nt-bwrap`(50)、`linuxqq-wayland-fix`(0 票但是新)。

选型逻辑：

- 微信要 **`wechat-universal-bwrap`**：腾讯官方 4.1.13 通用版 + bubblewrap 沙箱（Electron 应用在 Wayland 下需要），票数最高
- QQ 要 **官方 `linuxqq`** + **`linuxqq-wayland-fix`** 修复层

### 2.2 一个惊喜：QQ 根本不用从 AUR 装

```console
$ yay -S --noconfirm --needed -- wechat-universal-bwrap linuxqq linuxqq-wayland-fix
...
sudo pacman -S --needed --noconfirm --config /etc/pacman.conf -- extra/patchelf extra/flatpak-xdg-utils omarchy/linuxqq
```

`omarchy/linuxqq` —— Omarchy 自带一个官方仓库：

```toml
# /etc/pacman.conf
[omarchy]
Server = https://pkgs.omarchy.org/stable/$arch
```

所以 QQ 是从 `pkgs.omarchy.org` 装的，版本 `3.2.34_53644-1`，根本不需要编译 AUR 包。真正要编译的是微信（753.74 MiB）和 Wayland 修复层。

### 2.3 Wayland 才是真正的坑

Omarchy 是纯 Wayland，QQ 的屏幕共享、共享电脑声音、剪贴板、截图在 Wayland 下全都有问题。`linuxqq-wayland-fix` 就是专门治这个的，它的工作方式是把四个注入库用 `LD_PRELOAD` 塞进 QQ：

```bash
export LD_PRELOAD="$preload${LD_PRELOAD:+:$preload}"
export XDG_SESSION_TYPE=x11      # 让界面和音视频 SDK 不拦截共享
export MESA_SHADER_CACHE_DISABLE=true   # QQ 自带的 ZSTD 符号会劫持 Mesa 缓存
exec "$QQ" --ozone-platform=wayland "${angle_arg[@]}" "$@"
```

它还带了个 `install` 子命令，把菜单里的「QQ」直接指向修复版：

```console
$ linuxqq-wayland-fix install
✓ 菜单里的「QQ」改为启动修复版
✓ 隐藏「QQ（Wayland修复版）」
恢复原样：linuxqq-wayland-fix uninstall
```

效果是 `~/.local/share/applications/qq.desktop` 的 Exec 变成 `linuxqq-wayland-fix %U`，菜单里只剩一个干净的「QQ」。

### 2.4 体检

```console
$ linuxqq-wayland-fix --doctor
✓ 屏幕共享修复: /usr/lib/linuxqq-wayland-fix/libqq-wl-portal.so
✓ 剪贴板修复: /usr/lib/linuxqq-wayland-fix/libqq-clipbridge.so
✓ 截图防闪退: /usr/lib/linuxqq-wayland-fix/libqq-screenshot.so
✓ Wayland 会话: wayland-1
✓ ANGLE 后端: vulkan（检测到 Vulkan，可规避观看共享花屏）
✓ Portal: xdg-desktop-portal 1.22.1-2 运行中，ScreenCast 后端 hyprland.portal
[正在运行的 QQ]
✗ QQ（PID 345802）没有注入：请完全退出 QQ（含托盘），再从「QQ（Wayland修复版）」打开
```

那次失败是因为我之前用原版 `qq` 起过一个实例还在跑。彻底清掉再用修复版启动：

```console
✓ QQ（PID 346692）已注入
```

微信这边：

```console
$ wechat-universal
Using '/home/snjesus/Documents/WeChat_Data' as WeChat Data folder
IME workaround for fcitx applied
```

窗口正常弹出，二维码登录页渲染完整；QQ 也原生跑在 Wayland 上：

```console
$ ps -o args -p <pid>
/opt/QQ/qq --ozone-platform=wayland --enable-wayland-ime
```

### 2.5 两条工具层面的教训

**`pkill -f` 会杀掉自己。** 清理 QQ 进程时：

```bash
pkill -9 -f "/opt/QQ/qq"     # ❌ 这条命令行本身就含 "/opt/QQ/qq"，把自己的 shell 也杀了
```

括号技巧：

```bash
pkill -9 -f "/opt/QQ/q[q]"   # ✅ 匹配 QQ 进程但不匹配自己
```

**从 agent 的 bash 里启动的程序会被进程组清理掉。** 第一次启动应用，窗口确实弹出来了，但下一条命令就没了。要用 `setsid` 脱离会话：

```bash
setsid nohup linuxqq-wayland-fix >/dev/null 2>&1 </dev/null & disown
```

---

## 三、屏保：15 分钟

### 3.1 找对地方

Omarchy 没有 `hypridle.conf`，屏保也不走 swayidle。真正的实现是 Quickshell 里的 idle 服务：

```bash
$ ls /usr/share/omarchy/shell/plugins/services/
battery  idle  media  nightlight  remote-session
```

配置在 `~/.config/omarchy/shell.json`：

```json
{
  "version": 1,
  "idle": {
    "screensaver": 150,
    "lock": 300
  },
  ...
}
```

单位是**秒**。官方文档写得很清楚：

> Set `idle.screensaver` and `idle.lock` in `~/.config/omarchy/shell.json`, in seconds since user idle began.

### 3.2 锁屏必须一起改

这是这次最容易被忽略的一点。看服务源码 `IdleModel.js`：

```javascript
function firstIdleTimeout(screensaverSeconds, lockSeconds) {
  var times = []
  if (screensaverSeconds > 0) times.push(screensaverSeconds)
  if (lockSeconds > 0) times.push(lockSeconds)
  // 取两者较小值
}

function delayAfterFirstIdle(timeoutSeconds, firstIdleSeconds) {
  var delay = timeoutSeconds - firstIdleSeconds
  return delay > 0 ? delay : 0
}
```

也就是说：空闲判定时刻 = `min(屏保, 锁屏)`，两个动作各自再算相对延迟。

如果只把屏保改成 900 而锁屏还是 300，那么 **5 分钟就锁屏了**，和「15 分钟才息屏」的预期完全冲突。所以锁屏必须跟着往后挪。保持出厂那 150 秒的间隔：

```json
"idle": {
  "screensaver": 900,
  "lock": 1050
}
```

推导一下最终时序：

- 空闲判定 = min(900, 1050) = **900 秒**
- 屏保延迟 = 900 − 900 = 0 → 空闲满 15 分钟**立即息屏**
- 锁屏延迟 = 1050 − 900 = 150 → **17.5 分钟锁屏**

### 3.3 生效

```bash
$ omarchy restart shell
$ journalctl --user -n 60 --no-pager | grep "omarchy idle"
omarchy idle 2026-11-11T07:12:52.004Z service-ready
omarchy idle 2026-11-11T07:12:52.500Z stay-awake: disabled state-file
omarchy idle 2026-11-11T07:12:53.176Z idle-monitor: active
```

`stay-awake: disabled state-file` 说明「保持唤醒」那个开关没有生效（`~/.local/state/omarchy/indicators/stay-awake` 不存在），计时器是真的在跑。

### 3.4 系统那一层的电源行为

查 logind 的实际生效值：

```console
$ busctl get-property org.freedesktop.login1 /org/freedesktop/login1/seat/seat0 \
    org.freedesktop.login1.Seat IdleAction
（空）
```

两项都没显式设置，走 systemd 默认：

| 行为 | 当前值 | 说明 |
|---|---|---|
| 空闲后自动挂起 | `ignore` | 系统**不会**因空闲自动 suspend，只有 Omarchy 自己的息屏/锁屏 |
| 合盖 | `suspend` | 默认休眠 |
| 电源键 | `ignore` | Omarchy 装的 `/etc/systemd/logind.conf.d/10-ignore-power-button.conf` |

---

## 四、接上 GitHub

### 4.1 工具不在直接列表里

`tool_search` 查 "github" 是空的——GitHub App 的工具不在模型直接工具表里。按规矩走连接器发现：

```bash
$ mcode-tools connector tools --keyword github
{
  "tools": [
    { "tool_name": "connector__github__actions_get", ... },
    { "tool_name": "connector__github__create_branch", ... },
    ...
```

一共 **135** 个 GitHub 工具，从 `get_me` 到 `actions_get` 全都有。

### 4.2 身份

```console
$ mcode-tools connector call connector__github__get_me --args '{}'
{"login":"snjesus","id":52002614,"profile_url":"https://github.com/snjesus",...}
```

| 项 | 值 |
|---|---|
| 登录名 | `snjesus` |
| 用户 ID | 52002614 |
| 主页 | https://github.com/snjesus |
| 账号创建 | 2019-06-19 |
| 公开仓库 | 18 |

### 4.3 可见范围只有 2 个

```console
$ mcode-tools connector call connector__github__search_repositories \
    --args '{"query":"user:snjesus","perPage":50}'
返回条数: 2 | total_count: 2
  hugo-blog | fork: False
  vercel    | fork: False
```

但账号资料明确写着 18 个公开仓库。这个差值说明 **GitHub App 的安装授权范围只覆盖这 2 个仓库**。要在 GitHub 侧给这个 App 扩大安装范围（安装时选 "Only select repositories" 的需要补勾）。

当前工具集里**没有**枚举 installation 授权清单的工具，所以给不出完整的已授权仓库列表——只能发现实际能访问到的目标。

| 仓库 | 可见性 | 主分支 | 语言 | 最后推送 |
|---|---|---|---|---|
| `snjesus/hugo-blog` | public | `main` | HTML | 2026-11-10 |
| `snjesus/vercel` | private | `main` | JavaScript | 2026-08-20 |

### 4.4 gh CLI 没登录

```console
$ gh --version
gh version 2.103.3
$ gh auth status
You are not logged into any GitHub hosts.
```

装了 gh 但没有身份，所以**不能用 `gh` 代替连接器操作**。另外 App 授权不会自动给 shell 提供 git 凭据，真要 clone/push 得另外配 SSH key。

### 4.5 记录

写了 `~/Work/github-account-2026-11-11.md`，包含身份、调用方式、常用工具对照表、标准操作流程和上面这两个限制。里面**不含任何 token 或 secret**——连接凭据由宿主管理，不导出到文件、命令行或提交内容里。

---

## 五、今天留下的几个通用结论

1. **Linux 桌面的「字号」不是一个开关。** 字体家族、fontconfig、GTK 主题、页面缩放、设备缩放是五件不同的事，各管各的。Chromium 界面只认设备缩放。

2. **Chromium 系应用在 Wayland 下要单独照顾。** Electron 应用需要 `--ozone-platform=wayland` 和沙箱处理，剪贴板/截图/屏幕共享各需要单独的补丁。这不是 bug，是 Wayland 的安全模型决定的。

3. **改配置前先确认谁来写它。** Edge 退出时会覆盖 Preferences，`omarchy refresh` 会覆盖配置文件，pacman 会覆盖被管理的包。任何「我改了但不生效」都该先查一下有没有人在抢着写回。

4. **App 的可见范围 ≠ 账号的全貌。** 账号有 18 个公开仓库不等于连接器能看到 18 个——安装授权范围是另一层开关。

---

*本文所有命令、报错和数字均为 2026-11-11 当天的真实输出。*
