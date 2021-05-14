---
layout:     post
title:      nacos server 集群CPU占用过高Full GC频繁
subtitle:   
date:       2020-11-19
author:     高强
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - nacos
    - 微服务
    - 问题解决
---

### top 发现两个进程占用 CPU 太高，导致整体 CPU 占用达 44.8%(4u)
```
%Cpu(s): 44.8 us,  9.1 sy,  0.0 ni, 42.7 id,  0.0 wa,  0.0 hi,  3.5 si,  0.0 st
PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
19278 djb       20   0 4482224 912904  15220 S 149.2  2.8  30926:49 java
19316 djb       20   0 4467880 893276  14244 S  30.6  2.8   7668:36 java
```

### ps, jps -lvm
```
19278 /home/djb/local/nacos8848/target/nacos-server.jar --spring.config.location=classpath:/,classpath:/config/,file:./,file:./config/,file:/home/djb/local/nacos8848/conf/ --logging.config=/home/djb/local/nacos8848/conf/nacos-logback.xml --server.max-http-header-size=524288 nacos.nacos -Xms256m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/djb/local/nacos8848/logs/java_heapdump.hprof -XX:-UseLargePages -Dnacos.member.list= -Djava.ext.dirs=/home/djb/local/jdk8/jre/lib/ext:/home/djb/local/jdk8/lib/ext -Xloggc:/home/djb/local/nacos8848/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/home/djb/local/nacos8848/plugins/health,/home/djb/local/nacos8848/plugins/cmdb,/home/djb/local/nacos8848/plugins/mysql -Dnacos.home=/home/djb/local/nacos8848

19316 /home/djb/local/nacos8849/target/nacos-server.jar --spring.config.location=classpath:/,classpath:/config/,file:./,file:./config/,file:/home/djb/local/nacos8849/conf/ --logging.config=/home/djb/local/nacos8849/conf/nacos-logback.xml --server.max-http-header-size=524288 nacos.nacos -Xms256m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/djb/local/nacos8849/logs/java_heapdump.hprof -XX:-UseLargePages -Dnacos.member.list= -Djava.ext.dirs=/home/djb/local/jdk8/jre/lib/ext:/home/djb/local/jdk8/lib/ext -Xloggc:/home/djb/local/nacos8849/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/home/djb/local/nacos8849/plugins/health,/home/djb/local/nacos8849/plugins/cmdb,/home/djb/local/nacos8849/plugins/mysql -Dnacos.home=/home/djb/local/nacos8849
```

### -Xloggc:/home/djb/local/nacos8848/logs/nacos_gc.log
tail -f 发现一直在 Full GC

### jmap -heap
```
Heap Usage:
PS Young Generation
Eden Space:
   capacity = 208666624 (199.0MB)
   used     = 40606024 (38.72492218017578MB)
   free     = 168060600 (160.27507781982422MB)
   19.45975988953557% used
From Space:
   capacity = 28835840 (27.5MB)
   used     = 0 (0.0MB)
   free     = 28835840 (27.5MB)
   0.0% used
To Space:
   capacity = 30932992 (29.5MB)
   used     = 0 (0.0MB)
   free     = 30932992 (29.5MB)
   0.0% used
PS Old Generation
   capacity = 268435456 (256.0MB)
   used     = 267923256 (255.51152801513672MB)
   free     = 512200 (0.48847198486328125MB)
   99.80919063091278% used

32879 interned Strings occupying 3955648 bytes.
```

### jstat -gcutil 19278
```
S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   0.00  44.55  99.81  95.86  93.29 287952 1154.402 2162204 186972.463 188126.865
```

### arthas 打印占用 CPU 最高的三个线程
```
[arthas@19278]$ thread -n 3
"http-nio-8848-Acceptor-0" Id=106 cpuUsage=4% RUNNABLE
    at sun.nio.ch.ServerSocketChannelImpl.accept0(Native Method)
    at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:422)
    at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:250)
    at org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept(NioEndpoint.java:448)
    at org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept(NioEndpoint.java:70)
    at org.apache.tomcat.util.net.Acceptor.run(Acceptor.java:95)
    at java.lang.Thread.run(Thread.java:748)


"http-nio-8848-ClientPoller-0" Id=104 cpuUsage=2% RUNNABLE (in native)
    at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
    at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
    at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
    at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
    at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
    at org.apache.tomcat.util.net.NioEndpoint$Poller.run(NioEndpoint.java:744)
    at java.lang.Thread.run(Thread.java:748)


"http-nio-8848-ClientPoller-1" Id=105 cpuUsage=2% RUNNABLE (in native)
    at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
    at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
    at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
    at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
    at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
    at org.apache.tomcat.util.net.NioEndpoint$Poller.run(NioEndpoint.java:744)
    at java.lang.Thread.run(Thread.java:748)

Affect(row-cnt:0) cost in 111 ms.
```

### 解决
- 官方描述中这两个配置的单位是错的，实际单位应为毫秒
> heart-beat-interval: 3000  
> heart-beat-timeout: 15000

- 处理后
```
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
  654 djb       20   0 5073376   1.1g  12724 S   7.3  3.5  11663:15 java
30584 djb       20   0 4853772   1.3g  11744 S   3.3  4.3 751:41.68 java
```

### 参考
- https://github.com/alibaba/spring-cloud-alibaba/issues/1877
- https://github.com/alibaba/nacos/issues/2064