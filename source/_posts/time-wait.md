---
title: time wait的问题
date: 2019-06-15 15:58:48
tags:
 - 计算机网络
 - 计算机基础
categories:
 - 计算机网络
---
<meta name="referrer" content="no-referrer" />
  有一天突发奇想，上去服务器去看看服务器的连接状态。发现大部分连接都是establish，有一小部分是time_wait。
  
  正常情况tcp连接成功后自然是establish状态，但是time_wait是怎么来的呢？查询过资料time_wait过多会影响服务器性能？

# time_wait怎么来的
## tcp四次挥手
 ![](time-wait/fin.jpg)
 
 上面是一个TCP连接四次挥手的过程。从图中得知主动关闭tcp连接的一方会进入time_wait状态，等待2MSL时间后才进入close。
 
 一般情况关闭tcp连接的服务器一段，如果此时有一部分连接开始关闭了(time_wait存在时间够长)，可以解析这时有一部分连接是处于time_wait。

## 为什么需要time_wait
 为什么需要time_wait这一个过渡状态，不可以直接进入close吗？

### (1)防止ack包丢失
 在tcp的四次挥手中，最后一个ACK包是主动关闭方发送的，用于响应被动关闭方发送的FIN包。 由于最后一个发送的ACK包有可能会丢失，被动关闭方会在一个超时时间后再发送一个FIN包。所以主动关闭方必须维持这个状态接受重发的FIN包和重发ACK包。

### (2)允许老的重复分节在网络中消逝
 在进行四次挥手关闭连接前，发送的数据包有可能因为网络延迟暂时没有发送到目的地。若没有time_wait后面tcp连接关闭了，服务器建立tcp连接有可能会复用以前的端口。在建立新的连接后，之前迷路的包重新到达目的端口。这个时候收到一个不是连接对方的包，会发送一个RCT包强制中断这次连接。为了避免这个尴尬的情况，关闭连接保持time_wait状态，让迷路的包在time_wait下全部收到。

## 为什么要等2msl
 MSL是maximum segment lifetime(最大分节生命期），这是一个IP数据包能在互联网上生存的最长时间，超过这个时间将在网络中消失。 time_wait状态保持2MSL，就能保证被动关闭方没有重发FIN包，所有迷路的数据包都在这个时候收到了。

# 为什么会出现大量time_wait
 上面可以知道，time_wait是主动关闭tcp连接的一个过渡状态，可以得出有如下情况会出现大量time_wait
 1. 高并发的短链接。服务器处理大量短链接的时候，大部分的连接都处于结束状态就有大量的time_wait。
 2. 服务器的连接数达到瓶颈。服务器达到瓶颈时候会断开一些无用连接这个时候就有大量time_wait。
 
 大量的time_wait会占用服务器的连接数，且浪费服务器资源去处理time_wit状态，严重影响服务器性能。

# 如何处理大量time_wait
 time_wait的存在十分合理，但是多起来对服务器影响又很大。这个问题好像变的无解了。其实并不是主要两个方向解决，连接的快速回收和复用。
 
 下面处理都是/etc/sysctl.conf的参数调整。以下我们先熟悉以下该配置重要参数。
 
 ```
 net.ipv4.tcp_timestamps
 ```
 
 RFC 1323 在 TCP Reliability一节里，引入了timestamp的TCP option，两个4字节的时间戳字段，其中第一个4字节字段用来保存发送该数据包的时间，第二个4字节字段用来保存最近一次接收对方发送到数据的时间。有了这两个时间字段，也就有了后续优化的余地。tcp_tw_reuse 和 tcp_tw_recycle就依赖这些时间字段。
 
 ```
 net.ipv4.tcp_tw_reuse
 ```
 
 出现TIME_WAIT状态的连接，一定出现在主动关闭连接的一方。所以，当主动关闭连接的一方，再次向对方发起连接请求的时候（例如，客户端关闭连接，客户端再次连接服务端，此时可以复用了；负载均衡服务器，主动关闭后端的连接，当有新的HTTP请求，负载均衡服务器再次连接后端服务器，此时也可以复用），可以复用TIME_WAIT状态的连接。
 
 ```
 net.ipv4.tcp_tw_recycle
 ```
 
 当开启了这个配置后，内核会快速的回收处于TIME_WAIT状态的socket连接。一个RTO（retransmission timeout，数据包重传的timeout时间）的时间后回收，这个时间根据RTT动态计算出来，但是远小于2MSL。当服务器再次使用这个端口(这里不是重启)，内核里会记录包括该socket连接对应的五元组中的对方IP等在内的一些统计数据，当然也包括从该对方IP所接收到的最近的一次数据包时间。当有新的数据包到达，只要时间晚于内核记录的这个时间，数据包都会被统统的丢掉。
 
 ![](time-wait/struct.jpg)
 
 一般服务架构如上图,客户端通过HTTP/1.1连接负载均衡，也就是说，HTTP协议投Connection为keep-alive，所以我们假定，客户端 对 负载均衡服务器 的socket连接，客户端会断开连接，所以，TIME_WAIT出现在客户端Web服务器和MySQL服务器的连接，我们假定，Web服务器上的程序在连接结束的时候，调用close操作关闭socket资源连接，所以，TIME_WAIT出现在 Web 服务器端。
 
  所以Web服务器上，肯定可以配置开启的配置：tcp_tw_reuse；如果Web服务器有很多连向DB服务器的连接，可以保证socket连接的复用。那么，负载均衡服务器和Web服务器，谁先关闭连接，则决定了我们怎么配置tcp_tw_reuse/tcp_tw_recycle了
 
## 负载均衡 首先关闭连接 
 在这种情况下，因为负载均衡服务器对Web服务器的连接，TIME_WAIT大都出现在负载均衡服务器上，
 ```
 # 在负载均衡服务器上的配置
 net.ipv4.tcp_tw_reuse = 1 //尽量复用连接
 net.ipv4.tcp_tw_recycle = 0
 
 # 在Web服务器上的配置
 net.ipv4.tcp_tw_reuse = 1 //这个配置主要影响的是Web服务器到DB服务器的连接复用
 net.ipv4.tcp_tw_recycle： 设置成1和0都没有任何意义
 
 ```
 在负载均衡和它的连接中，它是服务端，但是TIME_WAIT出现在负载均衡服务器上；它和DB的连接，它是客户端，recycle对它并没有什么影响，关键是reuse。

## Web服务器首先关闭来自负载均衡服务器的连接

 在这种情况下，Web服务器变成TIME_WAIT的重灾区。负载均衡对Web服务器的连接，由Web服务器首先关闭连接，TIME_WAIT出现在Web服务器上。；Web服务器对DB服务器的连接，由Web服务器关闭连接，TIME_WAIT也出现在web服务器上。
 
 ```
 # 负载均衡服务器上的配置
 net.ipv4.tcp_tw_reuse：0 或者 1 都行，都没有实际意义
 net.ipv4.tcp_tw_recycle=0 //一定是关闭recycle
 
 # 在Web服务器上的配置
 net.ipv4.tcp_tw_reuse = 1 //这个配置主要影响的是Web服务器到DB服务器的连接复用
 net.ipv4.tcp_tw_recycle=1 //由于在负载均衡和Web服务器之间并没有NAT的网络，可以考虑开启recycle，加速由于负载均衡和Web服务器之间的连接造成的大量TIME_WAIT
 ```
