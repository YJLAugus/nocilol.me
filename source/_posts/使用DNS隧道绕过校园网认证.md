---
title: 使用DNS隧道绕过校园网认证
date: 2018-09-04 02:37:00
categories: lab
tags: [校园网,DNS隧道,SSH]
urlname: using-dns-tunnel-to-bypass-campus-network-authentication
---
![あたいが・・・来たぁぁぁぁ！ @ 玄月ちひろ](https://img.imjad.cn/images/2018/09/04/66192197_p0.jpg)

{% cplayer %}
- 751472
{% endcplayer %}

## 一点~~大胆的~~想法

[前几日分析完校园网的登录认证过程](https://imjad.cn/archives/lab/giwifi-auth-process-analysis-and-simulation-login)之后已经过了几天，近日突发奇想，想试着能不能绕过校园网的这个网页认证，实现诸如免费上网、在上网区间外上网等的操作

由于本人对web安全方面所知甚少，遂又是一番搜寻，发现在此之前已经有许多先辈完成了这方面的目标。自己也试着折腾了一通

于是便拾人牙慧，凑成这篇文章


## 技术原理

经过一番搜寻，我找到了一些方法。而这些方法，虽说用的软件工具各不相同，但总体看下来，利用的都是一个原理：

为了实现网页认证，网关设备必须要给终端分配IP，同时劫持80、443端口的数据包并跳转至认证页(这也是http页才能跳转至认证页的原因，https将会无法跳转或报证书错误)。在认证通过后放行这两个端口，其他的端口也是如此，但一般情况下有一些端口却并非如此，比如本文的主角——UDP 53端口

> 类似这样的端口还有一些，比如UDP 67，用于DHCP

学过计算机网络基础知识的同学应该知道，在UDP 53上运行的是DNS协议，域名解析的数据包就通过这个端口进行传输，一般情况下网关都会放行这个端口的UDP通信，甚至是整个端口(注意是一般情况，这点之后会详细说明)。对于此类网关，要绕过认证就很简单了，只需一台使用53端口通信的代理服务器即可。这种情况过于简单，没有单独写一篇文章的必要，下面我们就来谈一谈那些「非一般」的情况(我校正是这种情况🌝)

部分网关虽然放行了53端口，却在网关处劫持了53端口的通信，这使得终端的数据根本到达不了远端，DNS解析请求会被网关抢答，而其他数据则是直接丢弃。表现就是使用`nslookup`等命令能查询到正确的解析结果，使用`tcping`等TCP数据包测试工具也能看到端口开放，延迟却是神奇的1ms(即本机到网关的延迟)

那么就没有办法了吗？非也，要知道劳动人民的智慧是无穷的，为了能够免费上网，这些极客们发明了许多神奇的，堪称黑科技的，甚至细细品味一番会发现有些搞笑的方法。

*这段是在调侃，「免费上网」只是表面，背后的思想及技术验证才是最可贵的*

比如给我个人印象颇深的利用ICMP报文(即`ping`命令的数据包)和本文介绍的利用DNS查询请求报文来进行数据通信，这两种通信方式被形象的称为「PING隧道」和「DNS隧道」

但需要说明的是，这两种方法虽然实践证明可行，但由于对应的协议设计不是用来传输数据的，强行使其支持数据传输后，速度也是慢的离谱，并且大量频繁的Ping请求或DNS查询也很容易被甄别为异常流量从而遭到封锁。因此本文的意义更在于「折腾」的过程，而不是结果。~~买一张无限流量卡开热点，相比这些体验要强多了，毕竟能用钱解决的事都不算事嘛~~~


## DNS隧道

实现DNS隧道有许多种方法，这里使用工具**dns2tcp**来实现

这种方法需要一台Linux系统的外部VPS来作为代理服务器、一个可配置NS解析的域名作为传输中介

**另请注意，本文出现的配置文件及命令的参数值不可直接套用，需根据实际情况进行修改**

### 在域名解析面板的操作

国内支持NS记录解析的DNS解析服务据我所知有DNSPod和CloudXNS，这里使用CloudXNS

如图所示，需要添加两条记录：

一条A记录，主机名任意，这里使用`temp1`，值为VPS的IP

一条NS记录，主机名任意但需牢记，这里使用`dns2tcp`，值为A记录的主机名对应的完整域名，这里是`temp1.nocilol.me`

![域名解析](https://img.imjad.cn/images/2018/09/04/sp180903_230655.png)


### 在服务端的操作

#### 防火墙配置

首先需要在安全组和防火墙规则中放行53端口的UDP连接，安全组参考各IDC的配置教程，防火墙若使用iptables的话，添加如下规则：

```shell
-A INPUT -p udp -m udp --dport 53 -j ACCEPT
-A OUTPUT -p udp -m udp --sport 53 -j ACCEPT
```


#### 安装dns2tcp

一般Linux发行版的软件仓库里都带了dns2tcp，直接安装即可，若无的话可以自行编译，项目地址为[alex-sector/dns2tcp](https://github.com/alex-sector/dns2tcp)

*Windows系统不支持dns2tcp的服务端dns2tcpd，这点需要注意*

在Debian系的发行版中使用下列命令安装：

```shell
# apt install dns2tcp
```

工具不大，很快就会安装完毕


#### 配置dns2tcpd

配置文件默认目录为`~/.dns2tcpdrc`，也可在随后启动`dns2tcpd`时以`-f`参数指定配置文件路径

配置文件内容为：

```
listen = 0.0.0.0
port = 53
user = nobody
key = your_own_password
chroot = /tmp
domain = dns2tcp.nocilol.me
resources = ssh:127.0.0.1:22 , smtp:127.0.0.1:25
```

其中的`key`字段值为口令，可以防止不知道口令的人连接到本机的dns2tcp服务


#### 运行dns2tcpd

调用说明

```
Usage : dns2tcpd [ -i IP ] [ -F ] [ -d debug_level ] [ -f config-file ] [ -p pidfile ]
         -F : dns2tcpd will run in foreground
```

执行下列命令测试启动服务端进程：

```shell
# dns2tcpd -F -d 1
```

如果配置文件无误，应当会输出类似如下的内容：

```shell
15:45:14 : Debug options.c:97   Add resource ssh:127.0.0.1 port 22
15:45:14 : Debug options.c:97   Add resource smtp:127.0.0.1 port 25
15:45:14 : Debug socket.c:55    Listening on 0.0.0.0:53 for domain dns2tcp.nocilol.me
Starting Server v0.5.2...
15:45:14 : Debug main.c:132     Chroot to /tmp
15:45:14 : Debug main.c:142     Change to user nobody
```

实际使用时执行下列命令以守护进程的方式在后台运行：

```shell
# dns2tcpd -d 1
```


### 在客户端的操作

从这一步开始就可以断网操作了，断网指的是连接热点或插上网线，但不进行认证的状态

dns2tcp的安装过程不再赘述，Windows系统用户可以自行从源代码编译或在[这个页面](http://azertyfab.free.fr/dns2tcp/)直接下载到编译好的dns2tcpc.exe

#### 配置dns2tcpc

配置文件默认目录为`~/.dns2tcpdrc`，也可在随后启动`dns2tcp`时以`-f`参数指定配置文件路径

配置文件内容为：

```
domain = dns2tcp.nocilol.me
resource = ssh
local_port = 2222
debug_level = 2
key = your_own_password
# server = your_vps_ip
```


#### 运行dns2tcpc

调用说明

```
dns2tcp v0.5.2 ( http://www.hsc.fr/ )
Usage : dns2tcpc [options] [server] 
	-c         	: enable compression
	-z <domain>	: domain to use (mandatory)
	-d <1|2|3>	: debug_level (1, 2 or 3)
	-r <resource>	: resource to access
	-k <key>	: pre-shared key
	-f <filename>	: configuration file
	-l <port|->	: local port to bind, '-' is for stdin (mandatory if resource defined without program )
	-e <program>	: program to execute
	-t <delay>	: max DNS server's answer delay in seconds (default is 3)
	-T <TXT|KEY>	: DNS request type (default is TXT)
	server	: DNS server to use
	If no resources are specified, available resources will be printed
```

执行下列命令测试与服务端的连接是否通畅：

```shell
$ dns2tcpc -z dns2tcp.nocilol.me -d 3 -k your_own_password
```

如果一切正常，将输出类似下面的内容：

```
Available connection(s) : 
	ssh
	smtp

Note : Compression SEEMS available !
No DNS given, using 223.5.5.5 (first system preferred DNS server)
debug level 3
Debug socket.c:233	Create socket for dns : '223.5.5.5' 
Debug session.c:46	Request challenge
Debug requests.c:146	Sending dns id = 0x3f0b
Debug requests.c:97	Query is AAAAABV/AA.=auth.dns2tcp.nocilol.me len 35
Debug rr.c:106	rr_decode_next_reply_encode base64 data was = AQUAABV/AFkxN0pEMEpZT0dUR1lJUUI (reply len = 34)
Debug session.c:53	Challenge = 'Y17JD0JYOGTGYIQB'
Debug session.c:54	Session created (0x501)
Debug session.c:77	Sending response : '11F5Ab37A7DE310AC28F396B75E45B186D24AB39' (key = your_own_password) 
Debug requests.c:146	Sending dns id = 0xd21f
Debug requests.c:97	Query is AQWBgAABADExRjVBQzE3QTdERTMzMEFDMjlGNzM2Qjc1RTQzNzE1NkIyNEFCNjk.=auth.dns2tcp.nocilol.me len 88
Debug rr.c:106	rr_decode_next_reply_encode base64 data was = AQWBgAABAA (reply len = 13)
Debug auth.c:58	Requesting resource
Debug requests.c:146	Sending dns id = 0x2e29
Debug requests.c:97	Query is AQWPJOMjAA.=resource.dns2tcp.nocilol.me len 39
Debug rr.c:106	rr_decode_next_reply_encode base64 data was = 1FwCb/x/AHNzaA (reply len = 17)
Debug rr.c:106	rr_decode_next_reply_encode base64 data was = 1FwCb/x/AHNtdHA (reply len = 18)
```

如果输出类似下面的内容，请检查操作是否有误，若操作无误，请等待一段时间，DNS解析缓存需要一定时间来刷新和传递：

```
No DNS given, using 223.5.5.5 (first system preferred DNS server)
debug level 3
Debug socket.c:233	Create socket for dns : '223.5.5.5' 
Debug session.c:46	Request challenge
Debug requests.c:146	Sending dns id = 0xbc7b
Debug requests.c:97	Query is AAAAAOt7AA.=auth.dns2tcp.nocilol.me len 35
No response from DNS 223.5.5.5
```

测试成功之后，执行下列命令启动客户端进程：

```shell
$ dns2tcpc
```

将会输出类似下面的内容：

```
debug level 2
Debug socket.c:233	Create socket for dns : '223.5.5.5' 
Listening on port : 2222
When connected press enter at any time to dump the queue
```

保持这个终端窗口打开，不要关闭


#### 转换为SOCKS v5通用代理

##### Linux

新打开一个终端窗口并执行下列命令~~是不是有点似曾相识~~：

```shell
$ ssh user@127.0.0.1 -p 2222 -D 2333
```

##### Windows

使用SSH连接工具的隧道功能，这里给一个Xshell的配置示例：

![连接至dns2tcpc转发的ssh端口](https://img.imjad.cn/images/2018/09/04/sp180904_095832.png)


![SSH隧道配置](https://img.imjad.cn/images/2018/09/04/sp180904_095617.png)

如果一切正常，SSH隧道将连接成功，同时可以观察到dns2tcpc的终端打印出大量日志信息


#### 配置系统代理或软件代理

在系统或软件的代理设置中配置为SOCKS v5，地址`127.0.0.1`，端口`2333`

具体配置方法不再详述，完成后即可正常上网


## 体验如何

首先泼一盆凉水，体验可以用「糟糕」来形容:(

本文开头曾提到：「…虽然实践证明可行，但由于对应的协议设计不是用来传输数据的，强行使其支持数据传输后，速度也是慢的离谱，并且大量频繁的Ping请求或DNS查询也很容易被甄别为异常流量从而遭到封锁。」

而具体体验如何，我就贴两张图吧，一切尽在不言中🌝

![浏览器的下载速度](https://img.imjad.cn/images/2018/09/04/TIM20180904101615.png)

![百度首页首次加载时间](https://img.imjad.cn/images/2018/09/04/TIM20180904101626.png)


虽然体验不尽如人意，但是生命在于折腾嘛。在折腾的过程中，能够了解到在小小的Web认证的背后还有如此多的、有趣的知识，比起最初的目的而言，要重要的多了。

人生也应大抵如此吧


## 参考

[透过DNS Tunnel绕过校园网认证系统实现免费上网](https://qiuri.org/posts/2152501839/)

[绕过校园网WEB认证_dns2tcp实现](https://www.cnblogs.com/nkqlhqc/p/7805837.html)