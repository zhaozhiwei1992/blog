---
title: 自定义swagger扩展解析jsondoc
date: 2022-10-30
tags: [java, swagger, javadoc]
---

# 需求规定

为了减少注释和swagger注解的重复定义, 通过规范注释,
让swagger可以通过javadoc来产生

替换@Api、@ApiOperation、@ApiModel、@ApiModelProperties等注解

只是对swagger的扩展，如果有swagger注解，以注解为准

# 运行环境

## springboot2.1.7

## jdk1.8

# 设计思想

## 系统构思

1.  编译完成的class里没有注释的，所以注释信息只有在编译代码时存储起来
2.  swagger本身是通过注解实现接口定义描述等加载的，现将代码注释生成json格式,
    利用swagger扩展在启动项目时通过json进行加载到swagger中
3.  需要配合自定义的javadoc-json-maven-plugin先将注释生成json文件

## 关键技术与算法

### 生成javadoc.json文件

com.example.CommentToJsonMain(已做成maven插件, 这里原始文件可做测试)

插件: <https://github.com/zhaozhiwei1992/javadoc-json-maven-plugin>

### swagger扩展代码

com.example.SpringbootSwaggerJavadocApplication启动即可生效

类定义: com.example.plugin.CommentApiBuilder

方法定义: com.example.plugin.CommentOperationBuilder

### 类代码注释规范

``` example
/**
 * @Title: PersonController
 * @Package: com/example/springbootcache/controller/PersonController.java
 * @Description: 用户信息接口
 * @author: zhaozhiwei
 * @date: 2022/10/25 下午8:23
 * @version: V1.0
 */

```

### 方法代码注释规范

``` example
/**
 * @date: 2022/10/25-上午10:19
 * @author: zhaozhiwei
 * @method: findByID
  * @param id : 唯一id
 * @return: com.lx.demo.springbootcache.domain.Person
 * @Description: 根据id获取用户信息
 * 获取十次， 只有第一次是读库，后续都是取缓存
 * 直接删掉redis缓存里的内容，仍然可以获取数据，并且走缓存，此时获取的是服务缓存ehcache中的信息
 * seq 10 |xargs -i curl -XGET 'http://localhost:8080/persons/2'
 */

```

# 基本处理流程

## 系统流程图

``` {.plantuml file="/tmp/xx01.png" cmdline="-charset UTF-8"}
start
:引入javadoc-json插件;
:使用插件:generate goals生成json版注释;
:正常启动web服务;
:访问swagger-ui.html;
note right
        此时即可看到没有注解的方法也可以显示方法描述信息
end note
end
```

# 代码

<https://github.com/zhaozhiwei1992/demo/tree/master/springboot/springboot-swagger-javadoc>

# 参考

## swagger扩展

<https://github.com/hadix-lin/springfox-plus>

<https://blog.csdn.net/ydonghao2/article/details/109593416>

<https://blog.csdn.net/baiihcy/article/details/53861267>

<https://blog.csdn.net/qq_17623363/article/details/109259315>
