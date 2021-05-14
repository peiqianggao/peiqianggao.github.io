---
layout:     post
title:      Websocket Session 重连逻辑导致内存泄漏
subtitle:   
date:       2021-04-03
author:     高强
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - websocket
    - 内存泄漏
    - 问题解决
---

### 项目
> 使用 socket 同步交易所交易及行情数据

### 描述
生产环境每个同步区块链交易所数据的服务运行一段时间后发现内存占用一直升高，程序运行缓慢，最后达到 -Xmx 内存最大限制后自动关闭(OOM)

开发人员只能 kill -9 杀进程，再重启服务，中间可能丢失数据
### 问题产生
缓存与各大交易所 websocket 信息的 sessionMap(sessionId, Session) 一直增加但没有被正常清理，每次 socket 断开先尝试重连，重连成功就清除 sessionMap 中旧 session 信息，每个 session 缓存的信息达到 1M 甚至 25M，重连失败后一直无法清除，后续有定时任务重连成功导致 sessionMap 中的信息一直递增

### 调试
跟踪逻辑发现 sessionMap 中 remove 的调用仅在 websocket#onClose 方法上有使用到，但 onClose 逻辑中先重连，重连成功才从 sessionMap 中 remove 旧 session

查看 jvm 内存信息发现 -Xmx 限制 10240M，整体已使用内存已达到99%
使用 jstat -gcutil 12026 1000 10 发现 Full GC 几乎达到两秒一次

使用 jmap -dump jmap -dump:file=jvm_dump_sync-exchange-data-server_12026.bin 12026 转储堆快照

大对象分析
![大对象分析](/img/article/0004.png)

### 问题解决
- onClose 方法执行时先删除旧 sessionId 缓存
- 其他方案：考虑使用 WeakHashMap 代替 HashMap

### 参考与工具
- top 查看进程占用内存
- jstat 查 GC 情况，jmap dump 堆快照
- arthas
> 查看JVM具体占用
> jad 查问题代码逻辑是否为最新版本
> trace, monitor, watch 跟踪方法执行情况