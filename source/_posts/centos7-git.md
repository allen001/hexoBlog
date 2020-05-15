---
layout: post
title: Centos7安装Git
date: 2020-05-14 10:52:31
toc: true
tags: 
- git安装
---

## 1. 从github下载最新版本

地址： https://github.com/git/git/releases 

选择最新的`tar.gz`包

![Git源码包](/image/centos-git/1587025342450.png) 

## 2. 通过Ftp上传服务器

ftp 上传

<!-- more -->

## 3. 安装必要依赖包

```bash
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel gcc perl-ExtUtils-MakeMaker
```

## 4. 解压&编译

```bash
# 解压
tar -zxvf git-2.26.1.tar.gz
cd git-2.26.1

#
make prefix=/usr/local/git all
make prefix=/usr/local/git install
```

## 5. 配置环境变量

```bash
# 打开文件
vim /etc/profile

# 底部加上git配置
PATH=$PATH:/usr/local/git/bin
export PATH

# 刷新环境变量
source /etc/profile
```

![环境变量](/image/centos-git/1587025280990.png)

## 6. 查看git版本

```bash
git --version
```

![git版本](/image/centos-git/1587025243394.png)