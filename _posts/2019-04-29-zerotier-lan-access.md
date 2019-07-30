---
layout: post
title: 使用 ZeroTier 三层路由实现内网访问
key: 2019-04-29-zerotier-lan-access

---
ZeroTier 配置内网访问 VPN 的步骤存档
<!--more-->

首先到 [ZeroTier 官网](https://www.zerotier.com/) 按照教程下载并安装 ZeroTier.

在 [my.zerotier.com](https://my.zerotier.com/) 创建网络, 认证客户端, 并且在 Advanced -> Managed Routes 配置路由:

![](/content\images\2019\zerotier-lan-access\route.png)

via 框内填写内网设备对应的地址

内网设备还需要其他一些额外配置：

首先，设置防火墙允许转发和 IP masquerade 

其中 `eth0` 是本地网络接口, `ztbtotfrd7` 是 ZeroTier 的网络接口名, 替换成实际名称

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o ztbtotfrd7 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i ztbtotfrd7 -o eth0 -j ACCEPT
```

另外在 `sysctl.conf` 加入 `net.ipv4.ip_forward=1` 允许转发, 或者直接使用下面的命令

```bash
sysctl -w net.ipv4.ip_forward=1
```

然后就可以通过 ZeroTier 访问内网中的其他机器了

### References

[ZeroTier Knowledgebase](https://zerotier.atlassian.net/wiki/spaces/SD/overview)

[Easy way of bridging lan for remote access](https://www.reddit.com/r/zerotier/comments/9714a2/easy_way_of_bridging_lan_for_remote_access)

