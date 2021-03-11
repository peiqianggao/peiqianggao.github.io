---
layout:     post
title:      SpringCloud微服务中出现failed and fallback报错解决
subtitle:   SpringCloud 微服务间交互出现非超时导致的失败情况处理
date:       2020-06-02
author:     高强
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - SpringCloud
    - 微服务
    - 问题解决
---

### 描述
通过接口初始化某种类型的统计数据，开发环境执行没问题，线上环境执行报错。微服务A调微服务B接口时报错，但微服务B没看到请求日志。

### 思路
- 两个服务是否部署的是最新包
> 通过 arthas#trace/jad 验证

- 服务注册列表
> 服务A是否有拉取到服务B的注册服务列表，通过 nacos 和 服务A其他调用到服务B的接口间接验证

- feign 限制
> 服务B没有调用日志，说明请求还在服务A里面就被限制了，考虑服务A#feign 限制，比如 hystrix, 线程栈大小, 参数属性和其他限制  
> feign 调用包装在 hystrix 隔离线程里面如果线程栈溢出，异常会不会被 CustomFeignException 或者 HystrixException 吞掉不打出来

- @GetMapping 里面的路径会不会因为正则匹配、jdk版本或其他原因被拦截
> 之前有个案例是开发告警服务，配置多个接收者邮箱时遇到过开发没问题线上有问题的情况

### 原因
- header 请求数据量超出 8k 限制
> @RequestParam 标识的接口为 query 参数，在 http 协议中该部分数据将置于 header 区域传输  
> 接口入参包含列表入参，开发环境由于数据量少，列表入参数据量有限，但到线上该列表入参由于没有做分页处理，导致数据量庞大出现此问题

### 解决
- 服务A调服务B中接口提供方 @RequestParam 入参改为 POST 请求 @RequestBody 入参

### 结论
- 大数据量尽量分页处理或分块处理
- 请求 query 参数由于浏览器或Http协议限制，尽量简短，参数量大时考虑使用 POST/PUT 请求方式实现

### 参考
- [Spring boot 使用 feign 调用参数过长（Post变Get）](https://zhuanlan.zhihu.com/p/111637853)

- [springBoot#server-properties](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html#server-properties)

- [Mozilla Http 请求头限制](https://developer.mozilla.org/zh-CN/docs/Glossary/%E8%AF%B7%E6%B1%82%E5%A4%B4)