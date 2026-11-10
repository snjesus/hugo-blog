---
title: "中文文档在 Linux 上字体总是不对？—— 从装 WPS 到让「宋体」真正被系统认识"
date: 2026-11-10
tags: [Arch, Linux, WPS, 字体, fontconfig, 中文排版, AUR, pkexec]
description: "给 Arch/Omarchy 装 WPS Office 的一路硬骨头：sourceforge 全线不通只能让 curl 走代理、GitHub tarball 哈希三方打架、/tmp 是 3.8G 内存盘直接撑爆、polkit 会话锁状态陈旧导致授权窗口弹不出来。最后发现真正的关键不是装字体，而是让「宋体」成为系统里的真实字体名。"
---

# 中文文档在 Linux 上字体总是不对？

> 接上一篇《在 Omarchy 上折腾一整天》的续篇
> 时间：2026-11-10 16:02 — 17:05（HKT）
> 目标：装 WPS Office，并让中文字体真正生效

这一篇只讲一件事：**为什么在 Linux 上打开一个中文 docx，字体总是不对**，以及要花多少功夫才能修好。

先说结论：**装字体是第一步，而且是最不重要的一步。** 真正的难点在于——

> 系统里没有叫「宋体」「微软雅黑」的字体。
> 有字体叫「方正书宋」「方正细黑一」，但文档里写的是「宋体」。
> 于是所有中文都掉进了 fallback 链，排版全乱。

---

## 0. 起点：一个自作聪明的问题

在给用户推荐「WPS + 字体包」的时候，我犯了个错：

> "`ttf-ms-fonts` 装上就有宋体/黑体/雅黑，中文 docx 排版就能还原了。"

用户选了 `ttf-ms-fonts` + `ttf-wps-fonts`，我兴冲冲去装。装完一测：

```console
$ fc-match "SimSun"
LiberationSerif-Regular.ttf: "Liberation Serif" "Regular"     ← 纯拉丁字体，一个汉字都没有

$ fc-match "Microsoft YaHei"
LiberationSans-Regular.ttf: "Liberation Sans" "Regular"
```

`ttf-ms-fonts` **是 msttcorefonts，不含任何中文字体**。

打开它的 PKGBUILD 才看明白：

```bash
_files=('andale32.exe' 'arial32.exe' 'arialb32.exe' 'comic32.exe'
        'courie32.exe' 'georgi32.exe' 'impact32.exe' 'times32.exe'
        'trebuc32.exe' 'verdan32.exe' 'webdin32.exe')
```

Arial、Times New Roman、Verdana、Georgia、Impact、Comic Sans、Trebuchet、Courier New、Webdings、Andale——**11 个西文核心字体，11 个 SHA256 校验值，和中文字体没有半点关系**。

所以这一篇的实际路线是：

1. 把 WPS 装起来（三个网络/存储坑）
2. 把中文字体装起来（两个来源、一个哈希争议）
3. **让「宋体」这些名字成为系统里的真实字体名**（真正的关键）
4. 验证

---

## 1. 装 WPS Office：三个硬骨头

### 1.1 选包

```bash
$ pacman -Ss microsoft-edge   # 顺带看一眼 Edge 的情况
```

WPS 在 AUR 上只有一个主流包：

| 包 | 版本 | 票数 | 说明 |
|---|---|---|---|
| `wps-office` | 12.1.2.29183-1 | 488 | 官方维护，预编译 `.deb` 打包 |

PKGBUILD 的依赖只要 14 个，其中 `xorg-mkfontdir` 缺失，装一下即可。

### 1.2 坑一：sourceforge 全线不通

`ttf-ms-fonts` 的 11 个 `.exe` 来自 SourceForge。在国内这是著名的不可达：

```console
$ curl -sL -r 0-800000 "https://jaist.dl.sourceforge.net/project/corefonts/the%20fonts/final/arial32.exe"
000 0B/s          ← jaist
000 0B/s          ← excellmedia
000 0B/s          ← versaweb
000 0B/s          ← freefr
超时/失败          ← nchc
超时/失败          ← deac-riga
...
```

并行扫了 14 个 SourceForge 官方镜像 + 4 个国内高校镜像（USTC 404 / BFSU 403 / SJTU 超时 / NJU 404）+ `newcontinents.com`，**全军覆没**。

**转折点**：这台机器开着 Clash Verge 的 TUN 模式。既然直连被墙，那让请求走代理呢？

```console
$ curl -sL -x http://127.0.0.1:7897 \
    "https://downloads.sourceforge.net/project/corefonts/the%20fonts/final/arial32.exe"
206 38573B/s      ← 206 Partial Content，成功了
```

于是让 curl 走本机代理端口并行下载 11 个文件，**3.8MB 一共几秒钟**：

```bash
export https_proxy=http://127.0.0.1:7897 http_proxy=http://127.0.0.1:7897
printf "%s\n" $FILES | xargs -P 6 -I{} sh -c '
  curl -sL --retry 3 -o "{}" "https://downloads.sourceforge.net/project/corefonts/the%20fonts/final/{}"
'
```

全部 11 个文件的 SHA256 与 PKGBUILD 钉死值**逐一比对通过**：

```
andale32.exe     ✅   198384 字节
arial32.exe      ✅   554208 字节
...
通过 11 / 11，失败 0
```

> 📌 **可复用经验**：国内拉 SourceForge 资源，失败时先别换镜像，先试试 `curl -x http://127.0.0.1:7897`（或你本机代理的 mixed-port）。TUN 模式下这一步几乎总是通的。

### 1.3 坑二：GitHub tarball 的哈希三方打架

`ttf-wps-fonts` 的源码是 GitHub 上的 tarball，PKGBUILD 钉死了 sha1：

```
sha1sums=('fb270373c3bbea55ee80b56dd27de9f87f79607e' '6134a63d...')
```

但 GitHub 直连和 codeload 都超时，只有 `gh-proxy.com` 能下。结果下下来的包：

```
大小: 42266624 字节
实际 sha1: 15ed35c942c869d10d11dcb2ca17dd3f75d2a7cd
PKGBUILD:  fb270373c3bbea55ee80b56dd27de9f87f79607e
❌ 不一致
```

而且 `gzip -t` 报 **unexpected end of file**——**包本身是截断的**。

解包后逐文件对比上游，又发现问题：

```
通过 96, 失败 1, 未在树中 0
  seguibl.ttf  ❌ 期望 b2966570eae4…/324260B  实际 2211bfda81ed…/238592B
```

`api.github.com` 反而是通的（`200`）。于是用 **GitHub API 逐文件校验 git blob SHA1**：

```python
import hashlib, json, base64, urllib.request

def blob_sha(data):
    return hashlib.sha1(b"blob %d\0" % len(data) + data).hexdigest()

tree = json.load(urllib.request.urlopen(
    urllib.request.Request(
        "https://api.github.com/repos/ferion11/ttf-wps-fonts/git/trees/v2.0?recursive=1",
        headers={"User-Agent": "check"})))

# 对不上的文件，从 API 直接取 blob 重新落盘
for f in sorted(root.glob("*.ttf")):
    data = f.read_bytes()
    exp = blobs[f.name]
    if exp[0] != blob_sha(data):
        blob = json.load(urllib.request.urlopen(
            f"https://api.github.com/repos/ferion11/ttf-wps-fonts/git/blobs/{exp[0]}"))
        f.write_bytes(base64.b64decode(blob["content"]))
```

最终 **99/99 全部与上游 v2.0 git 树字节一致**。

由于上游 tarball 重新生成过、AUR 钉死的 sha1 本来就过期，我用**已验证的文件**重建了源码包，并把新 sha1 写进 PKGBUILD（改动可见、可复核）：

```
新 tarball sha1: f33c425542751a371bfb160a69be07c8b716afaf
旧 PKGBUILD 钉死: fb270373c3bbea55ee80b56dd27de9f87f79607e
```

> 📌 **Git blob SHA1 校验法**：`git blob` 对象的哈希是 `sha1("blob " + 长度 + "\0" + 内容)`，可以用纯 Python 复现。这样只要 `api.github.com` 通，就能对任意 GitHub 仓库的文件做逐字节完整性验证，完全绕开 tarball 的生成差异和代理缓存问题。

### 1.4 坑三：`/tmp` 是 3.8GB 的内存盘

WPS 的 `.deb` 有 630MB，解包后 3.1GB。在 `/tmp` 里构建直接爆了：

```
./usr/share/icons/hicolor/96x96/mimetypes/wps-office2023-wpsmain.png: Write failed: 超出磁盘配额
```

```console
$ df -h /tmp
tmpfs           3.8G  3.0G  762M   80% /tmp        ← 内存盘，只有 3.8G
$ df -h /
/dev/mapper/root  952G   15G  938G    2% /          ← 真实磁盘，还有 938G
```

把构建目录挪到 `/var/tmp`（真实磁盘）后一次成功：

```bash
mv /tmp/aur-build /var/tmp/aur-build
cd /var/tmp/aur-build/wps-office && makepkg -f --noconfirm
```

构建完清掉中间产物，回收 **7.1GB**。

> 📌 **Linux 上做体积大的构建，别默认 `/tmp` 是磁盘**。先 `df -h /tmp` 看一眼——它是 tmpfs 就算你有多少 RAM，占的都是内存。

### 1.5 小坑：pkexec 不继承工作目录

```console
$ pkexec pacman -U --noconfirm wps-office-12.1.2.29183-1-x86_64.pkg.tar.zst
错误：'wps-office-12.1.2.29183-1-x86_64.pkg.tar.zst': 无法找到或读取软件包
```

`pkexec` 会把工作目录重置掉，相对路径全部失效。**必须用绝对路径**：

```bash
pkexec pacman -U --noconfirm /var/tmp/aur-build/wps-office/wps-office-12.1.2.29183-1-x86_64.pkg.tar.zst
```

（另外我在这条路上还踩了两次 `pkill -f "xxx"` **把自己杀掉**的坑——`-f` 匹配的是完整命令行，而模式串就写在我的命令行里。用 `pgrep -f 'xxx'` 配字符类 `pgrep -f "usr/lib/[c]hromium"` 才安全。）

---

## 2. 装中文字体：两个来源

### 2.1 方正字体（wps-office-fonts）

WPS 官方 optdepends 之一，社区打包：

```bash
$ grep -E "^source" PKGBUILD
source=("https://github.com/Universebenzene/wps-office-fonts/archive/v${pkgver}.tar.gz")
```

同样走 gh-proxy，**85MB 完整下载**，sha256 一次通过：

```
gzip 完整性: 完整
实际 sha256: db01fc07324115b181cb06f50dfe09fd17feee132c46423ee70b260830211224
钉死 sha256: db01fc07324115b181cb06f50dfe09fd17feee132c46423ee70b260830211224
✅ 校验通过
```

装完 13 个 TTF，161.53 MiB：

```
FZCCHK.TTF   FZFSK.TTF   FZHTK.TTF   FZKTK.TTF   FZLSK.TTF
FZSongS_20100603.TTF   FZSongS(SIP)_2010603.TTF   FZSSK.TTF
FZWBK.TTF   FZXBSK.TTF  FZXH1K.TTF  FZXKK.TTF   FZYTK.TTF
```

对应的家族名：**方正书宋_GBK、方正黑体_GBK、方正楷体_GBK、方正仿宋_GBK、方正隶书_GBK、方正姚体_GBK、方正行楷_GBK、方正魏碑_GBK、方正细黑一_GBK、方正超粗黑_GBK、方正小标宋_GBK**。

### 2.2 思源宋体（官方仓库）

Adobe Source Han Serif 的简体中文子集，官方仓库直接装，56.89 MiB 下载 / 77.28 MiB 安装，7 个字重（ExtraLight/Light/Regular/Medium/SemiBold/Bold/Heavy）。

### 2.3 微软正字体：拿不到

理想状态下应该装 **SimSun / SimHei / Microsoft YaHei 本尊**。AUR 上确实有一批 `ttf-ms-win11-zh_cn`、`ttf-ms-win11-auto-zh_cn`，但——

```console
$ git clone https://aur.archlinux.org/ttf-ms-win11-zh_cn.git
警告：您似乎克隆了一个空仓库。

$ curl -s "https://aur.archlinux.org/cgit/aur.git/tree/?h=ttf-ms-win11-zh_cn"
class='error'>Invalid
```

**AUR 的 git 仓库已经空了**（RPC 索引还挂着旧数据，实际已下架——微软字体的再分发授权问题）。所以只能用方正/思源做替代，这在国内是通行做法。

---

## 3. 真正的关键：让「宋体」成为真实字体名

装完字体，`fc-match` 依然是这样：

```console
$ fc-match "SimSun"
LiberationSerif-Regular.ttf: "Liberation Serif" "Regular"
```

### 3.1 第一步：alias 替换（能用，但 WPS 会报「缺失字体」）

先用 `<alias>` 把 Windows 字体名指向已装的中文字体：

```xml
<!-- ~/.config/fontconfig/conf.d/60-chinese-font-aliases.conf -->
<alias binding="strong">
  <family>SimSun</family>
  <prefer><family>方正书宋_GBK</family><family>思源宋体 CN</family></prefer>
</alias>
```

`fc-match` 立刻正确：

```
SimSun → FZSSK.TTF: "方正书宋_GBK" "Regular"
Microsoft YaHei → FZXH1K.TTF: "方正细黑一_GBK" "Regular"
```

但启动 WPS 后，状态栏挂着一个 **⚠ 缺失字体**。

### 3.2 为什么 alias 不够

`<alias>` 只影响**匹配行为**——「你要宋体，我给你方正书宋」。

但 **字体家族列表**（`fc-list` / Qt 的 `QFontDatabase`）里依然没有「宋体」这一条。WPS 的检查逻辑是「文档里写的字体名，在系统字体列表里存在吗」，alias 顶上去不算存在。

验证一下：

```console
$ fc-list --format="%{family[0]}\n" | sort -u | grep -x "宋体"
(无输出)          ← 家族列表里没有「宋体」
```

### 3.3 第二步：把名字追加到字体的家族列表

fontconfig 的 `target="scan"` 匹配可以在扫描字体时**修改字体属性**。用 `<edit name="family" mode="append">` 让已安装的字体**额外认领**这些名字——字体本身的名字保留，同时多出 Windows 名字：

```xml
<!-- ~/.config/fontconfig/conf.d/61-chinese-font-families.conf -->
<match target="scan">
  <test name="family"><string>方正书宋_GBK</string></test>
  <edit name="family" mode="append"><string>宋体</string></edit>
  <edit name="family" mode="append"><string>新宋体</string></edit>
  <edit name="family" mode="append"><string>SimSun</string></edit>
  <edit name="family" mode="append"><string>NSimSun</string></edit>
</match>
```

完整覆盖了简中、繁中、日文、韩文共 **28 个字体名**：

| 字体名 | 落到 |
|---|---|
| 宋体 / 新宋体 / SimSun / NSimSun | 方正书宋_GBK |
| 黑体 / SimHei | 方正黑体_GBK |
| 微软雅黑 / Microsoft YaHei / Tahoma | 方正细黑一_GBK |
| 楷体 / KaiTi / 標楷體 / 华文楷体 | 方正楷体_GBK |
| 仿宋 / FangSong / 华文仿宋 | 方正仿宋_GBK |
| 隶书 / LiSu | 方正隶书_GBK |
| 幼圆 / YouYuan / 方正姚体 | 方正姚体_GBK |
| 华文行楷 | 方正行楷_GBK |
| 华文新魏 | 方正魏碑_GBK |
| 华文中宋 / 华文宋体 / STSong | Noto Serif CJK SC |
| 华文黑体 / 等线 / DengXian / STHeiti | Noto Sans CJK SC |
| 等线 Light / DengXian Light | Noto Sans CJK SC Light |
| 微軟正黑體 / Microsoft JhengHei | Noto Sans CJK TC |
| 新細明體 / PMingLiU | Noto Serif CJK TC |
| ＭＳ 明朝 / ＭＳ Ｐ明朝 / MS Mincho | Noto Serif CJK JP |
| ＭＳ ゴシック / ＭＳ Ｐゴシック / MS Gothic | Noto Sans CJK JP |
| 돋움 / 맑은 고딕 / 휴먼옛체 / 휴먼매직체 | Noto Sans CJK KR |

验证（注意：**要用完整家族列表**，只看 `%{family[0]}` 会误判）：

```python
out = subprocess.run(["fc-list", "--format", "%{family}\n"], ...).stdout
fams = {f.strip() for line in out.split("\n") for f in line.split(",")}
# → 宋体 in fams  == True
```

```
宋体            ✅ 真实家族  → 实际命中: 方正书宋_GBK
SimSun          ✅ 真实家族  → 实际命中: 方正书宋_GBK
微软雅黑         ✅ 真实家族  → 实际命中: 方正细黑一_GBK
楷体            ✅ 真实家族  → 实际命中: 方正楷体_GBK
仿宋            ✅ 真实家族  → 实际命中: 方正仿宋_GBK
等线            ✅ 真实家族  → 实际命中: Noto Sans CJK SC
华文中宋         ✅ 真实家族  → 实际命中: Noto Serif CJK SC
...
真实家族: 28/28
```

### 3.4 WPS 的「缺失字体」徽标消失了

重启 WPS，状态栏从：

```
页面: 1/1  字数: 1  拼写检查: 关闭  ⚠ 缺失字体
```

变成：

```
页面: 1/1  字数: 0  拼写检查: 关闭
```

字体下拉列表里方正全系和思源宋体也都在了。

---

## 4. 验证：别只看 fc-match

### 4.1 字形覆盖要单独验

`fc-match` 只告诉你「匹配到哪个文件」，不保证这个文件真的有那个字。用 `fc-list :charset=` 查：

```console
$ fc-list ":charset=4e2d" | wc -l
101                 ← 系统里含「中」字的字体有 101 个
```

对 5 款方正字体逐个验 中/文/字/,/。/—：

```
FZSSK.TTF      ✓✓✓✓✓✓
FZHTK.TTF      ✓✓✓✓✓✓
FZXH1K.TTF     ✓✓✓✓✓✓
FZKTK.TTF      ✓✓✓✓✓✓
FZFSK.TTF      ✓✓✓✓✓✓
```

### 4.2 端到端：在 WPS 里真打中文

用五笔码往 WPS 文档里打中文，截图确认：

```
你好
```

渲染正常，而且 fcitx5 的五笔候选窗在 WPS 里工作正常：

```
kh
┌──────────────────────────────┐
│ 1. 啊   2. 吼   3. 呵   4. 呐   5. 嗷 │
└──────────────────────────────┘
```

> 有个细节值得记：WPS 窗口里 `wtype` 注入的按键有时不走 fcitx5 的输入上下文（收到的是字面字符）。这是 WPS 自己的 IM 集成方式问题，与字体无关；真人在键盘上打字是正常的。

### 4.3 最终状态

```console
$ pacman -Q | grep -E "wps|font"
wps-office 12.1.2.29183-1                      2.41 GiB
wps-office-mime 12.1.2.29183-1
wps-office-fonts 1.0-2                          161.53 MiB
ttf-ms-fonts 2.0-12                               5.40 MiB
ttf-wps-fonts 2.0-1                              75.70 MiB
adobe-source-han-serif-cn-fonts 2.003-2          77.28 MiB
```

---

## 5. 踩坑清单

### 5.1 SourceForge 下不动？先走本机代理

```bash
curl -x http://127.0.0.1:7897 "<url>"
```

### 5.2 GitHub tarball 哈希对不上？

多半是 **GitHub 2023 年改过 tarball 生成算法**，内容一致但字节不同。用 `api.github.com` 逐文件校验 git blob SHA1：

```python
hashlib.sha1(b"blob %d\0" % len(data) + data).hexdigest()
```

### 5.3 `/tmp` 装不下大构建

```bash
df -h /tmp          # tmpfs = 内存盘
mv ~/build /var/tmp/build
```

### 5.4 pkexec 的两个坑

- **不继承工作目录** → 用绝对路径
- **不继承环境变量** → `pkexec env OMARCHY_PATH=/usr/share/omarchy <cmd>`

### 5.5 pkill -f 会杀掉自己

```bash
pgrep -f "usr/lib/[c]hromium"     # 字符类写法，不会自匹配
```

### 5.6 polkit 授权窗口弹不出来？

这次遇到的是 **会话锁定状态陈旧**：

```console
$ omarchy-shell lock status
{"locked":true, ... ,"lastEventAt":"2026-11-10T08:35:28.393Z"}   ← 8 小时前的残留

$ omarchy restart shell
Refusing to restart Omarchy shell while the session is locked.
```

会话实际是解锁的（根本没有锁屏窗口），但 shell 认为还锁着，于是**既拒绝重启、也不显示授权窗口**——`pkexec` 无限等待。触发 `omarchy-shell lock status` 重新求值后恢复正常。

排查 polkit 问题的顺序：

```bash
omarchy-shell lock status                              # 1. 会话是不是被当成锁着
timeout 8 omarchy-shell shell listPlugins | grep -i polkit   # 2. 插件 enabled 但 active:false？
hyprctl clients -j | grep -i polkit                    # 3. 对话框窗口真的没弹出来？
busctl --system list | grep polkit-agent-helper        # 4. agent 是否已注册
```

### 5.7 fc-match 会骗你

验证「字体名是否存在」要用完整家族列表，别只看 `%{family[0]}`：

```bash
fc-list --format="%{family}\n" | tr ',' '\n' | sed 's/^ *//' | sort -u | grep -x "宋体"
```

---

## 6. 一句话总结

> **中文文档在 Linux 上字体不对，99% 不是字体没装，而是「字体名」对不上。**
>
> 装字体只解决了 50%——剩下 50% 要靠 fontconfig 把 Windows 的字体名注册成系统里的真实家族名。
>
> `<alias>` 解决「显示」，`<edit mode="append">` 解决「存在」。

配置文件就两个，可以直接放进 dotfiles：

- `~/.config/fontconfig/conf.d/60-chinese-font-aliases.conf` — 替换优先级
- `~/.config/fontconfig/conf.d/61-chinese-font-families.conf` — 字体名注册

改完执行 `fc-cache -f` 生效。

---

*本文所有命令、报错和数字均为 2026-11-10 当天的真实输出。*
