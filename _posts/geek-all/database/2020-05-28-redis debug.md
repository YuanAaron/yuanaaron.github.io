---
layout: post 
author: oshacker
title: Redis Debug
category: database
tags: [database,redis]
excerpt: 用gdb/vscode调试redis源码
---


通过 gdb / vscode 调试 redis 源码，理解 redis 工作流程。

## gdb 常用命令

详细文档通过命令 `man gdb` 查看。

| 命令                | 操作                                                                       |
| ------------------- | -------------------------------------------------------------------------- |
| r                   | 运行调试                                                                   |
| n                   | 下一步                                                                     |
| c                   | 继续运行                                                                   |
| ctrl + c            | 中断信号                                                                   |
| c/continue          | 中断后继续运行                                                             |
| s                   | 进入一个函数                                                               |
| finish              | 退出函数                                                                   |
| l                   | 列出代码行                                                                 |
| b                   | 断点<br/>显示断点列表 info b<br/>删除断点 delete number<br/>清除断点 clear |
| n                   | 下一步                                                                     |
| until               | 跳至行号<br/>until <number>                                                |
| p                   | 打印<br/>打印数组信息 p *array@len<br/>p/x 按十六进制格式显示变量          |
| bt/backtrace        | 堆栈bt <-n><br/>-n表一个负整数，表示只打印栈底下n层的栈信息。              |
| f/frame             | 进入指定堆栈层<br/> f <number>                                             |
| thread apply all bt | 显示线程所有堆栈                                                           |
| attach              | 绑定进程调试<br/>attach <-p pid>                                           |
| detach              | 取消绑定调试进程                                                           |
| disassemble         | 看二进制数据<br/>disassemble <func>                                        |
| x                   | 查看内存                                                                   |
| focus               | 显示源码界面                                                               |
| display             | 显示变量                                                                   |
| info registers      | 查看寄存器                                                                 |

## 安装编译 redis

```shell
wget http://download.redis.io/releases/redis-3.2.8.tar.gz
tar xzf redis-3.2.8.tar.gz
cd redis-3.2.8
```

更新 Makefile，修改相应编译项

```shell
vim src/Makefile
```

```shell
# OPTIMIZATION?=-O2
OPTIMIZATION?=-O0
# REDIS_LD=$(QUIET_LINK)$(CC) $(FINAL_LDFLAGS)
REDIS_LD=$(QUIET_LINK)$(CC) $(FINAL_LDFLAGS) $(OPTIMIZATION)
```

编译

```shell
make clean; make
```

我按照本文原作者的上述构建命令操作，报如下错误：

![](https://www.coderap.cn/assets/images/2020/05/redis_error1.png)

解决办法：make clean; make MALLOC=libc

测试
```shell
# server
cp redis.conf src/
./src/redis-server redis.conf

# client
./src/redis-cli
set name zhangsan
```

## gdb 调试流程

| 步骤 | 命令                                          | 描述                                                    |
| ---- | --------------------------------------------- | ------------------------------------------------------- |
| 1    | sudo gdb --args ./src/redis-server redis.conf | 启动调试                                                |
| 2    | r                                             | 运行程序                                                |
| 3    | ctrl + c（键盘操作）                          | 中断程序                                                |
| 4    | b dict.c:dictAdd                              | 对应代码下断点                                          |
| 5    | c                                             | 继续运行程序                                            |
| 6    | redis-cli<br/>set k5 v5                       | 启动 client 连接redis-server测试（redis 默认端口 6379） |
| 7    | focus                                         | 进入源码窗口调试                                        |
| 补充1 | info win                                     |  查看当前的focus                                        |
| 补充2 | fs cmd                                        | 切换到cmd                                             |
| 8    | bt                                            | 程序堆栈(查看接口调用流程)                              |
| 9    | f 0                                           | 进入堆栈第 0 层                                         |
| 10   | n                                             | 单步调试                                                |

![](https://www.coderap.cn/assets/images/2020/05/redis1.png)

![](https://www.coderap.cn/assets/images/2020/05/redis2.png)

## vscode 调试流程

### 启动 vscode

因为 gdb 在 macOS 下需要 sudo 提升权限，vscode 配置貌似没有这个选项设置。所以只能用下面这个命令启动 vscode 项目

```shell
# redis 源码本地目录
cd ~/src/other/redis-3.2.8

#  vscode 打开 redis 源码目录
sudo code --user-data-dir="~/.vscode-root" .
```

### vscode 项目配置

* launch.json

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "gcc build and debug active file",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/src/redis-server",
            "args": [
                "redis.conf"
            ],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "preLaunchTask": "shell"
        }
    ]
}
```

* tasks.json

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "shell",
            "type": "shell",
            "command": "/usr/bin/make"
        }
    ]
}
```

### vscode 调试

![](https://www.coderap.cn/assets/images/2020/05/redis3.png)

## 参考

* [gdb 调试工具 --- 使用方法浅析](https://blog.csdn.net/men_wen/article/details/75220102)
* [Linux中gdb 查看core堆栈信息](https://blog.csdn.net/suxinpingtao51/article/details/12072559)
* [linux上用gdb调试redis源码](https://www.jianshu.com/p/692d1cd27e9b)

> 🔥文章来源：[wenfh2020.com](https://wenfh2020.com/)

