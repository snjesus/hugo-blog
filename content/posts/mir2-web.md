---
title: "网页端《传奇》部署准备"
date: 2026-11-06
---

想在家里把网页端《传奇》跑起来，部署目标是爱快路由器的 Docker。选的方案是 github.com/zombieliu/mir2 的 mir2-web3，一个社区重制项目，不是盛大官方原版，这点先说清楚。

部署方式是 Docker，官方 dev.sh + compose.developer.yml，一条命令 `./scripts/dev.sh up --build` 拉起。默认端口都只绑 127.0.0.1：玩家页面 3002、网关 WebSocket/健康检查 7110（/ws、/health）、Crystal TCP 7000。局域网或外网访问要改网关地址配置（MIR2_BIND_ADDRESS 和 NEXT_PUBLIC_MIR2_GATEWAY_WS_URL，前端默认网关地址是 ws://127.0.0.1:7110/ws）。仓库自带的只是精简的 starter 素材，完整素材包约 18GB，是私有的，需要找作者授权才能拿到。首次运行还要在本地编译 Rust 代码，是开发环境的做法，比较吃内存和磁盘。

动手之前先摸清机器底子：系统架构、内存、磁盘余量这些。内存和磁盘得够，架构得对得上镜像，否则编到一半才发现跑不起来。第一步的环境核查命令是 `docker --version; docker compose version; git --version; uname -m; free -h; df -h /`，门槛是 x86_64、内存 4G 以上、磁盘 20G 以上。

现状：等机器配置输出，达标再开始部署。素材授权和完整包的获取方式也还没定。
