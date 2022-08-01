---
title: 自制 C 编译器 - 00 - 开发环境搭建
date: 2022-08-01 12:00:00
tags: 
- chibicc
- compiler
- Docker
categories: 
- 自制 C 编译器
toc: true
---

## 下载 chibicc 源码

```shell
$ cd ~/Code
$ git clone https://github.com/rui314/chibicc.git
```

## 拉取 Docker 镜像

选择 ubuntu:18.04 镜像

```shell
$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
Digest: sha256:478caf1bec1afd54a58435ec681c8755883b7eb843a8630091890130b15a79af
Status: Downloaded newer image for ubuntu:18.04
docker.io/library/ubuntu:18.04
```

启动一个容器，并保持其在后台运行

```shell
$ docker run -itd -v ~/Code/chibicc:/chibicc ubuntu:18.04 bash
```

## VS Code 容器开发环境搭建 

VS Code 安装 Remote Development 插件

连接到容器内

![](/posts/chibicc-compilerbook-step-00/images/截屏2022-08-02%20上午12.18.05-tuya.webp)

打开文件夹

![](/posts/chibicc-compilerbook-step-00/images/截屏2022-08-02%20上午12.23.24-tuya.webp)

安装构建依赖

```shell
$ apt update
$ apt install -y gcc make git binutils libc6-dev gdb
```

![](/posts/chibicc-compilerbook-step-00/images/截屏2022-08-02%20上午12.53.53-tuya.webp)

## 编译 chibicc

```shell
$ make 
$ make test-all
```

## 测试

```c
#include <stdio.h>

int main() {
  printf("Hello world!\n");
  return 0;
}
```

![](/posts/chibicc-compilerbook-step-00/images/截屏2022-08-02%20上午12.54.52-tuya.webp)
