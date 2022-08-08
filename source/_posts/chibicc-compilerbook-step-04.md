---
title: 自制 C 编译器 - 04 - 改进错误信息
toc: true
date: 2022-08-02 22:15:00
tags: 
- chibicc
- compiler
categories: 
- 自制 C 编译器
---

目前制作的编译器，如果输入在语法上不正确，它只会笼统地告诉我们某处有错误，但不能主动定位错误。因此，这一步的目标是让编译器显示更加直观的错误信息。

<!-- more -->

## 预期效果

```bash
$ ./chibicc "1+3++" > tmp.s
1+3++
    ^ expected a number

$ ./chibicc "1 + foo + 5" > tmp.s
1 + foo + 5
    ^ expected a number
```

## 错误显示函数

将整个程序字符串保存在一个名为 user_input 的变量中，并定义一个新的错误显示函数，该函数接收指向字符串中间的指针，输出错误定位信息。

```c
char *user_input;

void error_at(char *loc, char *fmt, ...) {
  va_list ap;
  va_start(ap, fmt);

  int pos = loc - user_input;
  fprintf(stderr, "%s\n", user_input);
  fprintf(stderr, "%*s", pos, "");
  fprintf(stderr, "^ ");
  vfprintf(stderr, fmt, ap);
  fprintf(stderr, "\n");
  exit(1);
}
```

## 测试

对于生产级编译器，应该在输入中有错误时编写行为测试。但目前错误消息只是输出以帮助调试，因此不需要在此阶段编写任何测试。

## 小结

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}
参考实现：
- [c6ff1d9](https://github.com/rui314/chibicc/commit/c6ff1d98a1419e69c31902447e2caa85af4e9844): Improve error message
- [f7a95db](https://github.com/zhifeng-essen/chibicc/commit/f7a95db926bc69bc28420ea0c91d18c692ab2386): 改进错误信息
{% raw %}</div></article>{% endraw %}
