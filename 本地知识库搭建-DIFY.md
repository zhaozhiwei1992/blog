---
title: 搭建个人AI知识库-DIFY
date: 2025-1-21
tags: [AI, 大语言模型, 知识库, DIFY]
categories: [AI，软件，工具]
---

# 前提
本地目前没有显卡，只能用cpu刚。

如果不想自己搭建本地模型，完全可以掏钱使用现成的API即可。

需要了解一些docker知识
# 搭建本地模型
## 环境
### os: archlinux
### 内存: 32g
### cpu: 6核12线程
### docker: 27.3.1
### docker-compose: 2.32.4
## ollama
```
pacman -S ollama

systemctl start ollama.service

 # 通过下述url判断ollama是否安装成功
http://127.0.0.1:11434/
```
## LLM模型 (qwen2:1.5b)
### 下载
```
ollama pull qwen2:1.5b
```
### 启动
```
ollama run qwen2:1.5b
```
### 测试
```
ollama run qwen2:1.5b
>>> who are you?
I am an AI language model, designed to answer questions and provide information on various topics. How can I assist you today?

>>> Send a message (/? for help)
```
## Text Embedding模型 (m3e)
### 下载
```
ollama pull milkey/m3e

embedding模型不需要run, ollama服务启动可直接使用
```

### 测试
```
curl http://127.0.0.1:11434/api/embed -d '{
  "model": "milkey/m3e",
  "input": "balabalabala"
}' | jq .
```
## 查看模型运行情况
```
ollama ps

NAME                        ID              SIZE      PROCESSOR    UNTIL    
qwen2:1.5b                  f6daf2b25194    1.5 GB    100% CPU     4 minutes from now
milkey/m3e:latest           1477f12451b0    860 MB    100% CPU     4 minutes from now
```
# 构建知识库(ollama+DIFY)
## 下载启动dify
参考官方文档，so easy!

https://docs.dify.ai/zh-hans/getting-started/install-self-hosted/docker-compose

本地采用的是 #systemd方式* 部署。这里一定要注意，不同的部署方式网络配置有点区别，比如systemd的方式服务启动需要增加环境变量OLLAMA_HOST，而对于docker启动方式，可以参考官方文档(暂未测试)
## 添加模型
这里我们需要两个模型，一个LLM，一个Text Embedding

### qwen2:1.5b模型添加

模型名称：qwen2:1.5b（必须完整填写）

基础 URL：http://<your-ollama-endpoint-domain>:11434 (这里的ip要是你本地ip,不能用localhost，127这些。本地是http://10.10.15.159:11434)

模型类型：对话

模型上下文长度：4096 (模型的最大上下文长度，若不清楚可填写默认值 4096)

最大 token 上限：4096 (模型返回内容的最大 token 数量，若模型无特别说明，则可与模型上下文长度保持一致)

是否支持 Vision：是

保存即可使用

![screenshot_x3ym8s_000.png](https://s2.loli.net/2025/01/21/XtlN3fowajFcrY2.png)

### milkey/m3e:latest
模型名称：milkey/m3e:latest (同上)

基础 URL：http://<your-ollama-endpoint-domain>:11434 (这里的ip要是你本地ip,不能用localhost，127这些。本地是http://10.10.15.159:11434)

模型上下文长度：4096

保存即可使用
### 访问测试

## 创建知识库
导入数据 --> 设置分段 -->设置索引及检索

对于word这种格式化的数据,分段模式最好使用 #父子模式* ; 索引方式使用高质量模式，使用上述m3e模型来生成索引数据。

![screenshot_zSwzzS_000.png](https://s2.loli.net/2025/01/21/MxvKOBUuGg1EJC8.png)
# 创建聊天助手
知识库是没法直接去使用的，顶多能做个召回测试。这里我们创建一个聊天助手，可以关联知识库，这样能真正使用。

聊天助手创建很简单，可以选择顶部工作室 -> 创建空白应用

![screenshot_UWVTYt_000.png](https://s2.loli.net/2025/01/21/D1xUA3iM4tLTchw.png)

选择已经创建好的知识库

![screenshot_NnXWmT_000.png](https://s2.loli.net/2025/01/21/hFE8kuyQ7qBNs52.png)

应用发布即可
## 直接通过dify使用
dify默认启动使用的是80端口，可以直接通过http://localhost 访问(首次使用需要注册用户密码)。 然后选择探索，选择我们刚刚创建的聊天助手就可以开始愉快的聊天了。

![screenshot_vq0gwR_000.png](https://s2.loli.net/2025/01/21/lnJCkKOuqLHadoE.png)

![screenshot_dhJZif_000.png](https://s2.loli.net/2025/01/21/mCRV8dXSBoZTJbY.png)
## 将dify嵌入到自己的应用中
可以通过api、iframe之类的方式将自己搭建的聊天助手嵌入到系统中(需要有开发能力，很简单)。
##  知识库工作流
![screenshot_pMpzgp_000.png](https://s2.loli.net/2025/01/21/O6FTmeAgRG5brck.png)
# 问题处理
## dify访问时提示11434拒绝
http://10.10.15.159:11434/ 请求失败
### 处理

检查服务启动正常，需要在service中增加环境变量 Environment="OLLAMA_HOST=0.0.0.0:11434"

```
sudo vim /usr/lib/systemd/system/ollama.service
systemctl daemon-reload
systemctl restart ollama.service
```

其它系统类似，就是让服务启动读取到该变量即可

## Reached maximum retries (3) for URL http://localhost:8090/api/system/ext/examples/echo 
### 分析处理
添加工具后，访问本地接口提示上述错误, 其实还是ip的问题，这里使用本地ip,如10.10.15.159。因为dify部署在容器中,localhost有特殊意义
