---
title: "Dify0.15.1升级1.4.3版本"
date: "2025-06-18"
updated: "2025-06-18"
tags: [AI,Dify]
categories: AI
---

# 安装1.4.3

<https://codeload.github.com/langgenius/dify/tar.gz/refs/tags/1.4.3>

下载后在原dify-0.15.1同级目录解压

## 启动

cd dify-1.4.3/docker

cp .env.example .env \# 公司环境需要调整nginx expose端口

docker-compose up -d \# 等待启动完成，正常访问页面即可

# 压缩原挂载目录

cd dify-0.15.1/docker

tar -zcvf volumes.tar.gz volumes

# 用0.15.1的挂载目录覆盖1.4.3的挂载

``` example
1. 备份1.4.3挂载目录
tar -zcvf volumes143.tar.gz volumes

2. 删除挂载目录
rm -rf volumes

3. 用0.15.1的挂载覆盖当前挂载目录，如果是开发本地使用，可以用官方提供的方式测试。
cp dify-0.15.1/docker/volumes.tar.gz .

tar -zxvf volumes.tar.gz

4. 启动服务访问测试
docker-compose up -d

```

# 启动后操作

1.  原项目中一些llm，embedding等都没有配置，需要自己在界面增加供应商再次配置即可，配置后重新刷新页面。

# 备注

1.  如果是内网环境，docker需要在公网环境下载好后，通过load方式加载到内网，对于插件同理。

# 参考

<https://docs.dify.ai/zh-hans/development/migration/migrate-to-v1>
