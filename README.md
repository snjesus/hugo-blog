# BD4TWO 的博客

BD4TWO 的个人博客源码，Hugo 构建的静态站，部署在 Cloudflare Pages，域名是 blog.bd4two.eu.org。

## 本地构建

需要 Hugo（本站用 Hugo v0.169.0 构建通过）：

```sh
hugo
```

构建产物在 `public/`。

## 部署（Cloudflare Pages）

- 构建命令：`hugo`
- 输出目录：`public`
- 环境变量：`HUGO_VERSION = 0.169.0`

## 目录

- `content/`：文章和页面
- `layouts/`、`static/`：自写模板和样式，不依赖外部主题
