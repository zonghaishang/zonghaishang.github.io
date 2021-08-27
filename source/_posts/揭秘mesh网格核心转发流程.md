---
title: 揭秘mesh网格核心转发流程
subtitle: 本文重点对mosn核心转发流程做了详细的梳理。
cover: 
author: 
  nick: 诣极
  link: https://github.com/zonghaishang
tags:
 - mosn
 - service mesh
categories:
 - service mesh
date: 2021-08-27 13:09:10
---

文档修订历史

| 版本号 | 作者 | 备注 | 修订日期 |
| --- | --- | --- | --- |
| 0.1 | 诣极 | 初始版本 | 2021.8.27 |



为了帮助同学深入掌握mesh转发核心流程，本文重点对mosn核心转发流程做了详细的梳理。详细流程图请参考图下方图1-1所示。


首先，mosn作为数据面，整体分为4层 (对应图1-1 从下往上分隔分层), 原始的mosn简介详见[doc](https://github.com/mosn/mosn/blob/0.1.0/docs/design/README.md) ：

- network/io 主要负责网络读写
- protocol 主要负责协议编解码
- stream 主要负责request、response等模型转发和协调
- proxy 主要负责路由等功能。


接下来，帮助同学清晰的掌握mosn的流程，整个流程简化为12个步骤，我会逐一拆解分析：



1. mosn在启动期间，会暴露本地egress端口接收客户端app的rpc请求。mosn会开启2个协程，分别死循环去对tcp进行读取和写处理。mosn会通过读协程获取到请求字节流，进入mosn的协议层处理。
1. mosn会通过[streamconn](https://github.com/mosn/mosn/blob/d1e857f3dc0d4c6dfc8d17189d2300b7e0cfac78/pkg/stream/xprotocol/conn.go#L162)实现类中循环解码，直到解析到完整的请求报文。一旦解析到请求报文，会创建和请求frame关联的[xstream](https://github.com/mosn/mosn/blob/d1e857f3dc0d4c6dfc8d17189d2300b7e0cfac78/pkg/stream/xprotocol/conn.go#L304)。这里的xtream用来保持和客户端app的tcp关联，后续用来用于响应response。
1. mosn需要将请求转发到服务集群的某一台机器，会到达proxy层创建downstream, 有以下目的：
   - 执行filter请求/响应链
   - 执行路由匹配
   - 执行负载均衡
4. 在选择服务集群的某一台后，会首先初始化选中ip对应host的连接池，此时mosn的角色变成了中间人的角色。一方面需要承担客户端app的服务端，另一方面需要承担远程服务方的客户端。

[upstreamrequest](https://github.com/mosn/mosn/blob/18e28b5fb9aa7a634736eb20ac321dc86e915067/pkg/proxy/downstream.go#L866) 对象起到关键作用：

   - 保持着对客户端app的tcp引用，通过持有downstream
   - 保持着对转发服务端tcp引用，转发客户端app请求以及响应服务端response时的通知
5. 在upstream的[appendHeaders](https://github.com/mosn/mosn/blob/18e28b5fb9aa7a634736eb20ac321dc86e915067/pkg/proxy/upstream.go#L177) + [appendData](https://github.com/mosn/mosn/blob/18e28b5fb9aa7a634736eb20ac321dc86e915067/pkg/proxy/downstream.go#L903)阶段，会用第4步骤中选择的host创建sender xstream, 这个xstream是客户端的流对象，主要有2个目的：
   - 充当client角色，初始化客户端请求信息，将待转发的请求创建对应的stream绑定关系
   - 在真正转发前，需要替换请求id信息，用来解决连接io复用导致请求互相覆盖的问题
      - 混淆原因： 客户端app有多个tcp连接，mosn转发到服务端只有1条cp连接，如果客户端2个tcp连接同时有个id=1的request，mosn会通过同一条tcp转发给服务端，因此响应回来时，mosn没办法区分id=1的response属于哪个客户端app的tcp连接了。这里解决办法，mosn转发到服务端的1条tcp连接，会重新修改请求的id成全局，这个时候就不会混淆请求和响应关联关系了。
6. 因为mosn在转发之前修改了请求id，因此会重新encode请求，一般优化手段不会encode完整报文，只会修改协议头的个别几个字节。
6. 一旦客户端xstream准备转发就绪(endOfStream)，就会通过第4步骤中选择下游host直接发送，此时请求的携程会被阻塞。
6. mosn转发给服务端host时，我们知道会新建tcp连接，与此同时也会每个tcp连接也会有2个携程去处理读写。mosn客户端xstream的会通过读io，收到响应byte字节流，并且交付上层protocol去解码。
6. 一旦完整解码成response对象，会通知[upstream request](https://github.com/mosn/mosn/blob/d1e857f3dc0d4c6dfc8d17189d2300b7e0cfac78/pkg/stream/xprotocol/conn.go#L371)对象。
6. upstream request持有客户端请求的downstream, [唤醒](https://github.com/mosn/mosn/blob/18e28b5fb9aa7a634736eb20ac321dc86e915067/pkg/proxy/upstream.go#L124)downstream阻塞的协程。
6. 对应步骤2中mosn作为服务方xstream被唤醒，会将收到的响应response, 重新替换回正确的reqest id，并能且去调用协议层重新encode成字节流。
6. xstream中持有客户端app请求时的tcp连接，直接将响应写会客户端，并且销毁mosn中所有请求相关的资源。至此，完整的请求响应流程在mosn中的流程介绍完毕。

![mosn核心流程.png](https://zonghaishang.github.io/images/mosn核心流程.png)
图 1-1 mosn核心流程转发

本文对完整的mosn转发流程做了关键的讲解，比如更多mosn中小功能点，比如buffer、router、connection pool的实现机制，会放到进阶原理篇给大家详细介绍。
