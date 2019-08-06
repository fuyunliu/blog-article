---
title: Screen 用法
date: 2018-03-13 17:36:52
categories:
- 笔记
tags:
- Linux
---

GNU Screen是一款由GNU计划开发的用于命令行终端切换的自由软件。用户可以通过该软件同时连接多个本地或远程的命令行会话，并在其间自由切换。

<!-- more -->

<!-- toc -->

## 安装 `screen`

```sh
yum install -y screen
```

## `screen` 常用命令

新建一个Screen Session

```sh
screen -S session_name
```

将当前Screen Session放到后台

```sh
CTRL + A + D
```

唤起一个Screen Session

```sh
screen -r session_name
```

分享一个Screen Session

```sh
screen -x session_name
```

终止一个Screen Session

```sh
exit
or
CTRL + D
```

默认显示一屏的内容，要查看之前内容，如下操作：

```sh
Ctrl + A ESC
```

列表所有的会话

```sh
screen -ls
```

进入某个会话

```sh
screen -r session_name
```

如果进不去，则

```sh
screen -d session_name
```

再

```sh
screen -r session_name
```

`ctrl + A + N` 切换窗口
