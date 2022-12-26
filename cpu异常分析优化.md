---
title: cpu异常分析优化
date: 2022-03-14 
tags: [CPU占用, 并发]
categories: [CPU占用, 并发]
---

# cpu占用飙升

## 现象

系统访问很慢

## 排查

top 获取进程pid

top -Hp pid, 查看线程情况,获取线程id

print \"%x\" \"线程id\" #输出16进制线程id

jstack 进程pid \> /tmp/jstack.txt

在jstack.txt中搜索上述16进制线程id, 紧跟着就是问题原因

jmap

查看最耗费cpu的线程堆栈信息

cat stack \|grep -i 34670 -C10 --color

# 备注

并发场景下使用currenthashmap替代hashmap, 避免死循环
