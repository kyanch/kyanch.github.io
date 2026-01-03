---
layout: post
title: Wireguard Setup
date: 2024-11-15 08:51 +0800
---

> 这B玩意这么简单，搞我好久X(

## 服务器配置

```
[Interface]
Address = 10.0.0.1/24
PrivateKey = <Server_PrivateKey>
ListenPort = 51820

[Peer]
PublicKey = <Peer_A_PublicKey>
AllowedIPs = 10.0.0.2/32
PresharedKey = <PresharedKey>
```

几点要注意的：
1. Address 后面的cidr表示网段.会变成这样一条route记录:
    `10.0.0.0/24 dev wg proto kernel scope link src 10.0.0.1`
2. Peer的AllowedIPs不要有重叠区域，否则无法正确转发，包会死循环。
3. 最佳实践是：
    1. Peer如果是客户端，AllowedIPs就ipv4/32或者ipv6/128
    2. Peer如果是服务端，AllowedIPs就包含转发的网段。多个网段不要有重叠
    3. wg本身并不区分客户端和服务端。这里说的的服务端和客户端的主要区别是能否转发包到其他ip或者网段

## 客户端配置

```
[Interface]
Address = 10.0.0.2/32
PrivateKey = <Peer_A_PrivateKey>

[Peer]
PublicKey = <Server_PublicKey>
AllowedIPs = 10.0.0.0/24, 10.0.1.0/24
Endpoint = ip/cidr
PresharedKey = <PresharedKey>
```
同样，几点注意：
1. wg并不支持动态ip，所以一个/32的ip是必要的。除非你想当服务端。
2. AllowedIPs指定的网段会在wg启动时添加到路由表。
    - 也就是说：发往这些网段的包会从这个peer转发。具体的处理由这个peer自己的服务决定（防火墙，ipv4.forward）
3. 知道Endpoint的peer才能主动发起连接，否则只能等别人来连接你。

如果你还有第二个客户端
```
[Interface]
Address = 10.0.0.3/32
PrivateKey = <Peer_B_PrivateKey>

[Peer]
PublicKey = <Server_PublicKey>
AllowedIPs = 10.0.0.0/24, 10.0.1.0/24
Endpoint = ip/cidr
PresharedKey = <PresharedKey>
```
这样，你应该可以直接从10.0.0.2直接ping通10.0.0.3（10.0.0.3需要开启ICMP）,你也可以在10.0.0.3上起一个简单的http服务来验证连接
当然，server需要开启ipv4.forward。


