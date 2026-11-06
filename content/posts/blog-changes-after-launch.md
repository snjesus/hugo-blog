---
title: "博客上线后的改动记录"
date: 2026-11-06
---

博客上线之后又做了几处改动：换 PaperMod 主题、关于页加图片、标题栏图标换成头像图、关于页图片改小并挪到呼号前面。这里把每一步的原样记录下来，包括涉及的地址和细节。

## 换 PaperMod 主题

上线时用的是自己写的极简模板，后来觉得功能太少，换成了 PaperMod。PaperMod 是 GitHub 上约 1.3 万星的开源主题，MIT 协议，零 JS 构建工具，自带搜索、深浅色切换、目录、面包屑、封面图、多语言支持，不用 Node 和 Ruby，Cloudflare Pages 自带的 Hugo 就能编。

安装没走常规的子模块或 Hugo Modules 路线，而是把主题文件直接拷进仓库根目录的 `layouts/`、`assets/`、`i18n/`，一共 121 个文件。旧的 4 个自定义模板（`layouts/_default/baseof.html`、`list.html`、`single.html`、`layouts/index.html`）被 PaperMod 的对应内容覆盖，`static/css/style.css` 改成一行注释弃用。PaperMod 要求 Hugo 不低于 0.158.0，当前用的 0.169.0 符合。

`hugo.toml` 按 PaperMod 的要求改了一遍：`locale = "zh-cn"`、`defaultContentLanguage = "zh-cn"`（`languageCode` 在 Hugo 0.158 起已废弃，改用 `locale`）、`mainSections = ["posts"]`（PaperMod 首页靠它取文章列表）、菜单加了首页、归档、搜索、关于，`[outputs] home` 加了 `JSON`（给搜索索引用），params 里阅读时间、分享按钮、上下篇导航、面包屑、代码复制按钮、目录都打开，`defaultTheme = "auto"`。另外新建了 `content/archives.md` 和 `content/search.md`。

推送提交是 `dbd66fcd48458e363ee461c52e2fd595b6476af3`。本地构建通过，Pages 部署后复核：首页 200 且含 PaperMod 标记，搜索索引 `index.json` 200，归档和搜索页 200，文章页有 `post-single` 类，中文界面串（分钟、归档、搜索）正常，`html lang="zh-cn"`。

仓库地址是 `github.com/snjesus/hugo-blog`，main 分支。部署地址是 `hugo-blog-5h6.pages.dev`，自定义域名是 `blog.bd4two.eu.org`。

## 关于页加图片

关于页原来只有两段文字，我把一张黑白鱼眼照片放了进去。照片是 1024×1024 的 JPEG，约 98KB，内容是走廊里两个人并立，其中一人手里拿着一瓶酒。

图片进仓库遇到一个实在的障碍：GitHub 的 `push-files` 快照推送只接受 UTF-8 文本文件，二进制推不上去；`create_or_update_file` 的 content 是 JSON 字符串，也塞不进原始二进制。所以图没进仓库文件，而是以 data URI 的形式嵌进 `content/about.md`。做法是先把图缩到 400×400（约 32KB），base64 编码约 43KB，写进 about.md，用 PaperMod 的 `figure` 短代码引用，显示宽 400、居中。

这里又踩了一个坑：PaperMod 的 `figure` 短代码原样输出 data URI，会被 Go 模板净化成 `#ZgotmplZ`，图片出不来。原因是 Go 的 `html/template` 认为 `data:` 链接不安全，直接给替换了。我把仓库里 `layouts/_shortcodes/figure.html` 的 `src` 改成过 `safeURL` 才放行。另外 base64 里的 `+` 会被转义成 `&#43;`，浏览器解析属性值时会还原，不影响显示；验证过解码后与原图逐字节一致，是合法 JPEG（以 `FFD8` 开头、`FFD9` 结尾）。

推送提交是 `4106eb6d715cfc77b38f8d468af8ad5fcad1901e`。部署后复核：关于页 200，图在、宽 400、居中。

## 标题栏图标换成头像图

浏览器标题栏原来是个默认的球标，我把它换成了上面那张图。favicon 不宜太大，我把图缩到 32×32（另有一份 16×16），以 data URI 写进 `hugo.toml` 的 `[params.assets]`，四项都填了：`favicon`、`favicon16x16`、`favicon32x32`、`apple_touch_icon`。32×32 的 data URI 只有约 1.5KB，每页 head 里带一份可以接受。

同样的问题又出现一次：PaperMod 的 `head.html` 里五个 favicon 链接用的是 `absURL`，data URI 照样被净化成 `#ZgotmplZ`。我把 `layouts/_partials/head.html` 里五个 favicon 链接都改成 `absURL` 后再过 `safeURL` 才放行。

## 关于页图片改小并挪到呼号前面

400 宽的图在关于页还是显得大，我把它改到 250 宽，图数据也重缩到 250×250（约 19KB base64），位置从文末挪到第一段之后、“呼号 BD4TWO”那段之前，仍居中。关于页现在是：第一段“这里是 BD4TWO 的个人博客。”，然后是图，再是“呼号 BD4TWO。博客主要写无线电、技术和生活方面的内容，想到什么写什么。”

这次改动连同 favicon 一起推送，提交是 `832445b66de0b33db76d11f467282c44297d2028`。部署后复核：首页 favicon 是 data URI（32×32 合法 JPEG，无 `#ZgotmplZ`），关于页宽 250、图在呼号前面。

一点提醒：浏览器会缓存旧 favicon，如果标题栏还是原来的球标，强制刷新一次（Ctrl+Shift+R）就能看到新的。
