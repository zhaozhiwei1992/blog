---
title: 迁移polardb问题一
date: 2021-05-18
updated: 2021-05-18
categories: 软件, 数据库
tags: polardb, mybatis
---

# 环境

## polardb版本

``` {.example}
<dependency>
  <groupId>com.aliyun</groupId>
  <artifactId>polardb-jdbc18</artifactId>
  <version>1.0.0</version>
  <scope>system</scope>
  <systemPath>${lib.dir}/polardb-jdbc18.jar</systemPath>
</dependency>
```

## mybatis版本

**直接使用springboot引入**

``` {.example}
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>2.1.4</version>
</dependency>
```

# 问题描述

``` {.example}
javax.servlet.ServletException: org.springframework.dao.DataIntegrityViolationException: 
### The error may exist in com/xx/templatedef/dao/oracle/XXTemplateDef_SqlMap.xml
### The error may involve com.xx.templatedef.dao.XXTemplateDefDao.selectTemplateDefByCategoryAndState-Inline
### The error occurred while setting parameters
### SQL: select * from act_xx_tem_def where category=? and deploy_state=1
### Cause: com.aliyun.polardb.util.PSQLException: 不良的类型值 long : 
com.aliyun.polardb.util.PSQLException: canot convert the column of type BYTEA to requested type long;
```

## 原因

mybatis查询act_tem_def表涉及到blob字段(content_bytes)，
默认的BlobTypeHandler的getBlob方法不适用, 需要重写handler

# 解决方案

1.  在业务下重写BlobTypeHandler, 获取getBinaryStream然后转换byte数组
2.  mapper.xml -\> resultmap中, 加入下述转换

``` {.example}
<result column="CONTENT_BYTES" property="contentBytes" jdbcType="BLOB"
  typeHandler="com.example.springbootpolardb.handler.PolarDbBlobTypeHandler"/>
```

## 转换逻辑

``` {.example}
public static byte[] toByteArray(InputStream input) throws IOException {
    ByteArrayOutputStream output = new ByteArrayOutputStream();
    byte[] buffer = new byte[4096];
    int n = 0;
    while (-1 != (n = input.read(buffer))) {
        output.write(buffer, 0, n);
    }
    return output.toByteArray();
}
```

# 代码

-   [springboot-polardb](https://github.com/zhaozhiwei1992/demo/tree/master/springboot/springboot-polardb)
