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
在设备的 Wi-Fi 处于开启状态时, 设备将会不断广播 Probe 帧以探测周围的热点. 即使已经连接到了某个 AP, 设备仍会定期发送 probe 帧, 只是频率会稍有降低. 通过一台设置为监听模式的无线网卡设备, 我们可以捕捉到这些设备广播的 Probe 帧.     

## 编写监听程序
### 使用 libpcap 抓取无线网帧
谈到抓包, 就不得不提到大名鼎鼎的 libpcap. libpcap 是 [tcpdump](https://www.tcpdump.org/) 的底层库,被广泛应用在 *nix 系统下的抓包应用中, 在 Windows 下它的名字叫 WinPcap (wpcap). 它的功能主要有: 抓包, 构造原始数据包 (raw packets) 并发送, 流量统计和包过滤.      
libpcap 主要采用旁路方式抓包, 当数据包到达网卡时, libpcap 从链路层驱动中获取数据包的内容, 将其发给过滤器, 然后传递到用户缓冲区. 可以设置一个回调函数, 这样在每个数据包被抓取到后, 都会自动调用回调函数进行处理.     

用 libpcap 抓包, 主要分为以下几个步骤:    
1. 用 ```pcap_open_live()``` 函数打开网络接口设备
2. ```pcap_compile()``` 函数编译过滤规则
3. ```pcap_setfilter()``` 函数应用过滤规则到过滤器
4. 设置回调函数, 并开始监听数据包
5. 监听程序退出时, 记得关闭接口, 使用 ```pcap_close()```

下面的代码演示了 1-3 步:     
```

if ((adhandle = pcap_open_live(args.interface, 65536, 1, 1000, errbuf)) == NULL) {
    fprintf(stderr, "Unable to open the adapter %s: %s\n", args.interface, errbuf);
    return 1;
} // 打开设备

if (pcap_compile(adhandle, &filter, filter_string, 1, PCAP_NETMASK_UNKNOWN) == -1) {
    pcap_perror(adhandle, "pcap_compile error: ");
    return 1;
} // 编译规则

if (pcap_setfilter(adhandle, &filter) == -1) {
    pcap_perror(adhandle, "pcap_setfilter error: ");
    return 1;
} // 应用规则

```

### 使用 library-radiotap 解析 RadioTap 头
在利用 libpcap 抓取到我们所需要的数据报文之后, 接下来就是解析提取这些报文中所包含的信息. 我们所需要的信息有两个: 接收到的信号强度和发送方的 MAC 地址.     
RadioTap 头是 IEEE 802.11 网帧注入和接收事实上的标准, 被很多系统所支持. 它的长度和字段都是不固定的, 具体包含的字段可以在一个叫做 ```present``` 的字段中找到. 这个字段通常有 4 bytes (32 bits) 长, 根据不同比特位的数值不同, 规定了在接下来的数据区域会有哪些字段的值出现. 具体的定义可以在[这里](https://www.radiotap.org/) 查到. 
在 C 语言中, 它的结构体定义如下:
```
struct ieee80211_radiotap_header {
        u_int8_t        it_version;     /* set to 0 */
        u_int8_t        it_pad;
        u_int16_t       it_len;         /* entire length */
        u_int32_t       it_present;     /* fields present */
} __attribute__((__packed__));
```
RadioTap 头的解析器已经有现成的轮子可以使用, 叫做 [radiotap-library](https://github.com/radiotap/radiotap-library)      
在监听程序中, 我们需要的字段只有 ```RSSI```    
代码如下:
```
// definition of struct frame_info
typedef struct frame_info {
    signed char ssi_signal_dBm;
    time_t timestamp;
    unsigned char src_mac[6];
    unsigned char dst_mac[6];
} frame_info;

// parsing function
int parse_frame(const u_char *data, size_t len, struct frame_info *f) {
    struct ieee80211_radiotap_iterator iter;
    int err = 0;
    uint16_t radiotap_len = (data[0x3] << 8) | data[0x2];
    memcpy(f->src_mac, data + radiotap_len + 0xa, 6);
    memcpy(f->dst_mac, data + radiotap_len + 0x4, 6);

    err = ieee80211_radiotap_iterator_init(&iter, (struct ieee80211_radiotap_header *) data, len, NULL);
    if (err) {
        return 0;
    }
    while (!(err = ieee80211_radiotap_iterator_next(&iter))) {
        if (iter.is_radiotap_ns) {
            if (iter.this_arg_index == IEEE80211_RADIOTAP_DBM_ANTSIGNAL && !f->ssi_signal_dBm) {
                f->ssi_signal_dBm = *(signed char *) iter.this_arg;
            }
        }

    }

    return 1;
}
```
### 组合
最终收集到数据后, 我们可以通过 UDP 协议将数据包发回服务器, 可以将其输出到本地文件中, 同时我们也希望程序可以有一些命令行选项可供配置. 由于这部分内容与抓包部分没有很大关联, 在此不再赘述.     
完整代码可以参考 [https://github.com/CubicPill/probe_frame_capture](https://github.com/CubicPill/probe_frame_capture)
## 在 TL-WR703N 上部署监听程序
写完监听程序之后, 下一步就是在路由器上部署监听程序. TL-WR703N 所用的芯片是 MIPS 架构, 所以我们需要 MIPS 的交叉编译器来编译最终的可执行程序.
首先, 到 OpenWrt 官方网站下载交叉编译工具链(x86 平台): [https://downloads.lede-project.org/releases/17.01.4/targets/ar71xx/generic/](https://downloads.lede-project.org/releases/17.01.4/targets/ar71xx/generic/)       
在页面最下方可以找到 Supplementary Files, 下载 SDK 压缩包.         
具体的编译步骤可以参考 repo 的 [REAMDE 文档](https://github.com/CubicPill/probe_frame_capture/blob/master/README.md)    

### 设置监听接口
由于我们需要在监听的同时保留路由器的上网功能, 所以有必要为路由器设置多个无线接口, 保留原有接口的同时, 新建一个接口 ```mon0``` 提供给监听程序.    
从 Network 选项卡中选择 Wireless, 再点击 add        
Interface Configuration 中, ESSID 可以不填, Mode 改为 Monitor, Network 不要勾选.     
![](\content\images\2018\wr703n_probe_device\monitor-1.png)


Advanced Settings 中, 将新接口命名为 ```mon0``` 
![](\content\images\2018\wr703n_probe_device\monitor-2.png)


在配置多 SSID 时, dnsmasq 可能会报错找不到用户 dnsmasq.dnsmasq, 运行如下命令即可:    

```shell
 echo "dnsmasq:*:453:dnsmasq" >> /etc/group
 echo "dnsmasq:*:453:453:dnsmasq:/var:/bin/false" >> /etc/passwd
 echo "dnsmasq:*:0:0:99999:7:::" >> /etc/shadow
 chown dnsmasq.dnsmasq /usr/sbin/dnsmasq
 ```
 
### 加载主程序 
因为 TL-WR703N 只有 4M 的 Flash 存储, 而 OpenWrt+LuCI 总共占用的空间约 3.5M, 没有足够的空间存储我们的监控程序. 因此, 只有曲线救国, 写一个脚本在每一次路由器启动时从服务器动态拉取程序     
最后经过考虑, 决定先用一个脚本从服务器上获取另外一个脚本并执行, 在第二个脚本中编写获取 Probe 帧捕获程序的命令.     
首先是第一个加载程序 ```loader.sh```      
```shell
#!/bin/sh
cd /tmp
wget -q <url of your init.sh>
chmod +x ./init.sh
./init.sh
```
拉取 init.sh 并执行     

```init.sh```

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
这样就完成了程序的部署, 现在可以在指定远程服务器的对应端口上接收到路由器发回的数据了.    

## TODO
目前这个程序只完成了基本的功能, 还有一些欠缺的地方, 比如只能监听单个频道, 发送的数据为明文等等. 以及, 在设备密集的场所, 每秒钟将会监听到数百个数据包, 这对远端服务器无疑是不小的负担( 可能同时有几十上百个路由器向服务器发送数据), 也可能会拖慢路由器的网速. 下一步可能会将一小段时间内内单个设备所发送的 probe 帧进行合并, 合并后的数据再回传服务器.




