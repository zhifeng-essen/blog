---
title: 自制 C 编译器 - 02 - 实现加减运算
toc: true
date: 2022-08-02 11:55:18
tags: 
- chibicc
- compiler
categories: 
- 自制 C 编译器
---

扩展[上一步](/posts/chibicc-compilerbook-step-01)中创建的编译器，以接受像 `2+11` 和 `5+20-4` 这样的涉及加减法的表达式，而不仅仅是单个整数。

<!-- more -->

## 实现思路

将加减法表达式视为一种“语言”，该语言可以定义如下：
- 以一个数字开头
- 后跟 0 个或多个“项”
- “项”是 `+` 后跟数字或 `-` 后跟数字

## 编译器实现

```c main.c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv) {
  if (argc != 2) {
    fprintf(stderr, "%s: invalid number of arguments\n", argv[0]);
    return 1;
  }

  char *p = argv[1];

  printf(".intel_syntax noprefix\n");
  printf(".global main\n");
  printf("main:\n");
  printf("  mov rax, %ld\n", strtol(p, &p, 10));

  while (*p) {
    if (*p == '+') {
      p++;
      printf("  add rax, %ld\n", strtol(p, &p, 10));
      continue;
    }

    if (*p == '-') {
      p++;
      printf("  sub rax, %ld\n", strtol(p, &p, 10));
      continue;
    }

    fprintf(stderr, "unexpected character: '%c'\n", *p);
    return 1;
  }

  printf("  ret\n");
  return 0;
}
```

C 库函数 `long int strtol(const char *str, char **endptr, int base)` 把参数 `str` 所指向的字符串根据给定的 `base` 转换为一个长整数，`endptr` 的值由函数设置为 `str` 中数值后的下一个字符。

## 添加测试

为了测试这个新特性，在 test.sh 中添加一个测试：

```diff test.sh
assert 0 0
assert 42 42
+ assert 21 '5+20-4'
```

## 小结

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}
参考实现：
- [afc9e8f](https://github.com/rui314/chibicc/commit/afc9e8f05faddf051aa3a578520d6484ab451282): Add + and - operators
- [c21978c](https://github.com/zhifeng-essen/chibicc/commit/c21978c77c3b921febebee7642bcb63d74edcd07): 实现加减法
{% raw %}</div></article>{% endraw %}
