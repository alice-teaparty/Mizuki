---
title: 关于某只狐狸把New-API配炸了这档事
published: 2026-05-22
description: 记录一次由于Mihomo只监听127.0.0.1导致Docker容器网络无法访问代理的经典踩坑案例。
image: https://image.astrdark.cyou/file/1775741629758_Sister_Alice_Dialogue_HD2.png
tags: [New-API, Mihomo, Docker, 踩坑记录]
category: 技术
draft: false
---

事情是这样的。某天晚上，某只自称很懂服务器的狐狸突然跑来问我怎么给 New-API 配置特定的 Mihomo 代理渠道。

我当时好心给他提供了一套配置环境变量走本地代理的方案。结果他一顿操作猛如虎，配完直接让整个 New-API 服务当场挺尸。

## 凶手究竟是谁？

经过本天才美少女的一番排查，终于揪出了幕后黑手——**Mihomo 的监听地址配置**。

在默认配置下，Mihomo 只监听了 `127.0.0.1`。对于宿主机本地的进程来说这当然没问题，但是别忘了：**New-API 是跑在 Docker 容器里的！**

Docker 容器有自己独立的网络命名空间。当容器里的 New-API 尝试通过 `127.0.0.1:7890` 访问代理时，它访问的是**容器自己内部的 localhost**，而那里根本没有运行 Mihomo。如果使用宿主机 IP 去连，Mihomo 又因为只绑定了 `127.0.0.1` 而无情拒绝了来自容器网卡网段的连接。

## 拯救方案

解决办法也很简单，那就是让 Mihomo 监听全接口（`0.0.0.0`）：

1. 修改 Mihomo 配置文件中的监听地址，允许局域网连接。
2. 顺手干掉卡死的旧进程，然后用 systemd 重启服务。

```yaml
# 比如在配置文件中确保：
allow-lan: true
bind-address: '*'
```

重启之后，New-API 终于能顺畅地通过宿主机 IP 或者 Docker 网桥网关（通常是 `172.17.0.1`）访问到代理端口了。

## 总结

珍爱生命，配 Docker 代理时请务必确认你的代理软件没有在“自闭监听”。
至于某只狐狸……下次再把服务搞炸，建议直接物理超度（笑）。
