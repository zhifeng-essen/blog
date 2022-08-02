---
title: 自制 C 编译器 - 01 - 编译单个整数
date: 2022-08-01 14:34:47
tags: 
- chibicc
- compiler
categories: 
- 自制 C 编译器
toc: true
---

C 语言最简单的子集

<!-- more -->

## 编译器主体

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv) {
  if (argc != 2) {
    fprintf(stderr, "%s: invalid number of arguments\n", argv[0]);
    return 1;
  }

  printf(".intel_syntax noprefix\n");
  printf(".global main\n");
  printf("main:\n");
  printf("  mov rax, %d\n", atoi(argv[1]));
  printf("  ret\n");
  return 0;
}
```

## 自动化测试

```shell
#!/bin/bash
assert() {
  expected="$1"
  input="$2"

  ./chibicc "$input" > tmp.s
  gcc -static -o tmp tmp.s
  ./tmp
  actual="$?"

  if [ "$actual" = "$expected" ]; then
    echo "$input => $actual"
  else
    echo "$input => $expected expected, but got $actual"
    exit 1
  fi
}

assert 0 0
assert 42 42

echo OK
```

## 编写 Makefile

```makefile
CFLAGS=-std=c11 -g -static

chibicc: main.o
	$(CC) -o $@ $? $(LDFLAGS)

test: chibicc
	./test.sh

clean:
	rm -f chibicc *.o *~ tmp*

.PHONY: test clean
```

## 使用 git 进行版本控制

.gitignore

```
*~
*.o
tmp*
a.out
9cc
```

```shell
$ git config --global user.name "zhifeng-essen"
$ git config --global user.email "zhifeng-essen@icloud.com"
```

```shell
$ git init
$ git add main.c test.sh Makefile .gitignore
```

```shell
$ git commit -m "创建一个编译单个整数的编译器"
```

通过运行 `git log -p` 来检查提交是否成功

```shell
$ git log -p
commit 6a43fd2529abfa6c2ebd6d53fd0c54c39c312f12 (HEAD -> master)
Author: zhifeng-essen <zhifeng-essen@icloud.com>
Date:   Tue Aug 2 03:13:54 2022 +0000

    创建一个编译单个整数的编译器

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..68e72e6
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,5 @@
+*~
+*.o
+tmp*
+a.out
+chibicc
...
```

推送到 Github 远程仓库

![](/posts/chibicc-compilerbook-step-01/images/FireShot%20Capture%20001%20-%20Create%20a%20New%20Repository%20-%20github.com-tuya.webp)

Check out the github guide to [generating SSH keys](https://docs.github.com/articles/generating-an-ssh-key/) or troubleshoot [common SSH problems](https://docs.github.com/ssh-issues/).

```
$ git remote add origin git@github.com:zhifeng-essen/chibicc.git
$ git branch -M main
$ git push -u origin main
```

## 小结

这样就完成了创建编译器的第一步。虽然这一步我们实现的编译器是一个过于简单而很难称之为编译器的程序，但它是一个包含编译器所需的所有元素的好程序。从现在开始，我们将渐进地增强这个编译器，不管你信不信，它会成长为一个优秀的 C 编译器。

参考实现：

- [6a43fd2529abfa6c](https://github.com/zhifeng-essen/chibicc/commit/6a43fd2529abfa6c2ebd6d53fd0c54c39c312f12): 创建一个编译单个整数的编译器

- [f722daaaae060611](https://github.com/rui314/chibicc/commit/f722daaaae0606115df4ace5a852da23c1a5b0f3): Compile an integer to an exectuable that exits with the given number
