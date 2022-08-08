---
title: 自制 C 编译器 - 00 - 搭建开发环境
toc: true
date: 2022-08-01 12:00:00
tags: 
- chibicc
- compiler
- Docker
categories: 
- 自制 C 编译器
---

VS Code + Docker 搭建编译 [chibicc](https://github.com/rui314/chibicc) 所需的 64 位 Linux 环境

<!-- more -->

[chibicc](https://github.com/rui314/chibicc) 项目是 Rui Ueyama 为自己的书 [《低レイヤを知りたい人のためのCコンパイラ作成入門》](https://www.sigbus.info/compilerbook) 写的参考实现，作者不仅对最终代码的可读性进行了把控，而是保证了每个 commit 的内容都相对独立，大小适中，且 commit log 写的非常清晰。读者可以随着 commit log 看到项目如何从只能输出一个整数的小程序，最终成长为一个可以完成自举的编译器。

原书 [附录3](https://www.sigbus.info/compilerbook#docker) 介绍了如何使用 Docker 在 macOS 上构建和运行 Linux 应用程序。本文在此基础上介绍如何通过 VS Code 的 Remote Development 插件实现 Docker 容器内开发环境的搭建。

## 下载 chibicc 源码

```bash
$ cd ~/Code
$ git clone https://github.com/rui314/chibicc.git
```

## 拉取 Docker 镜像

选择 ubuntu:18.04 镜像

```bash
$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
Digest: sha256:478caf1bec1afd54a58435ec681c8755883b7eb843a8630091890130b15a79af
Status: Downloaded newer image for ubuntu:18.04
docker.io/library/ubuntu:18.04
```

启动一个容器，并保持其在后台运行

```bash
$ docker run -itd -v ~/Code/chibicc:/chibicc ubuntu:18.04 bash
```

## VS Code 容器开发环境搭建 

VS Code 安装 Remote Development 插件

连接到容器内

![](/posts/chibicc-compilerbook-step-00/images/截屏2022-08-02%20上午12.18.05-tuya.webp)

打开文件夹

![](/posts/chibicc-compilerbook-step-00/images/截屏2022-08-02%20上午12.23.24-tuya.webp)

安装构建依赖

```bash
$ apt update
$ apt install -y gcc make git binutils libc6-dev gdb
```

![](/posts/chibicc-compilerbook-step-00/images/截屏2022-08-02%20上午12.53.53-tuya.webp)

## 编译和测试

依赖安装好以后，chibicc 编译器的编译构建十分简单

```bash
$ make 
$ make test-all
```

## Hello world!

最后，编译一个简单的 C 程序

```c hello.c
#include <stdio.h>

int main() {
  printf("Hello world!\n");
  return 0;
}
```

![](/posts/chibicc-compilerbook-step-00/images/截屏2022-08-02%20上午12.54.52-tuya.webp)
