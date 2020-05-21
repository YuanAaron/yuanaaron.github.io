---
layout: post 
author: oshacker
title: Linux下追查线上问题常用命令
category: linux
tags: [linux]
excerpt: linux查用命令集
---

## 查找占用CPU最多的10个进程

方法一：ps -aux | sort -k4nr | head -10
+ -a(all): 所有的进程
+ -u(userid): 执行该进程的用户id
+ -x: 指代显示所有程序，不以终端机来区分
+ -k: 根据哪一个关键词排序（-k4表示按照第四列排序，即%MEM;-k3表示按照第三列排序，即%CPU）
+ -n(numberic sort): 根据其数值排序
+ -r(reverse): 反向比较结果，输出时默认从小到大，反向后从大到小

其中，ps -aux的输出格式如下：
> USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND

方法二：top（然后按下P，注意大写）

## 查找某个进程的服务名

ps -aux | grep PID

## 查找占用内存最多的10个进程

方法一：ps -aux | sort -k3nr | head -10

方法二：top（然后按下M，注意大写）

## 查看某个端口的占用情况

方法一：lsof -i:端口号

方法二：netstat -tunlp | grep 端口号
+ -t（tcp): 显示tcp相关选项
+ -u（udp): 显示udp相关选项
+ -n: 拒绝显示别名，能显示数字的全部转化为数字
+ -l: 列出在Listen(监听)的服务状态
+ -p: 显示建立相关链接的程序名

## 杀掉对应的进程

kill -9 PID

