---
title: atomikos实现分布式事务
date: 2022-04-25
categories: [java, 编程]
tags: [分布式事务]
---

# 概述

多数据源单服务写入, 分布式事务实现

使用随机数控制产生异常

**注: 网上很多都是只有多数据源配置，实际不能控制事务统一回滚,
单服务场景下如果多个数据源只有一个写，剩下都是读, 则不需要分布式事务**

为减少篇幅，详细代码在代码仓库，可自行参考

# 版本

springboot 2.1.7.RELEASE

# 配置引入

``` example
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-jta-atomikos</artifactId>
</dependency>
```

# 增加配置类

AtomikosJtaPlatform

JPAAtomikosTransactionConfig Atomikos事务配置类

PrimaryConfig 主数据源配置

SecondaryConfig 其它数据源配置

# 数据对象实现

User

# 增加仓储实现

**需要分别实现primary和secondary, 从而支持多个数据源访问**

PrimaryUserRepository

SecondaryUserRepository

# 实现服务类

PrimaryUserService

SecondaryUserService

## ExampleService

负责统一调用 PrimaryUserService 和 SecondaryUserService,
模拟一个服务同时访问多个服务类(**每个服务类引用了不同数据源**)

如果其中某个服务抛出异常，则整个事务回滚, \@Transactional(rollbackOn =
Exception.class) 是必须的

# 测试

com.example.springbootatomikos.services.UserServiceTest#testSave

# 代码

[github地址](https://github.com/zhaozhiwei1992/demo/tree/master/springboot/springboot-atomikos)

[gitee地址](https://gitee.com/zhaozhiwei_1992/demo/tree/master/springboot/springboot-atomikos)
