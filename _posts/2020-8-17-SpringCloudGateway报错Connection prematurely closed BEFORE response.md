---
layout:     post
title:      SpringCloudGateway报错Connection prematurely closed BEFORE response
subtitle:   
date:       2020-08-17
author:     高强
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - SpringCloudGateway
    - 微服务
    - 问题解决
---

### 描述
- 1. 并发比较高的时候接口报错
> SpringCloudGateway报错Connection prematurely closed BEFORE response

处理:
```
hystrix:
  threadpool:
    default:
      # 核心线程池大小
      coreSize: 8    
      # 线程池队列最大值
      maxQueueSize: 200
      # 设置队列拒绝的阈值，人为设置的拒绝访问的最大队列值，即使当前队列元素还没达到maxQueueSize。 当将一个线程放入队列等待执行时，HystrixCommand使用该属性。注意：如果maxQueueSize设置为-1，该属性不可用
      queueSizeRejectionThreshold: 200
```
参考: 
[Hystrix线程池配置](https://github.com/Netflix/Hystrix/wiki/Configuration#ThreadPool)

### 问题产生
- jmeter 并发线程 300 压测接口, 通过 gateway 转发到 BFF 层, gateway 报错
```
2020-09-04 12:07:18.350  WARN [linkbar-gateway,0710a0cd84714357,0710a0cd84714357,false] 8824 --- 
[reactor-http-epoll-3] c.d.l.g.e.CustomGatewayExceptionHandler  : Exception: Connection  prematurely closed BEFORE response, method: GET, path: /h5-api/api/v1/event-interaction/show/51ceefef38dd450abed5dab980711d48
2020-09-04 12:07:18.350  WARN [linkbar-gateway,,,] 8824 --- 
[reactor-http-epoll-3] r.netty.http.client.HttpClientConnect    : [id: 0x23ac8075, L:/172.26.186.113:56210 ! R:172.26.186.113/172.26.186.113:11006] 
The connection observed an error
```

### 问题分析
- springcloud gateway, reactor-netty, maxIdleTimeout 默认不设置, 即永不过期(org.springframework.cloud.gateway.config.HttpClientProperties.Pool#maxIdleTime)
- h5-api undertow, no-request-timeout
- nginx: proxy_read_timeout         30s;
- io.undertow.Undertow#start
> OptionMap serverOptions = OptionMap.builder()
.set(UndertowOptions.NO_REQUEST_TIMEOUT, 60 * 1000)
.addAll(this.serverOptions)
.getMap();
- gateway -> auth, gateway -/> h5
- gateway 和 h5-api 保持一个 requestA, gateway 过了 59s 后新请求过来, 发现已经有 requestA 跟 h5-api 连接，复用连接，但这时候 h5-api undertow 到 60s 回收该无数据互动的请求, requestA 被迫关闭, 出现报错

### 问题解决
- 加入JVM参数 -Dreactor.netty.pool.leasingStrategy=lifo

- 新增配置
```
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          maxIdleTime: 30000
```

### 调试
- org.springframework.cloud.gateway.config.HttpClientProperties.Pool#maxIdleTime
- io.undertow.server.protocol.http.HttpReadListener#HttpReadListener
- io.undertow.Undertow#start
- nginx: proxy_read_timeout         30s;
- 设置 spring.cloud.gateway.httpclient.pool.maxIdleTime: 1200000 后，设置 jmeter 并发线程数 600, 会出现同样的问题
- gateway.httpclient.pool.maxIdleTime: 30000, 未出现该问题, 并发数高导致响应时间延长的时候可能出现 504 状态码, 分析由于 nginx 配置的 proxy_read_timeout 所dkvi

### 参考
- https://github.com/reactor/reactor-netty/issues/796
- https://github.com/reactor/reactor-netty/issues/1296
- https://blog.csdn.net/rickiyeat/article/details/107900585
  
Jmeter 调试
![Jmeter调试](/img/article/0005.png)

网关请求路由时序状态
![网关请求路由时序状态](/img/article/0006.png)

服务内存数据调试
![服务内存数据调试](/img/article/0007.png)


