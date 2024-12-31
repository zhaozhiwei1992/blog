---
title: 好物推荐-telegram消息提醒
date: 2024-12-31
tags: [telegram, bot]
categories: [软件，工具]
---

# 什么是telegram
Telegram Messenger, commonly known as Telegram, is a cloud-based, cross-platform, social media and instant messaging (IM) service

Telegram Messenger，通常简称为Telegram，是一款基于云的、跨平台的社交媒体和即时通讯(IM)服务。
# 目标
日常生活中有一些动态的提醒，比如说服务器故障，或者找到了一些有意思的内容，我们需要有一种手段可以直接通知。理想情况下，如果能直接通知到微信会更好，但目前似乎没有既安全又好用的方法来实现这一点。。虽然钉钉提供了机器人的方式(需要有群组)，但是除了工作800年不会打开。所以借这次机会正好玩下telegram的机器人。
# 环境
## python3.x
## 科学上网
# 操作步骤
## 创建一个机器人
在botfather中输入/newbot, 按照提示创建一个机器人。创建后会返回一段内容包括机器人的tokenid。
## 获取要通知用户的chat_id
搜索添加机器人 @get_id_bot，私聊发送命令 /my_id，如下图，会返回你的id信息
## 代码实现
``` python
# -*- coding: utf8 -*-
from datetime import date, datetime
import requests
from telegram import Bot
from telegram.utils.request import Request
import os
...
def send_telegram_message(msg):
    # 替换为你的API令牌
    token = os.environ['TELEGRAM_TOKEN']
    # 替换为你的Telegram账号的ID
    chat_id = os.environ['TELEGRAM_CHAT_ID']
    """
    python 3.x
    """
    proxy = Request(proxy_url='http://127.0.0.1:7890')
    bot = Bot(token=token, request=proxy)
    bot.send_message(chat_id=chat_id, text=msg)

```
按照上述步骤即可实现将msg信息发送给机器人。所以理论上你可以创建很多个机器人，分别做不同类型的通知，enjoy！
