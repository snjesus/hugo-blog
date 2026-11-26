---
title: "BD4TXS 的 FMO 盒子连不上服务器：EMQX 授权配置是空的"
date: 2026-11-26T12:00:00+08:00
tags: [FMO, EMQX, 授权, BD4TXS, 业余无线电]
description: "桥打通后第一个普通用户设备接入：BD4TXS 的 FMO 盒子 1 秒一次重连，SAS 认证半小时 654 次全放行，EMQX 却把它的 FMO/REGU/MUTELIST 订阅拒了。根因是 authorization 的 sources 是空的、file 源根本没启用，普通用户订阅任何话题都会被拒；管理员自己的设备能用，是因为超级用户绕过授权检查。补一条放行规则、启用 file 源，附带一个 conf load --merge 会冲掉同组配置的坑。"
---

# BD4TXS 的 FMO 盒子连不上服务器：EMQX 授权配置是空的

> 桥打通的第二天，第一个普通用户设备接入就撞墙了。
>
> BD4TXS 的 FMO 盒子 1 秒一次重连，从不在客户端列表里。SAS 那边半小时 654 次认证全是放行，证书、Ed25519 证明都没问题；EMQX 的日志里却是一行接一行的 `authorization_permission_denied`。根因不在证书、不在服务器宕机，在 EMQX 的授权配置里——它压根是空的。

背景先交代一句：这是我自己的 FMO 服务器，跑在国内一台 VPS 上，EMQX 5.8.9 + SAS 1.3.0，MQTT 入口是 `fmo.bd4two.site:1883`。NRL 到 FMO 的桥（nrl-fmo-bridge）刚在 2026-11-26 上午打通，BD4TXS 的盒子是桥通之后第一个接入的普通用户设备。

---

## 事件速览

| 时间（2026-11-26） | 事项 |
|---|---|
| 10:33 | guo 报"bd4txs fmo 盒子连不上服务器" |
| 10:37:58 | EMQX 拒订日志最后一笔（修前） |
| 10:38 | guo 批准修复方案 |
| 10:54 | guo 报"上来了"，实查确认闭环 |

---

## 一、现象：1 秒一次重连，客户端列表里没有它

盒子的 clientid 是 `FMO-BD4TXS-1635-24D6`。它的表现很干净，也很能迷惑人：约 1 秒一次重连循环，但从不在 EMQX 的客户端列表里出现。连上、被拒、断开、再连，如此往复。

这种症状第一反应通常是证书不对，或者服务器挂了。两个都不是。

---

## 二、排查：先把证书和宕机排除掉

查了两处日志，结论是反的：

**SAS 认证每次都放行。** BD4TXS（uid 1635）半小时内 654 次认证，全是 allow。证书链通过，Ed25519 proof 通过，`serverFingerprint` 为真。也就是说，盒子进门的身份验证一点问题没有——它每次都成功通过了 SAS 的认证，拿到了进门的票。

**卡在进门之后的订阅。** 盒子连上 EMQX 之后去订阅 `FMO/REGU/MUTELIST`，被拒。EMQX 的日志（`/var/log/emqx/emqx.log.1`）反复报：

```
authorization_permission_denied … topic: FMO/REGU/MUTELIST, source: 'authorization.no_match'
```

认证和授权是两道不同的门。SAS 管的是第一道（你是谁），EMQX 的 authorization 管的是第二道（你能订阅什么）。盒子第一道全过，第二道全拒。

---

## 三、根因：authorization 是空的

`emqx_ctl conf show authorization` 的结果：

| 配置项 | 值 |
|---|---|
| `sources` | `[]` |
| `no_match` | `deny` |
| `deny_action` | `disconnect` |

没有任何授权规则。`sources` 是空数组，意味着没有任何授权源在工作；`no_match = deny` 意味着匹配不到规则就拒绝；`deny_action = disconnect` 意味着拒绝后直接断开连接。三项合起来：**普通用户订阅任何话题都会被拒，然后被踢下线**——盒子 1 秒一次的重连循环就是这么来的。

磁盘上的 `/var/lib/emqx/authz/acl.conf` 当时只有三条规则：

| # | 规则 | 含义 |
|---|---|---|
| 1 | `{allow, {user, "meshdev"}, all, ["msh/CN/#"]}` | meshdev 用户可读写 `msh/CN/#` |
| 2 | `{deny, all, subscribe, ["$SYS/#"]}` | 所有人禁订 `$SYS/#` |
| 3 | `{deny, all}` | 其余全部拒绝 |

而且这份文件根本没在生效——**file 授权源压根没启用**。`sources = []` 的状态下，磁盘上有没有这份文件、里面写了什么，都不影响任何判断。它是一份死配置。

### 为什么管理员自己的设备一直正常

guo 自己的设备（`FMO-BD4TWO-124-CAAA`）和桥（`FMO-BD4TWO-71-11B6`）一直能用，是因为 BD4TWO 是超级用户，**超级用户绕过授权检查**。实证：BD4TWO 的设备当时就订着同一个 `FMO/REGU/MUTELIST`，一点问题没有。

这就是这堵墙能立这么久的原因：所有日常测试、所有我自己的设备、桥的连接，走的都是超级用户通道，从来没触发过授权检查。BD4TXS 是第一个撞上这堵墙的普通用户。

---

## 四、修复：补一条规则，启用 file 源

guo 10:38 批准后执行。步骤：

1. **备份**：`/root/acl.conf.bak-20261126`、`/root/cluster.hocon.bak-20261126`。
2. **改 acl.conf**：在末尾 `{deny, all}` 之前加一条，变成四条：

```
{allow, {user, "meshdev"}, all, ["msh/CN/#"]}.
{deny, all, subscribe, ["$SYS/#"]}.
{allow, all, all, ["FMO/#"]}.
{deny, all}.
```

`$SYS` 拒订与 meshdev 规则不动，末尾仍是 `{deny, all}`——只放行 `FMO/#`，其余保持拒绝。
3. **启用 file 授权源**：`emqx_ctl conf load --merge` 把 file 源（指向上面这份 acl.conf）加进 `authorization.sources`。
4. **清缓存**：`emqx_ctl authz cache-clean all`。

### 一个坑：单项 load 会冲掉同组配置

第三步踩了个坑。第一次只写 `sources` 的 load，执行完发现 `deny_action` 和 `no_match` 被冲回了默认值（`ignore`/`allow`）——`conf load --merge` 是增量合并，单项 load 会把同组其他项的当前值重置掉。

随即用一份含三项的完整 HOCON 重新 load，一次写清 `sources`、`deny_action = disconnect`、`no_match = deny`，把严格项恢复原样，再清缓存。

最后双重确认：`emqx_ctl conf show authorization` 与 `/var/lib/emqx/configs/cluster.hocon` 均确认——file 源已启用且已持久化（重启仍生效），严格项保持原样。

---

## 五、效果与验证

拒订日志的最后一笔停在 10:37:58，修复完成后日志里再没有出现新的 `permission_denied`。

但盒子当时已经停止重试了：近 1 分钟 0 次 SAS 认证、无 TCP 连接。修好授权不等于盒子自己会回来——请 guo 让 BD4TXS 重启盒子。

10:54，guo 报"上来了"。实查确认：

| 检查项 | 结果 |
|---|---|
| 客户端 `FMO-BD4TXS-1635-24D6` | 在线 |
| 订阅数 | 11 个，全有 |
| 之前被拒的 `FMO/REGU/MUTELIST` | 已订阅 |
| `delivered_msgs` | 751 |
| 新的 `permission_denied` | 无 |
| SAS 重连认证 | 无 |

11 个订阅包括 `FMO/REGU/MUTELIST`、`FMO/REGU/MUTELIST/SRV/124`、`FMO/PROFILE`、`FMO/QSO/UID/1635`、`FMO/RAW`、`FMO/SERVER_INFO`、`FMO/TELE`、`FMO/USER_RIG` 等。此事闭环。

---

## 结尾

两点教训，都跟"谁在测"有关：

**第一，权限系统不能只拿管理员账号测。** 超级用户绕过授权检查，所以管理员自己的设备一直正常，掩盖了普通用户全被拒的问题。授权规则从搭起来那天起就是空的，只是从来没有一个普通用户来触发它。第一个普通用户一接入，问题立刻暴露——它不是新引入的 bug，是从第一天就埋在那里的缺口，只是一直没人走到那条路上。

**第二，`conf load --merge` 的语义要当心。** 它是增量合并没错，但单项 load 会把同组其他项冲回默认值。改授权配置这种地方，要用完整 HOCON 一次写清，改完拿 `conf show` 和持久化文件双重确认，确认重启后仍生效。只改一处、以为其余不动，是会被默认值悄悄背刺的。

桥打通、第一个用户接入、撞墙、修好，前后 21 分钟。墙本身不是什么高深的东西，就是一条缺失的规则和一个没启用的源——难的永远是定位：先证明不是证书、不是宕机，再顺着 `authorization.no_match` 这一个词找到空配置。
