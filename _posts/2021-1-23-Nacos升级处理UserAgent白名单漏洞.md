---
layout:     post
title:      Nacos升级处理UserAgent白名单漏洞
subtitle:   
date:       2021-01-23
author:     高强
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - nacos
    - 微服务
    - 问题解决
---


### 部署内容
- Nacos 版本升级
- Nacos User-Agent 安全漏洞处理

### 执行步骤
- [x] 各个集群节点下载 1.4.1 release 包并解压
- [x] 同步 1.4.0 配置及部署脚本到各个集群节点
> 1. 同步 application.properties, cluster.conf
> 2. 修改 startup.sh 中 JVM 内存限制项(不能直接复制旧版本的 startup.sh)
> 3. 同步集群启动停止自定义脚本

- [x] 修改集群节点启动端口，并新增配置
```yml
# 关闭使用user-agent判断服务端请求并放行鉴权的功能
nacos.core.auth.enable.userAgentAuthWhite=true

# 配置自定义身份识别的key和value
nacos.core.auth.server.identity.key=xxx
nacos.core.auth.server.identity.value=xxx
```

- [x] 执行持久化变更
```sql
ALTER TABLE `config_info_tag`
MODIFY COLUMN `src_ip` varchar(50) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT 'source ip' AFTER `src_user`;

ALTER TABLE `his_config_info`
MODIFY COLUMN `src_ip` varchar(50) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL AFTER `src_user`;

ALTER TABLE `config_info`
MODIFY COLUMN `src_ip` varchar(50) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT 'source ip' AFTER `src_user`;

ALTER TABLE `config_info_beta`
MODIFY COLUMN `src_ip` varchar(50) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT 'source ip' AFTER `src_user`;
```

- [x] 启动新集群，跟踪启动日志检查服务是否正常
- [x] Openresty 配置负载均衡指向到新集群，检查注册的各个服务是否正常
- [x] 修改集群各节点配置，并重启集群，检查集群及服务运行情况
```yml
acos.core.auth.enable.userAgentAuthWhite=false
```
- [x] 关闭旧版本 Nacos 集群

### 回滚步骤
- Openresty 配置负载均衡指向到新集群，发现注册的各个服务不能正常使用
> 回滚：  
> 修改 Openresty 配置指向回旧版本 Nacos 集群

- 修改集群各节点配置，并重启集群，发现Nacos集群或注册到集群的服务不能正常使用
> 回滚：  
> 修改 Openresty 配置指向回旧版本 Nacos 集群，检查 Nacos 新版本 Auth 配置项

### 部署后
- [x] 更新服务器组件文档信息
- [x] 重新开放开发环境 nacos 外网访问域名(仅需要修改配置文件及调试服务问题时开放)

### 参考信息
- [Report a security vulnerability in nacos to bypass authentication](https://github.com/alibaba/nacos/issues/4593)
- [Report a security vulnerability in nacos to bypass authentication(identity) again](https://github.com/alibaba/nacos/issues/4701)
- [1.4.1 release](https://github.com/alibaba/nacos/releases/tag/1.4.1)