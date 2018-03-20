---
layout: post
title: 将 TL-WR703N 改造成 Wi-Fi 探针设备 (保留无线路由功能)
key: 2018-03-21-wr703n-probing-device
---    
上一篇文章 [给 v1.7 版本的 TL-WR703N 刷 openwrt](/2018/03/17/wr703n-openwrt.html) 刷好了 OpenWrt (其实后面我又把固件换成了 LEDE, 不过都算是一个东西) 之后, 我们就可以在这个基础上进行进一步的折腾.     
本篇文章将会讲解如何将这台路由器改造成一个带探针功能的路由器. 所谓探针, 就是可以收集附近开启了 Wi-Fi 的设备的 MAC 地址和信号强度等信息的设备 (主要通过 probe 帧). 通过收集和分析这些数据, 我们可以得知附近的设备数量, 或者使用多台探针设备配合使用记录设备的轨迹.
<!--more-->
## Wi-Fi 探针简介
设备获取周围的 Wi-Fi 基站列表有两种方法: 第一种方式是通过解析 AP 发出的 Beacon frame 得知附近 AP 的存在 (被动方式), 另一种方式 (主动方式) 则是客户端主动发送 Probe request, AP 接收到 Probe request 帧后向客户端回复 Probe response, 从而完成 AP 的发现过程.

## 未完待续


loader.sh
```shell
#!/bin/sh
cd /tmp
wget -q <url of your init.sh>
chmod +x ./init.sh
./init.sh
```

init.sh
```shell
#!/bin/sh
cd /tmp
wget -q  http://<server addr>/libpcap.so
wget -q  http://<server addr>/libradiotap.so
wget -q  http://<server addr>/probe_frame_capture
chmod +x ./probe_frame_capture
export LD_LIBRARY_PATH=/tmp:$LD_LIBRARY_PATH
./probe_frame_capture mon0 127.0.0.1 8888 > /dev/null

```

```shell
 echo "dnsmasq:*:453:dnsmasq" >> /etc/group
 echo "dnsmasq:*:453:453:dnsmasq:/var:/bin/false" >> /etc/passwd
 echo "dnsmasq:*:0:0:99999:7:::" >> /etc/shadow
 chown dnsmasq.dnsmasq /usr/sbin/dnsmasq
 ```