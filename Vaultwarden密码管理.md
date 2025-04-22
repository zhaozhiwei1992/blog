---
title: 本地密码管理器-Vaultwarden
date: 2025-03-10
updated: 2025-03-10
tags: [OpenManus, AI, DeepSeek]
---
# 前言

之前主要使用lastpass，目前是浏览器端免费，其它都要收费，而且密码存在别人服务器总是不放心。

所以我们要本地部署vaultwarden并且可以在各种Bitwarden正常使用。这里使用vaultwarden是因为相比原生更加轻量，占用资源极少。

# 部署

## 环境准备​​

​​服务器要求​​：腾讯云服务器（1核2GB内存，Ubuntu/Debian系统）。

​​开放端口​​：80（HTTP）、443（HTTPS），通过腾讯云控制台配置安全组

注:
如果家里本身就有公网ip，或者只是本地玩玩，是不需要公网服务器的。我这里公网服务器资源紧张，用来做转发，实际服务部署到内网机，所以公网和内网交互需要内网穿透，我这里使用的是zerotier。

## 内网机部署

为了方便从云服务器转发到内网，所以通过nginx包一层，给vaultwarden套一个前缀，比如:/bitwarden。否则云服务nginx转发会很乱

``` yaml
services:
  nginx-proxy:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "8000:80"    # 对外暴露 HTTPS 端口
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl   # 证书存储目录
    networks:
      - vaultwarden-net
    depends_on:
      - vaultwarden
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    networks:
      - vaultwarden-net
    volumes:
      - ./vw-data/:/data/
    expose:          # 仅内部暴露端口，nginx能访问即可
      - "80"
      - "3012"

# 自定义网络（实现容器间通信）
networks:
  vaultwarden-net:
    driver: bridge
```

### 挂载目录结构如下

``` bash
vaultwarden/
├── docker-compose.yaml
└── volumes
    ├── nginx
    │   ├── conf.d
    │   │   └── default.conf
    │   ├── ssl
    │   │   ├── certificate.crt
    │   │   └── private.key
    │   └── uwsgi_params
    └── vw-data
```

### conf.d/default.conf配置

这里nginx.conf会默认引入所有conf.d/\*.conf

``` bash
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    return 301 https://$host$request_uri;  # 301 永久重定向

}

# HTTPS 主配置
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name localhost;

    # SSL 证书配置（自签名示例，生产环境替换为 Let's Encrypt）
    #ssl_certificate /etc/nginx/ssl/cert.pem;
    #ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_certificate /etc/nginx/ssl/certificate.crt;
    ssl_certificate_key /etc/nginx/ssl/private.key;
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout 10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # 静态文件服务（可选）
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }

    # Vaultwarden 代理配置（核心修正）
    location /bitwarden/ {
        proxy_pass http://vaultwarden/;  # 末尾斜杠确保移除路径前缀
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 解决路径前缀导致的资源加载问题
        proxy_redirect off;
        sub_filter_once off;
        sub_filter_types *;
        sub_filter 'href="/' 'href="/bitwarden/';
        sub_filter 'src="/' 'src="/bitwarden/';
    }

    # WebSocket 支持（实时同步）, 自 v1.31.0 起，已移除对 3012 端口 WebSocket 流量的支持，因为它已集成至主 HTTP 端口。
    location /bitwarden/notifications/hub {
        proxy_pass http://vaultwarden:3012;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 错误页面配置（保持原样）
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

### 配置自签名证书

``` bash
# 在docker-compose同级目录执行
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ./volumes/nginx/ssl/private.key \
  -out ./volumes/nginx/ssl/certificate.crt
```

### 服务启动

``` bash
docker-compose up -d
```

### 测试访问

``` bash
https://10.241.189.234:8443/bitwarden
```

## 公网ECS配置nginx(无域名)

``` bash
cd /etc/nginx/conf.d

# 增加 234-proxy-443.conf, 这里一定要注意，把nginx.conf中server啥的都干掉，避免影响。

[root@VM-8-12-centos conf.d]# cat 234-proxy-443.conf
server {
    listen 443 ssl;
    server_name 替换为你的公网域名或IP;  # 替换为你的公网域名或IP

    # SSL 证书配置（必须使用有效证书，如 Let's Encrypt）
    # ssl_certificate /etc/letsencrypt/live/your-public-domain.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/your-public-domain.com/privkey.pem;
    ssl_certificate /etc/nginx/ssl/certificate.crt;
    ssl_certificate_key /etc/nginx/ssl/private.key;


    # 反向代理到内网服务
    location /bitwarden {
    proxy_pass https://10.241.189.234:8443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 信任内网服务的自签名证书（可选）
        proxy_ssl_verify off;
    }

    # 其他安全头配置
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
}
```

### nginx -s reload

### 测试

``` bash
https://ECS公网IP/bitwarden

# 这种方式配置后，在浏览器可以直接使用，包括插件。但是会提示证书不安全，手机端除非手动能安装自建证书，否则无法直接使用。
```

## 公网ECS配置nginx(有域名)

以腾讯ECS为例

### 域名配置

根据官方提示，将域名绑定到ECS机器

进行域名备案

### 调整nginx配置

使用免费或者付费的ssl证书，下载后替换nginx中配置的证书信息

``` bash
# SSL 证书配置（必须使用有效证书）
ssl_certificate /etc/nginx/ssl/自己的域名.crt;
ssl_certificate_key /etc/nginx/ssl/自己的域名.xyz.key;
```

## 客户端配置

1.  下载安装bitwarden相关客户端，包括app端，浏览器插件等
2.  配置为自托管，然后按照自己设置的用户密码登录即可

# 备份

找一个自己觉得安全的地方备份自己的挂载(volumes目录)，比如NAS或者网盘，U盘等

# 问题处理

## console-log.service.ts:53 Unhandled error in angular Error: Could not instantiate WebCryptoFunctionService. Could not locate Subtle crypto.

### 分析处理

必须使用https访问，否则报错

## 安卓客户端无法访问，提示无法验证服务器证书

### 分析处理

这个真得有域名了。
