---
title: Komari 面板迁到腾讯云 + 公网域名折腾笔记
date: 2026-10-29 09:00:00
tags:
---

家里几台机器（腾讯云、华为云、DediRock）想统一看一下状态，之前用 [Komari](https://github.com/komari-monitor/komari) 把面板挂在 DediRock 上，Agent 各机连过去。后来想把面板挪到腾讯云，并挂上自己的域名，顺便把「未登录也能看状态」这块收紧。记一下走过的路。

## 为啥要迁

DediRock 那台更适合当被监控的节点，控制台放腾讯云更顺手：域名、Nginx、证书本来就在那台机上。面板迁走之后，各机 Agent 只改上报地址，历史数据跟着 data 目录一起搬。

## 迁面板（Docker Compose）

腾讯云上用 Docker（snap 装的那套）拉起来，目录大致在 `/home/ubuntu/komari-panel/`，compose + 持久化 data 都在里面。

流程很直白：

1. DediRock 停掉旧面板，导出 compose 和 data。
2. 腾讯云建好同结构目录，导入后 `docker compose up -d`。
3. 各机 Agent 的 `-e`（面板地址）改成腾讯云的 Tailscale 地址，例如 `http://100.80.30.15:25774`（端口以自己 compose 为准）。
4. 确认节点都重新上线后，DediRock 旧面板保持停用，别两套一起抢。

Tailscale 在这几台之间当「内网」用，Agent 不走公网，少一层暴露。当时几台大概是：腾讯 `100.80.30.15`、DediRock `100.107.25.46`、华为 `100.75.188.83`。

## 公网：Nginx 按 Host 分流

腾讯云上本来就有 Nginx + Let's Encrypt，主站 `neoyao.site` 指 `/var/www/blog`，还有 `api`、`derp` 等子域。Komari 走同一套路：DNS 给 `komari.neoyao.site` 加 A 记录到公网 IP，再加一个 `server_name komari.neoyao.site` 的站点，反代到本机 `127.0.0.1:25774`。

分流靠的就是 HTTP `Host`：同一个 443，Nginx 看你访问的是哪个域名，再进对应的 `server` 块。证书用 certbot 给子域签一张（或扩到现有证书里），和博客那套一样维护就行。

弄完之后公网入口是：

https://komari.neoyao.site

（管理账号是自己设的那个，这里不写密码。）

## 别让路人直接看到状态

默认面板有时会把节点概况露给未登录访客。公网挂上之后，第一件事是在设置里打开 **Private Site**（或同等「禁止游客看状态」），需要的话再开 2FA。反代只解决「怎么访问」，不解决「谁能看」。

## 现在这套大概长这样

- 面板：腾讯云 Docker，只在本机监听，公网经 Nginx 反代  
- 节点：各机 Agent 经 Tailscale 回报  
- 域名：`komari.neoyao.site`，和博客、API 一样按 Host 分流  
- 安全：Private Site 开着，密码和 2FA 自己管  

以后要是再加机器，装 Agent、指到腾讯云 Tailscale 地址即可，不必再搬一次面板。
