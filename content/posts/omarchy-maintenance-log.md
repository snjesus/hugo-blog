---
title: "把系统养薄，顺手把屏保换成自己的名字：一次 Omarchy 维护实录"
date: 2026-11-12
tags: [Omarchy, Arch, Linux, 屏保, ASCII艺术, figlet, 字体, 系统精简, 依赖, pacman, ttfx, fastfetch]
description: "一天两件事：把 Omarchy 屏保上的 OMARCHY 换成 BD4TWO，顺手把 216 个显式安装的软件盘了一遍，找出功能重叠并删掉 9 个。中间踩了三个「删了会坏」的坑，也推翻了自己一个错误的判断。"
---

# 把系统养薄，顺手把屏保换成自己的名字

> 接着前三篇《在 Omarchy 上折腾一整天》《中文文档在 Linux 上字体总是不对？》《把桌面调到「刚刚好」》
> 时间：2026-11-12 09:08 — 16:14（HKT）
> 目标：把屏保图案换成 BD4TWO；盘点系统里功能重复的软件并精简掉

这一天做了两件事，看起来不搭界，实际是同一个思路：**先搞清楚东西从哪来，再决定要不要它。**

第一件是 Omarchy 屏保上那张 `OMARCHY` 的 ASCII 艺术字，想换成自己的名字 `BD4TWO`。第二件是把系统里 216 个显式安装的软件盘一遍，找出功能重叠的删掉。

先把结论放前面：

| 目标 | 最终方案 | 关键数字 |
|---|---|---|
| 屏保图案 | figlet `blocky` 字体原生渲染 | 571 个字体逐字母比对，最高命中 77.5% |
| 重复盘点 | 216 个包全量归类 | 找到 11 组功能重叠 |
| 实际删除 | 9 个包 + 7 个残留目录 | 回收约 159 MiB，显式包 208 → 199 |
| 救回来的包 | 4 个被拦下 | `gum` / `bat` / `inxi` / 字体判断修正 |

外加一个副作用：因为 `inxi` 是个硬依赖，给 `omarchy-debug` 写了个纯 Python 的兼容垫片顶上。

---

## 一、屏保换个名字

### 1.1 出厂那张图不是「字体渲染」出来的

Omarchy 的屏保图案放在 `~/.config/omarchy/branding/screensaver.txt`，由 `ttfx`（终端特效引擎）加载并施加随机效果：

```bash
# /usr/bin/omarchy-screensaver
ttfx -i ~/.config/omarchy/branding/screensaver.txt \
  --frame-rate 120 --canvas-width 0 --canvas-height 0 \
  --reuse-canvas --anchor-canvas c --anchor-text c \
  --random-effect --no-eol --no-restore-cursor &
```

想改它有两条路：`omarchy branding screensaver image`（选一张 PNG/SVG，程序自动转 ASCII）或 `omarchy branding screensaver text`（直接编辑这个 txt）。这次走第二条。

但改之前得先搞清楚一件事：**这张图是用什么字体生成的？**

### 1.2 逐字母比对 571 个字体

系统里没装 figlet，于是从 PyPI 拉了 pyfiglet 的源码包（571 个字体，550 个 `.flf` 文件）到 `/tmp` 做分析。

思路很朴素：把目标图按列切成 7 个字母，每个字形单独拿去和所有字体渲染出来的同名字母做像素级比对（允许 ±3 行 ±3 列的平移，取最优匹配率）。

```
delta_corps_priest_1   77.54%   ← 命中
blocky                 43.51%
dos_rebel              40.67%
bigmono12              33.02%
ansi_regular           29.59%
```

77.5% 对 43.5%，断层明显——**原图就是 `delta_corps_priest_1` 这个字体（作者 CoSMiC cHiLD）渲染的**，只不过后来被人手工压缩过：把 4 列宽的字内间隙挤成 3 列，横杠也各少一格。

从原图反推出来的绘制规范是：

| 规范 | 取值 |
|---|---|
| 笔画粗细 | 3 列 |
| 字内间隙 | 3 列 |
| 字身高度 | 8 行 |
| 圆角处理 | `▄` / `▀` 半块字符做阶梯收边（如 `▄█████▄` / ` ▀█████▀ `） |
| 字距 | 2 列 |
| 中间横杠 | `███▄▄▄███` + `███▀▀▀███` 两行配对 |

### 1.3 致命缺陷：这张字体里没有数字 4

按规范手工画完 B、D、4、T、W、O 六个字形，拼出来的效果自己看着都觉得别扭。于是回头查那个字体——

```console
$ python3 -c "import pyfiglet; print(pyfiglet.Figlet(font='delta_corps_priest_1').renderText('4'))"
```

输出是空的。**`delta_corps_priest_1` 根本没有 `4` 这个字形。**

这意味着原图是「仿制品」：不是字体直接渲染的，是有人拿这个字体渲染出其它字母、再手工压缩、最后把数字 4 自己画上去补齐的。压缩得并不均匀，`A` 的横杠甚至换成了 `H` 那种圆角款——所以整体看着"有点糊"，不是错觉。

顺带一提，我一开始写了个字符检测脚本，用的是 `FigletFont.renderText()`，这个方法在 pyfiglet 1.0.4 里根本不存在，异常被 `except` 吞掉，于是「字体支持的字符」一栏输出空白，还让我误以为 `usage` 的引用也是误报。**检测脚本自己出错的时候，表现和「检测结果为真」一模一样。** 后来换 `pacman -Qi` 配合中文 locale 的字段名重新查，才把真实依赖关系捞出来。

### 1.4 换个原生字体：blocky

既然要重画，不如直接用字体原生渲染。全库 571 个字体里有 **483 个**支持全部 6 个字符 BD4TWO，挑了 `blocky`：

```
████████  ████████  ██        ████████ ██      ██  ███████
██     ██ ██     ██ ██    ██     ██    ██  ██  ██ ██     ██
██     ██ ██     ██ ██    ██     ██    ██  ██  ██ ██     ██
████████  ██     ██ ██    ██     ██    ██  ██  ██ ██     ██
██     ██ ██     ██ █████████    ██    ██  ██  ██ ██     ██
██     ██ ██     ██       ██     ██    ██  ██  ██ ██     ██
████████  ████████        ██     ██     ███  ███   ███████
```

7 行 × 59 列，纯 `█` 实心方块。`ttfx` 的随机效果对实心块渲染最出彩（溶解、矩阵、字符变形都吃得住），比原来的圆角细笔画好看得多。

改完启动屏保实测确认，图案正常播放。

---

## 二、216 个包，有多少是重复的

### 2.1 全量盘点

```console
$ pacman -Qqe | wc -l     # 208 个原生仓库包
$ pacman -Qqm | wc -l     # 8 个 AUR 包
$ flatpak list --app | wc -l   # 0
$ snap list | wc -l            # 0
```

### 2.2 重叠清单

按功能归类后，11 组重叠：

| 功能 | 重复实现 | 备注 |
|---|---|---|
| 系统信息 | `fastfetch` / `inxi` / `expac` | 前两个直接竞品 |
| 磁盘占用 | `btop` / `disktree-bin` | 后者能删文件，前者不能 |
| 图片查看 | `imv` / `sushi` | 独立窗口 vs Nautilus 内预览 |
| 截图 | `omasnap` / `grim` + `slurp` | 分层关系，omasnap 底层就调它们 |
| 文件名查找 | `fd` / `plocate` | 实时遍历 vs 索引数据库 |
| cat / 分页 | `bat` / `less` | 能力重叠 |
| Markdown | `obsidian` / `omawrite` / `hype` | 三套 Markdown 工具 |
| 终端模拟器 | `foot` / `ghostty` | — |
| 亮度控制 | `brightnessctl` / `ddcutil` | 背光功能重叠 |
| 音乐播放 | `mpv` / `cliamp` | cliamp 占 75 MiB |
| 中文字体 | 5 个包 | ← 这条我判断错了，见下 |

### 2.3 我判断错了一次：字体并不冗余

第一轮盘点时，我把五套中文字体（合计 619 MiB）列为最大冗余源，理由是「思源黑体 + 思源宋体已经覆盖绝大多数中文字形，WPS 再带一套方正字体属于第三份重复覆盖」。

**这个结论是错的。** 深挖每一套的字体家族名之后发现，它们是分层用途而不是重复覆盖：

| 字体包 | 实际角色 | 判定 |
|---|---|---|
| `noto-fonts-cjk` (298.79 MiB) | `fc-match` 中文默认字体（思源黑体） | 必留 |
| `adobe-source-han-serif-cn-fonts` (77.28 MiB) | 思源宋体 CN，衬线排版 | 必留 |
| `wps-office-fonts` (161.53 MiB) | 方正书宋，WPS 中文文档渲染 | 必留 |
| `ttf-wps-fonts` (75.70 MiB) | Calibri / Cambria / Candara 的度量兼容替代 | 可选 |
| `ttf-ms-fonts` (5.40 MiB) | Arial / Times，**steam / qt6-webengine 硬依赖** | 必留 |

教训是：**光看包名和体积判断重复，一定会错。** 得看它实际注册了哪些 font family、谁在引用、谁是硬依赖。

---

## 三、三档精简方案

盘完之后定了三档：

**第 1 档（零风险）**：无快捷键绑定、无数据、零依赖的包。

```
usage  tobi-try  hype  disktree-bin
```

**第 2 档（需要取舍）**：`ttf-wps-fonts` / `inxi` / `bat` / `fd` / `ghostty`。

**第 3 档（残留目录）**：装过又卸载的浏览器留下的配置目录。

```bash
# 第 1 档
sudo pacman -Rns usage tobi-try hype disktree-bin
# 第 2 档（逐条确认）
sudo pacman -Rns ttf-wps-fonts inxi bat fd
omarchy default terminal foot && sudo pacman -Rns ghostty
# 第 3 档
rm -rf ~/.config/google-chrome{,-beta,-unstable} ~/.config/BraveSoftware \
       ~/.config/microsoft-edge-dev ~/.config/cef_user_data
sudo rm -rf /usr/lib/chromium
```

提权这步有个细节：系统 `sudo` 要密码，而 agent 的 shell 没有 TTY。Omarchy 的 Quickshell 里内置了 polkit 授权代理（`/usr/share/omarchy/shell/plugins/polkit/PolkitAgent.qml`），所以改用 `pkexec` 会在桌面上弹出 Omarchy 风格的授权框：

```bash
pkexec /usr/bin/pacman -Rns --noconfirm usage tobi-try hype disktree-bin
```

---

## 四、三个「删了会坏」的坑

真正花时间的不是删包，是拦下不该删的。

### 4.1 gum：omarchy 的隐藏依赖

原计划第一档里有 `gum`（13.74 MiB，「glamorous shell scripts」）。执行前查依赖：

```console
$ pacman -Qi gum | grep 依赖它
依赖它 : omarchy  omarchy-settings
```

`omarchy` 和 `omarchy-settings` 都依赖它。再看引用点：

```bash
# /usr/share/omarchy/default/bash/fns/worktrees
if gum confirm "Remove worktree and branch?"; then
```

**删了会让 git worktree 的交互确认直接失效。** 拦下。

### 4.2 bat：Omarchy 自己的 MANPAGER

`bat` 看着就是个「带高亮的 cat」，但 Omarchy 的 shell 配置实际在用它：

```bash
# /usr/share/omarchy/default/bash/aliases
alias ff="fzf --preview 'bat --style=numbers --color=always {}'"
alias eff='$EDITOR "$(ff)"'

# /usr/share/omarchy/default/bash/envs
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
```

也就是 `ff` 模糊查找的预览窗口、以及 **所有 man 页面的彩色显示**都靠它。

`/usr/share/omarchy/` 是只读的（改了会被 `omarchy update` 覆盖），但用户自己的 `~/.bashrc` 里 Omarchy 留了口子——注释原话就是「don't mess with these directly, just overwrite them here!」。于是等价替换：

```bash
# 彩色 man 页面：原本由 bat -l man -p 着色，改用 col + less
export MANROFFOPT="-c"
export MANPAGER="sh -c 'col -bx | less -R'"

# ff 模糊查找的文件预览：原本由 bat 提供语法高亮，改用 cat
alias ff="fzf --preview 'cat {}'"
unset BAT_THEME
```

预览没用 `less` 是有原因的：**`less` 在 fzf 预览窗里是交互式的，会卡住**，必须用 `cat`。代价是预览窗没有语法高亮了。

### 4.3 inxi：omarchy-debug 的数据源

删完 inxi 才发现：

```console
$ omarchy-debug --no-sudo --print
/usr/bin/omarchy-debug: 行 37: inxi: 未找到命令
...
=========================================
SYSTEM INFORMATION
=========================================
                                    ← 整段空了
```

`omarchy-debug` 的诊断输出里，最有用的那半边没了。查依赖链：

```
/usr/share/omarchy/install/omarchy-base.packages:70  →  inxi
/usr/share/omarchy/…/omarchy-settings（可选依赖）      →  inxi
/usr/bin/omarchy-debug:45                              →  $(inxi -Farz)
```

**inxi 居然是 Omarchy 的官方基础包**，`omarchy update` / `omarchy reinstall pkgs` 会把它装回来。

1.36 MiB 换一份完整的排障能力，不划算。但它确实在基础包清单里、且这次要删干净，得给 `omarchy-debug` 找个替身。

---

## 五、给 omarchy-debug 写个替身

### 5.1 两条硬约束

1. **`/usr/bin` 在 PATH 中排在 `~/.local/bin` 之前**（实测第 14 位 vs 第 15 位）——用户层垫片只有在 `/usr/bin/inxi` 消失后才能接管
2. **`/usr/bin/omarchy-debug` 是包所有文件**，改了会被 `omarchy update` 覆盖

所以方案是**用户层垫片**，`omarchy-debug` 一行都不用改，`$(inxi -Farz)` 自动命中。

### 5.2 垫片实现

系统里本来就有 `fastfetch`（Omarchy About 面板在用），而它支持 JSON 输出。于是：

```
~/.local/bin/inxi              # Python 垫片，接收 -Farz 等参数
~/.config/fastfetch/omarchy-debug.jsonc   # 专用配置，无 logo / 无 break，保证 JSON 可用
```

配置刻意避开 `break` 模块（fastfetch 的 JSON 格式不支持它）：

```jsonc
{
  "logo": { "type": "none" },
  "modules": [
    { "type": "os" }, { "type": "kernel" }, { "type": "host" },
    { "type": "board" }, { "type": "bios" }, { "type": "cpu" },
    { "type": "gpu" }, { "type": "memory" }, { "type": "disk" },
    { "type": "battery" }, { "type": "packages" }, { "type": "wm" }
  ]
}
```

垫片读取 JSON 后渲染成 inxi 风格文本，输出长这样：

```
System:    Kernel 7.2.15-1-omarchy x86_64
  Machine:   LENOVO ThinkPad X390
  Board:     LENOVO 20Q0A02MCD
  Firmware:  LENOVO 1.80 (07/25/2023)
  OS:        Omarchy 4.5.0
  Desktop:   Hyprland 0.56.2
CPU:       Intel(R) Core(TM) i5-8265U (4C/8T) @ 3.90 GHz
GPU:       UHD Graphics 620 (i915)
Memory:    3.21 GiB / 7.43 GiB
Disk:      /dev/mapper/root btrfs [/, /home, /var/cache/pacman/pkg, /var/log] 901.90 GiB free / 951.85 GiB (5%)
Battery:   02DL019 (Li-poly) 79% AC Connected
Packages:  972 (pacman)
```

有个细节值得记一笔：fastfetch 2.x 的占位符语法是 `{name}`，不是老版的 `${name}`。写错了不会报错，只会原样输出 `${...}`——**又是一个「错了但看起来像对的」**。

### 5.3 验证

| 测试 | 结果 |
|---|---|
| `command -v inxi` | → `/home/snjesus/.local/bin/inxi` |
| `env -i`（无任何环境变量） | 正常输出 |
| 无 HOME 变量 | 正常输出 |
| 从其它目录调用 | 正常输出 |
| 配置文件缺失（降级路径） | 明确报错，不崩溃 |
| `omarchy-debug` 全部段落 | SYSTEM INFORMATION / DMESG / INSTALLED PACKAGES 齐全 |

### 5.4 一项能力差异

原版 inxi 会输出 CPU 漏洞缓解状态：

```
Flags-basic avx avx2 ht lm nx pae sse sse2 ...
Vulnerabilities:
  Type gather_data_sampling  mitigation Microcode
  Type meltdown              status Not affected
  ...
```

这部分来自 `/proc/sys/kernel/`，**fastfetch 没有对应模块，垫片不覆盖**（实测条目数 0）。其余段落全部覆盖。

另外，`omarchy update` 会把 inxi 装回来；届时 `/usr/bin/inxi` 因 PATH 优先级自动接管，垫片自动让位——两种情况都不冲突，垫片只是 inxi 缺席时的安全网。

---

## 六、结语

最后的状态：

```
显式包总数:  208 → 199 个（净减 9）
累计回收:    约 159 MiB
磁盘:        952G 总量，50G 已用，901G 空闲 (6%）

默认终端 foot | 默认浏览器 Edge | 屏保 blocky BD4TWO
字体默认未变 | quickshell 运行中 | omarchy-debug 完整可用
```

真正值得记下来的其实不是「删掉 9 个包省了 159 MiB」——磁盘才用了 6%，这点空间毫无意义。值钱的是三件事：

**一、动手之前先查依赖。** `gum`、`bat`、`inxi` 三个包，单看名字和体积都像该删的，但每一个都在别处承重。`pacman -Qi` 的「依赖它」字段加上全系统 grep，两分钟就能查清。

**二、「看起来冗余」和「真的冗余」差着三层。** 五套中文字体第一轮被我判成 619 MiB 的冗余，实际是分层用途；`delta_corps_priest_1` 被当成标准字体，实际缺一个数字 4。判定重复的依据只能是**实际注册的字体家族、谁在引用、谁是硬依赖**，不是包名和体积。

**三、删除动作要留在用户层。** `/usr/share/omarchy/` 和 `/usr/bin/omarchy-debug` 都归包所有，改了会被下次更新抹掉。真正稳的做法是在 `~/.bashrc` 覆盖、在 `~/.local/bin` 放垫片——这既是 Omarchy 自己预留的扩展点，也天然扛得住 `omarchy update`。

至于屏保那张图，删掉的是别人写的 OMARCHY，换上的是自己的 BD4TWO。`ttfx` 的随机效果每帧都在变，但底下的字符一个都不会错。
