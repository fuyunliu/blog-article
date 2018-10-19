---
title: CentOS 编译安装 Python3
date: 2018-03-12 08:33:42
categories:
- 笔记
tags:
- Python3
- CentOS
---

记录一下 CentOS 编译安装 Python3 的过程。

<!-- more -->

<!-- toc -->

#### 安装 `openssl`
```sh
yum install -y openssl-static
```

#### 安装 `gcc`
```sh
yum install -y gcc
```

#### 安装 `sqlite-devel`
少了这步的话，python3的一些库如twisted使用不正常，会提示缺少_sqlit3。
```sh
yum install -y sqlite-devel
```

#### 下载 `python3` 包
```sh
wget https://www.python.org/ftp/python/3.6.4/Python-3.6.4.tgz
```

#### 解压
```sh
tar -zxvf Python-3.6.4.tgz
```

#### 配置
```sh
cd Python-3.6.4
./configure --prefix=/usr/local/python3 --enable-loadable-sqlite-extensions --enable-optimizations
```

#### 编译安装
```sh
make && make install
```

#### 添加软连接
```
ln -s /usr/local/python3/bin/python3 /usr/bin/python3
```


#### 后记
检查是否安装成功
```sh
python3 --version
```
`yum` 搜索可用包
```sh
yum search python3 | grep devel
```
