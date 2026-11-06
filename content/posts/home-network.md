---
title: "家里网络：爱快、Lucky 与隧道"
date: 2026-11-06
---

家里宽带有公网 IP，但运营商封了 80 和 443 入站。想不带端口访问内网服务，路子是两条：内网用 Lucky 反向代理，外网真免端口得走 Cloudflare Tunnel。

爱快网关是 192.168.31.1，docker 网段是 172.18.20.x，Lucky 容器是 172.18.20.2，docker 网关 172.18.20.1 已确认就在爱快本机上。Lucky 是以 Docker 方式跑在爱快路由器上的，实测版本 2.20.2 自带 Cloudflared 模块，不用另起 cloudflared 容器，直接在 Lucky 后台（http://爱快IP:16601）的 Cloudflared（或"隧道"）模块里配。隧道在 Cloudflare 控制台建好，名叫 home，把令牌填进 Lucky 的隧道模块——第一次填错了，填的是隧道 ID（UUID）而不是 eyJh 开头的真正令牌，换对之后就通了，guo 亲口确认"通了"，Cloudflare 侧 1 个连接器状态正常。随后给 NAS 加了公网路由：nas.bd4two.eu.org → https://192.168.31.5（服务类型 HTTPS），NAS 内网地址是 https://192.168.31.5（飞牛 fnOS，HTTPS，默认 443 端口）。一开始报 502，原因是 NAS 的自签证书过不了 Cloudflare 的 TLS 校验，开了 No TLS Verify 就好了，2026-10-16 公网实测能打开 NAS 登录页。Lucky 管理后台只监听 HTTP（16601 端口，没有 HTTPS），加路由时服务类型必须选 HTTP、URL 填 http://127.0.0.1:16601，公网侧仍走 HTTPS（Cloudflare 自动套证书），只是隧道到家内网这一段走 HTTP。

后来两条隧道都删了。2026-10-28，guo 把家里 home 隧道和 VPS 的 argo 隧道都删了，起因是 10 月 27 日 Cloudflare 发来的 cloudflared 升级邮件（要求升到 2026.10.2 以上，eMudhra 弱证书原因）。删了隧道升级事项也就随之关闭，经隧道走的公网路由（nas.bd4two.eu.org 等）随之失效，内网访问不受影响。

还有一件没解决的：容器到爱快本机的流量被整段静默丢弃。实测容器里 ping 172.18.20.1 和 192.168.31.1 均 100% 丢包，wget 连 172.18.20.1:80 超时（不是拒绝）。而 cloudflared 出网、Lucky 反代 NAS 都正常，拦截点在爱快本机 INPUT 侧，管转发的 ACL 大概率管不着。下一步是补测容器内 `wget -T5 -O /dev/null http://192.168.31.1/` 和 `cat /proc/net/arp`，再决定要不要在爱快本机加 iptables INPUT 放行规则。教训是：容器访问爱快本机走网关地址，局域网其他设备访问容器服务必须做端口映射，容器 IP 只有爱快本机能直接访问。

另外提醒自己一句：Lucky 默认账密是 666/666，挂公网前必须改掉，或者加一层 Cloudflare Access 验证。
