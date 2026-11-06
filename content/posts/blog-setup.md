---
title: "博客建站记录"
date: 2026-11-06
---

今天把这个博客搭起来了，记录一下过程，免得以后忘了。

## 目标

一个个人博客，域名用自己的 `blog.bd4two.eu.org`，以后写无线电、技术和生活方面的东西。要求就一个：免费、稳定、自己能控制。

## 第一次尝试：GoogieHost，没成

一开始想走 GoogieHost 的免费主机，注册、选域名、下单都走完了，订单号都拿到了，结果被拒。拒的原因写得很直白：订单国家在中国，而中国在他们的禁用名单上。免费主机这条路就这么断了。

## 改走 Cloudflare Pages + Hugo

免费主机被拒后，改成静态博客的路线：

- **Hugo**：静态站点生成器，写 Markdown 就能出站，没有数据库、没有后台，简单可靠。
- **GitHub**：源码放在 `snjesus/hugo-blog`，推到 main 分支即是发布。
- **Cloudflare Pages**：接 GitHub 仓库，推代码自动构建部署，免费额度个人博客完全够用。
- **域名**：`blog.bd4two.eu.org`，在 Cloudflare 上做 CNAME 指向 Pages，SSL 自动配好。

中间有个小插曲：GitHub 上本来已经有个 `snjesus/blog` 的库，里面是另一套完整的博客系统（Qingyu'Blog），不是空库。所以 Hugo 这套单独建了个新库 `hugo-blog`，没去动原来的那个。

## 搭建步骤

1. 本地用 Hugo 建站，标题定为“BD4TWO 的博客”，写了篇欢迎帖占位。
2. GitHub 上新建公开空仓库 `snjesus/hugo-blog`，把源码推上去。
3. Cloudflare 上新建 Pages 项目 `hugo-blog`，连接这个仓库，生产分支 main。
4. 构建设置：框架 Hugo，构建命令 `hugo`，输出目录 `public`，环境变量 `HUGO_VERSION=0.169.0`。
5. 首次部署成功后，给项目加自定义域名 `blog.bd4two.eu.org`，Cloudflare 自动加了 CNAME 记录并启用 SSL。
6. 打开 `https://blog.bd4two.eu.org/` 验证，首页正常，标题、简介、文章列表都在。

## 写文章的方法

以后写新文章很简单：在仓库的 `content/posts/` 里新建一个 Markdown 文件，文件头写标题和日期，正文写内容，推到 main 分支，Pages 会自动重新部署，几分钟后就能在博客上看到。

## 现状

博客已上线，地址是 [https://blog.bd4two.eu.org/](https://blog.bd4two.eu.org/)。全套免费，源码在 GitHub，部署在 Cloudflare Pages。先这样，内容慢慢写。
