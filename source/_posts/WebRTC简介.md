---
title: WebRTC简介
date: 2022-04-04 22:31:30
tags:
 - 原创
 - WebRTC

categories: WebRTC

---


# 什么是WebRTC

WebRTC 全称网页及时通信（Web Real-Time Communication），是一个支持网页浏览器进行实时音视频对话的API ，于2011年6月1日开源，并被纳入W3C标准。以被众多浏览器支持，如Edge,Chrom,Firefox,Opera等

![image](https://tva2.sinaimg.cn/large/8dfd1ceegy1h0istpz079j22jy0fygv0.jpg)

# WebRTC 整体示意图

![WebRTC 1对1 音视频交互图](https://tva1.sinaimg.cn/large/8dfd1ceegy1h0l13tijgzj20qw0ciacw.jpg)

WebRTC主要用在音视频的实时通信上，上图是WebRTC 1对1音视频通信的示意图，通过这个示意图，我们可以明白，使用WebRTC开发音视频的实时通信的整体结构包括：

* WebRTC终端，主要负责音视频的采集/展示，编/解码，NAT 穿越和数据传输等
* Signal服务器，主要负责信令处理（加入房间/退出房间，以及媒体协商SDP等），可以使用Websocket/Socket等实现
* STUN/TURN 服务器，主要负责获取WebRTC终端的公网IP地址或NAT穿越失败的数据中转（若NAT穿越成功，则采用P2P通信，反之，采用中转服务器中转）



## 交互流程

WebRTC端到端建立连接，并进行音视频实时通信的大体过程如下

1. WebRTC两个客户端分别与Signal 服务器建立连接 ，Signal 服务端为WebRTC端分配房间/加入指定的房间，并返回WebRTC房间信息
2. WebRTC端会创建`RTCPeerConnection`媒体连接，这个连接需要知道双方的流媒体数据格式才能进行后续的数据传输，它们通过Signal 服务端进行`SDP`媒体协商
   1. WebRTC-1先创建`RTCPeerConnection`媒体连接，并生成Offer请求（包含了它这个客户端支持的的媒体格式等内容），并将其设置到`RTCPeerConnection`的`LocalDescription`，然后向Signal 服务器发送Offer 请求,由其转发给WebRTC-2端。
   2. WebRTC-2端收到了Offer请求，也会创建`RTCPeerConnection`媒体连接，并将Offer请求中对端支持的SDP 设置到`RTCPeerConnection`的`RemoteDescription`,同时生成Answer应答，并将其设置到`RTCPeerConnection`的`LocalDescription`,然后发送到Signal 服务器，由其转发给WebRTC-1
   3. WebRTC-1 收到Answer，将`SDP`设置到`RTCPeerConnection`的`RemoteDescription`，完成媒体协商
3. WebRTC两端都有了对端的连接信息和媒体协商信息，就可以进行连接，并通讯



注意：WebRTC-1和WebRTC-2建立连接的方式有P2P和经过中转服务器中转两种方式，其中`SDP`中就包括对端的连接信息，这个信息是在生成`SDP`时，会和`STUN/TURN`服务器进行通讯，收集该WebRTC 端的ICE（包含如何与其建连的信息）,然后WebRTC 客户端就可以通过对方的ICE 进行连接。



# WebRTC是如何建连

WebRTC是支持浏览器之间建立连接的连接，但是浏览器所在主机之间的网络环境可能差异很大，有的主机在局域网内，有的在公网上，局域网和公网的主机通讯需要经过NAT，所以需要先了解有哪些类型的NAT,在看看WebRTC如何为不同网络环境的机器之间建立连接

## NAT 种类

### 完全锥形NAT

主机host 通过NAT 访问外网B,在NAT上就会打个”洞“，所有知道这个”洞“的外网主机都可以通过这个与host 上的侦听程序通讯。这个”洞“对应NAT 映射:

>  内网IP:内网Port<-->外网IP:外网Port

那么机器A或C 就可以通过这个外网IP:外网Port 和host 上的进程通讯了

![image](https://tvax4.sinaimg.cn/large/8dfd1ceegy1h0iuw8eeohj21fc0vwjx0.jpg)

### IP限制锥形NAT

IP 限制锥型要比完全锥型 NAT 严格得多，它主要的特点是，host 主机在 NAT 上“打洞”后，NAT 会对穿越洞口的 IP 地址做限制。只有登记的 IP 地址才可以通过，也就是说，只有**host 主机访问过的外网主机才能穿越 NAT**

也就是NAT 映射表为：

>  内网IP:内网Port<-->外网IP:外网Port<-->被访问的主机IP

那么这里只有B可以和X 通信，而A，C 由于IP 未被访问，故无法与其通信

![image](https://tvax3.sinaimg.cn/large/8dfd1ceegy1h0iuvhs9kjj21f80w40yo.jpg)

### 端口限制锥型

端口限制锥型要比IP限制形还要严格，它主要的特点是，host 主机在 NAT 上“打洞”后，NAT 会对穿越洞口的 IP 地址和端口做限制。也就是说，只有**host 主机访问过的外网主机及提供服务的程序的端口才能穿越 NAT**

也就是NAT 映射表为

>  内网IP:内网Port<-->外网IP:外网Port<-->被访问的主机IP：被访问主机Port

那么这里只有B上的P1端口的进程才能和其通信

![image](https://tvax2.sinaimg.cn/large/8dfd1ceegy1h0iuu6fs3wj21fo0von3j.jpg)

### 对称型NAT

这是NAT 中最严格的一种类型，也就是说host 主机访问A 时会打一个”洞“，访问B是会再打一个”洞“，也即六元组中

>  内网IP:内网Port<-->外网IP:***外网Port***<-->被访问的主机IP：被访问主机Port

不同的被访问主机对应不同的**外网Port**

![image](https://tva3.sinaimg.cn/large/8dfd1ceegy1h0iv20kiroj21g40ve7ap.jpg)

那么host 主机怎么知道自己的网络属于哪一种类型的？

我们需要一台公网的服务器，且它拥有两个网卡，那么我们就可以通过如下步骤判断它的NAT环境

1. 主机向服务器#1的一个网卡和端口发送请求,服务器会用相同的IP和端口发送响应
2. 若主机收不到响应，说明主机的网络限制了UDP协议，直接退出
3. 如果能收到回包，且返回的主机的IP 和主机自身的IP 一样，说明主机拥有公网IP,如果不一样，则说明主机处于NAT防护下
4. 如果主机拥有公网IP,需要判断它的防火墙类型，此时可以再向服务器\#1网卡和端口发送请求，然后服务器\#1另一个网卡网卡和端口进行回包
5. 如果主机能收到，说明它是没有防护的公网主机，反之，则收到对称型防火墙保护

若第3步中判断主机在NAT防护下，需要判断它是在哪种类型的NAT 环境，步骤如下

1. 主机向服务器#1一个网卡和端口发送请求，服务器#1用另一个网卡和不同端口回包

2. 若主机收到消息，说明是完全锥形NAT

3. 若收不到，向服务器#2一个网卡和端口发送请求，并用相同的网卡和端口回包

4. 主机收到消息后比对服务器#1和#2返回的主机的外网的IP 和端口是否一致

   ，如果不一致，则是对称型NAT（对称型NAT的特点就是与不同机器通讯都会打不同的”洞“）

5. 若放回的外网IP和端口一样（说明使用的是同一个”洞“），这时候只需要判断是IP限制还是端口限制，可以向服务器#1 的一个网卡和端口发送请求，并用相同网卡的不同端口回包

6. 若能收到就是IP限制型，收不到就是端口限制型



我们知道不同限制型的NAT限制的严格程度为

>  对称型NAT>端口限制NAT >IP限制NAT >完全锥形NAT

我们上述方法先排除对称型和完全锥形，在区分是IP 限制还是端口限制。

对称型 NAT 与对称型 NAT 之间以及对称型 NAT 与端口限制型 NAT 之间无法打洞

## WebRTC如何建连

WebRTC建连是比较复杂，它要考虑端到端之间的连通性和数据传输的高效性，所以当有多条有效连接时，需要选择传输质量最好的线路，如能用内网连通就不用公网，如果公网也连通不了（比如双方都是处在对称型NAT下），那么就需要通过服务端中继的方式。

![image](https://tvax3.sinaimg.cn/large/8dfd1ceegy1h0jndi6a6rj20r10d7wh2.jpg)

WebRTC建连就是处理图中红色框的部分。

### 双方处于同一网段

* 直接内网连接
* 通过公网绕一圈后再连接

### 双方位于不同网段

* P2P的方式建立连接（需要使用NAT打”洞“），优先使用
* 通过中继服务器中转

### ICE 候选者

ICE（Interactive Connectivity Establishment ，ICE） 候选者表示WebRTC 与远端通信是使用的协议、IP、端口等，如

```json
{
  IP: xxx.xxx.xxx.xxx,  //本地IP
  port: number, //端口
  type: host/srflx/relay, //类型，host 表示本机候选者，srflx 表示内网主机映射的外网的地址和端口，relay 表示中继候选者
  priority: number,  //优先级
  protocol: UDP/TCP,
  usernameFragment: string
  ...
}
```

WebRTC建立连接是选择的ICE优先级为 host> srflx > relay

那么srflx （外网IP）和relay(中继IP)是怎么获取？这就需要了解`STUN`和`TURN`协议

#### STUN协议

通过上述NAT 介绍，我们知道我们可以向公网的服务器发送一个请求，然后服务器返回主机的外网IP和端口，这个就是STUN协议，遵守该协议就能拿到自己的公网IP(在公网上部署STUN服务器，内网主机向其发送binging request 即可获得自己的外网IP和端口）

#### TURN协议

relay型候选者的获取是通过也是STUN获取，只不过消息类型不一样，主要使用Allocation指令。主机向TURN 服务器发送Allocation指令，relay服务器会子啊服务端分配新的端口，用于中转UDP数据报



# SDP 简介

`SDP`简称Session Description Protocal，用来描述各端支持的音视频编解码格式和连接相关的信息。主要包括以下几个部分：

* Session Metadata，会话元数据
* Network Description，网络描述
* Stream Description，流描述
* Security Descriptions，安全描述
* Qos Grouping Descriptions， 服务质量描述

以下是会话描述和

```
//==========以上表示会话描述==========
v=0    //protocol version
o=- 4007659306182774937 2 IN IP4 127.0.0.1     //username sessionID version networkType addressType IP
s=-           //sessionName
t=0 0   //session的开始和结束时间，0 0 表示持久会话

...
//下面的媒体描述，在媒体描述部分包括音频和视频两路媒体
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 110 112 113 126  
//媒体类型audio 端口9 传输协议 流媒体格式
...
a=rtpmap:111 opus/48000/2 //对RTP数据的描述
a=fmtp:111 minptime=10;useinbandfec=1 //对格式参数的描述
...
a=rtpmap:103 ISAC/16000
a=rtpmap:104 ISAC/32000
...
//上面是音频媒体描述，下面是视频媒体描述
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 102 122 127 121 125 107 108 109 124 120 123 119 114 115 116
...
a=rtpmap:96 VP8/90000
...


...
//=======安全描述============
a=ice-ufrag:1uEe //进入连通性检测的用户名
a=ice-pwd:RQe+y7SOLQJET+duNJ+Qbk7z//密码，这两个是用于连通性检测的凭证
a=fingerprint:sha-256 35:6F:40:3D:F6:9B:BA:5B:F6:2A:7F:65:59:60:6D:6B:F9:C7:AE:46:44:B4:E4:73:F8:60:67:4D:58:E2:EB:9C //DTLS 指纹认证，以识别是否是合法用户
...
//========服务质量描述=========
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 goog-remb //使用 google 的带宽评估算法
a=rtcp-fb:96 transport-cc //启动防拥塞
a=rtcp-fb:96 ccm fir //解码出错，请求关键帧
a=rtcp-fb:96 nack    //启用丢包重传功能
a=rtcp-fb:96 nack pli //与fir 类似
...
```

# 多人音视频通讯的架构？

上面介绍的1对1的通讯模型，若要实现多对多的通信，一般有三种方案

1. Mesh方案。各个终端两两之间建立连接，形成网状。
2. MCU（Multipoint Conferencing Unit） 方案。服务端分别和各个客户端建立连接，组成星形结构，所有终端发送的音视频需要咋服务器进行混合，之后再发送给客户端。
3. SFU （Selective Forwarding Unit）方案。不同于MCU,这里不对音视频混合，仅仅转发给房间内的其他终端



![image](https://tvax4.sinaimg.cn/large/8dfd1ceegy1h0l4uqwe38j20ce08n75p.jpg)

[Mesh 架构]

![image](https://tvax1.sinaimg.cn/large/8dfd1ceegy1h0l4vynzvqj20ah08o3zc.jpg)

[MCU架构]

![image](https://tva2.sinaimg.cn/large/8dfd1ceegy1h0l4wunu29j20ag08rgmv.jpg)

[SFU架构]

|      | 优点                                         | 缺点                                                         |
| ---- | -------------------------------------------- | ------------------------------------------------------------ |
| Mesh | 简单，不需要中转数据，不需要开发媒体服务器   | 上行带宽压力大，资源消耗大，兼容各种客户端                   |
| MCU  | 技术成熟，客户体验号                         | 服务端编解码和混流资源消耗大，延迟高                         |
| SFU✅ | 服务端资源消耗小，延时低，能适应不同终端类型 | 用户体验不好（音视频不同步），客户端使用复杂（混流，渲染等） |



# 参考文献

* https://zh.wikipedia.org/wiki/WebRTC
* https://time.geekbang.org/column/intro/100031801

