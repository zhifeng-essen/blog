---
title: 自制 C 编译器 - 03 - 引入 Tokenizer
date: 2022-08-02 18:22:22
tags: 
- chibicc
- compiler
categories: 
- 自制 C 编译器
toc: true
---

[上一步](/posts/chibicc-compilerbook-step-02)中构建的编译器有一个缺点：如果输入包含空白字符，则会发生错误。一种解决方法是在尝试读取 `+` 或 `-` 之前跳过空白字符。但在这一步中，我们将尝试一种不同的方式——将输入字符串拆分为 Token 序列。

<!-- more -->

## 定义

在当前的加减表达式语法中，Token 分为三种：`+`、`-` 和数字。此外，定义一个特殊类型来表示 Token 序列的结尾可以简化程序（类似于以 `\0` 结尾的字符串）。

使用链表，以便我们可以处理任意长度的输入。

```c
typedef enum {
  TK_RESERVED,
  TK_NUM,
  TK_EOF,
} TokenKind;

typedef struct Token Token;
struct Token {
  TokenKind kind;
  Token *next;
  int val;
  char *str;
};
```

## Tokenizer

在构建链表时，如果创建一个虚拟头元素，并在最后返回 `head->next`，则代码会更简单。这样分配给头元素的内存大部分都被浪费了，但是分配局部变量的开销几乎为零，所以不用担心。

calloc 是一个类似于 malloc 的内存分配函数。与 malloc 不同，calloc 会将分配的内存置零。

```c
Token *new_token(TokenKind kind, Token *cur, char *str) {
  Token *tok = calloc(1, sizeof(Token));
  tok->kind = kind;
  tok->str = str;
  cur->next = tok;
  return tok;
}

Token *tokenize(char *p) {
  Token head;
  head.next = NULL;
  Token *cur = &head;

  while (*p) {
    if (isspace(*p)) {
      p++;
      continue;
    }
    if (*p == '+' || *p == '-') {
      cur = new_token(TK_RESERVED, cur, p++);
      continue;
    }
    if (isdigit(*p)) {
      cur = new_token(TK_NUM, cur, p);
      cur->val = strtol(p, &p, 10);
      continue;
    }
    error("invalid token");
  }

  new_token(TK_EOF, cur, p);
  return head.next;
}
```

## 添加测试

这个改进的版本应该可以跳过空白字符，在 test.sh 中添加以下一行测试。

```diff test.sh
assert 0 0
assert 42 42
assert 21 '5+20-4'
+ assert 41 " 12 + 34 - 5 "
```

Unix 进程退出代码应该是 0 到 255 之间的数字，所以这里我们编写测试时，尽量将整个表达式的结果保持在 0 到 255 之间。

## 小结

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}

参考实现：

- [5f158de](https://github.com/zhifeng-essen/chibicc/commit/5f158deb3003bd9db5d1cc0396a5ecfa86e9efb2): 引入 Tokenizer

- [ef6d179](https://github.com/rui314/chibicc/commit/ef6d1791eb2a5ef3af913945ca577ea76d4ff97e): Add a tokenizer to allow space characters between tokens

{% raw %}</div></article>{% endraw %}
