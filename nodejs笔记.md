---
title: linux下常用工具
date: 2019-01-13 17:50:40
tags: linux
---
# nodejs项目初始化
  1. 创建目录 mkdir nodejs & cd nodejs

  2. npm init (根据提示，或者一路回车即可)

  3. 创建server.js文件,并写入以下内容
     ``` javascript
     // 内置组件
      var http=require("http");
     
      http.createServer(function(request, response){
          // 发送 HTTP 头部 
          // HTTP 状态值: 200 : OK
          // 内容类型: text/plain
          response.writeHead(200, {"Content-Type":"text/plain"});
     
          //响应数据
          response.end("hello nodejs");
      }).listen(8888);
      //服务启动成功后输出日志
      console.log('Server running at http://127.0.0.1:8888/'); 
     ```

  4. node server.js 启动成功

  5. 浏览器访问 127.0.0.1:8888

     
# npm使用(root)



## 搜索
+ npm search <mould>

  

## 更新
+ npm install -g npm

+ npm update -g npm

  

## 安装卸载模块
   + 当前目录
     npm install <mould>
     npm uninstall <mould>

   + 全局
     npm install -g <mould>
     npm uninstall -g <mould>

     
## 查看模块安装信息
   + 一个
     npm list -g

   + 所有
     npm list <mould>

     
## Package.json属性说明
   ```
    name - 包名。
    version - 包的版本号。
    description - 包的描述。
    homepage - 包的官网 url 。
    author - 包的作者姓名。
    contributors - 包的其他贡献者姓名。
    dependencies - 依赖包列表。如果依赖包没有安装，npm 会自动将依赖包安装在 node_module 目录下。
    repository - 包代码存放的地方的类型，可以是 git 或 svn，git 可在 Github 上。
    main - main 字段指定了程序的主入口文件，require('moduleName') 就会加载这个文件。这个字段的默认值是模块根目录下面的 index.js。
    keywords - 关键字
   ```


## whatis npm

   1. npm就是js下的maven, 支持包依赖， 执行等
   2. package.json == pom.xml



# var 和 const



# 初始化buffer

  + 类似于初始化一个list
  + Buffer.alloc(5)



# nodejs对标的是其他的后端语言类似java



# require的执行流程

1.  如果 X 是内置模块
   1.  返回内置模块
   2.  停止执行
2. 如果 X 以 '/' 开头
   1. 设置 Y 为文件根路径
3. 如果 X 以 './' 或 '/' or '../' 开头
   1. LOAD_AS_FILE(Y + X)
   2. LOAD_AS_DIRECTORY(Y + X)
4. LOAD_NODE_MODULES(X, dirname(Y))
5. 抛出异常 "not found"

# extend

Sub 仅仅继承了Base 在原型中定义的函数，而构造函数内部创造的 base 属 性和 sayHello 函数都没有被 Sub 继承。
