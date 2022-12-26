---
title: java注释转json插件开发实战
date: 2022-10-27
categories: [java, 编程]
tags: [maven插件, javadoc]
---

# 目的

将java的代码注释转换为json格式，并写入文件

本文介绍了完整的开发流程及如何使用

# 运行环境

## jdk1.8

## maven3.x

# 设计思想

## 系统构思

1.  编译完成的class里没有注释的，所以注释信息只有在编译代码时存储起来
2.  将能够生成javadoc.json的代码做成maven插件

## 关键技术与算法

### 需要实现Doclet

必须引入下述jar包, 来导入com.sun.javadoc.Doclet

``` example
<dependency>
    <groupId>com.sun</groupId>
    <artifactId>tools</artifactId>
    <version>1.8</version>
    <scope>system</scope>
    <systemPath>/home/zhaozhiwei/applications/jdk1.8.0_181/lib/tools.jar</systemPath>
</dependency>

```

### 代码路径

<https:github.com:zhaozhiwei1992/javadoc-json-maven-plugin>

入口: com.example.Javadoc2JsonMojo#execute

### 生成javadoc.json文件

com.example.CommentToJson(做成maven插件使用)

```
{
  "com.example.web.rest.PersonResource.findByID(java.lang.Long)#date": "2022/10/25-上午10:19",
  "com.example.web.rest.PersonResource#Package": "com/example/springbootcache/controller/PersonController.java",
  "com.example.web.rest.PersonResource.findByID(java.lang.Long)#Description": "根据id获取用户信息",
  "com.example.web.rest.PersonResource.save(Person)#method": "save",
  "com.example.web.rest.PersonResource#author": "zhaozhiwei",
  "com.example.web.rest.PersonResource.deleteByID(java.lang.Long)#Description": "根据id删除person信息",
  "com.example.web.rest.PersonResource#Title": "PersonController",
  "com.example.web.rest.PersonResource.update(Person)#date": "2022/10/25-上午10:21",
  "com.example.web.rest.PersonResource.save(Person)#Description": "保存方法",
  "com.example.web.rest.PersonResource#Description": "用户信息接口",
  "com.example.web.rest.PersonResource#date": "2022/10/25 下午8:23",
  "com.example.web.rest.PersonResource.update(Person)#Description": "修改内容看是否会调整缓存",
  ... 省略了一大堆
}

```

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

### 参数定义

``` example
@parameter额外属性：

@parameter alias="<aliasName>":为mojo参数使用别名

@parameter expression="${aSystemProperty}":使用系统属性表达式对mojo参数进行赋值

@parameter defaultValue="aValue/${anExpression}":提供一个默认值
```

# 开发流程

## 项目构建

``` example
mvn archetype:generate -DgroupId=com.example -DartifactId=javadoc-json-maven-plugin -DarchetypeArtifactId=maven-archetype-mojo -DinteractiveMode=false
```

## 使用插件

### Install

在javadoc-json-maven-plugin项目中执行maven的install命令,
将插件安装到本地仓库

### 引入自定义插件

在其它项目引入插件测试

展开插件可以看到Mojo列表，由于项目里只有一个Mojo，且Mojo上用注释@goal指定了名称为generate，所以这里只能看到一个javadoc-json:generate

``` example
            <plugin>
                <groupId>com.example</groupId>
                <artifactId>javadoc-json-maven-plugin</artifactId>
                <version>2.0-SNAPSHOT</version>
                <configuration>
<!--                    指定java注释扫描目录-->
                    <basePackage>com/example/web/rest</basePackage>
                </configuration>
            </plugin>

```

### 执行插件

双击javadoc-json:generate就会执行这个插件。

执行完毕后，可以看到项目根目录的target文件下多了个javadoc.json文件。

### 命令行执行插件(可选)

命令格式为mvn groupId:artifactId:version:goal，执行如下命令：

mvn com.<example:javadoc-json-maven-plugin:1.0.0-SNAPSHOT:generate>

由于我们的插件命名符合规范，所以上面的命令可以简写为：

mvn javadoc-json:generate

# 尚未解决的问题

javadoc插件去掉异常提示

# 参考

## maven插件开发

<https://www.jianshu.com/p/9cfe599b3c5e>

## 参数定义

<https://blog.csdn.net/z69183787/article/details/52984622>
