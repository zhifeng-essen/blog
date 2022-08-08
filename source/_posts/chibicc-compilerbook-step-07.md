---
title: 自制 C 编译器 - 07 - 加入比较运算符
toc: true
date: 2022-08-08 19:23:54
tags: 
- chibicc
- compiler
categories: 
- 自制 C 编译器
---

本节实现比较运算符：`<`、`<=`、`>`、`>=`、`==` 和 `!=`。

<!-- more -->

尽管这些比较运算符看起来有特殊含义，但它们实际上是普通的二元运算符，它们接受两个整数并返回一个整数。就像 `+` 返回两边相加的结果一样，`==` 如果两边相同则返回 `1`，否则返回 `0`。

## 修改词法分析器

到目前为止，我们处理的所有 Token 都是一个字符长，并且我们已经在代码中假设了这一点，但是现在我们需要泛化代码以处理像 `==` 这样的比较运算符。在 Token 结构体中新增 `len` 成员，以便将字符串的长度存储在 Token 中。新的结构体类型如下所示：

```c main.c
// Token type
typedef struct Token Token;
struct Token {
  TokenKind kind; // Token kind
  Token *next;    // Next token
  int val;        // If kind is TK_NUM, its value
  char *str;      // Token string
  int len;        // Token length
};
```

随着这一变化，`consume` 和 `expect` 等函数也需要修改：

```c main.c
// Consumes the current token if it matches `op`.
bool consume(char *op) {
  if (token->kind != TK_RESERVED || strlen(op) != token->len ||
      memcmp(token->str, op, token->len))
    return false;
  token = token->next;
  return true;
}

// Ensure that the current token is `op`.
void expect(char *op) {
  if (token->kind != TK_RESERVED || strlen(op) != token->len ||
      memcmp(token->str, op, token->len))
    error_at(token->str, "expected \"%s\"", op);
  token = token->next;
}
```

## 新语法的生成式

```
expr       = equality
equality   = relational ("==" relational | "!=" relational)*
relational = add ("<" add | "<=" add | ">" add | ">=" add)*
add        = mul ("+" mul | "-" mul)*
mul        = unary ("*" unary | "/" unary)*
unary      = ("+" | "-")? primary
primary    = num | "(" expr ")"
```

`equality` 代表 `==` 和 `!=`，`relational` 代表 `<`、`<=`、`>`、`>=`。注意，在上面的语法中，`expr` 和 `equality` 是分开的，表示整个表达式是相等的。我本可以将 `equality` 的右侧直接写在 `expr` 的右侧，但我认为上面的语法可能更容易阅读。

## 修改解析器

```c
// expr = equality
Node *expr() {
  return equality();
}

// equality = relational ("==" relational | "!=" relational)*
Node *equality() {
  Node *node = relational();

  for (;;) {
    if (consume("=="))
      node = new_binary(ND_EQ, node, relational());
    else if (consume("!="))
      node = new_binary(ND_NE, node, relational());
    else
      return node;
  }
}

// relational = add ("<" add | "<=" add | ">" add | ">=" add)*
Node *relational() {
  Node *node = add();

  for (;;) {
    if (consume("<"))
      node = new_binary(ND_LT, node, add());
    else if (consume("<="))
      node = new_binary(ND_LE, node, add());
    else if (consume(">"))
      node = new_binary(ND_LT, add(), node);
    else if (consume(">="))
      node = new_binary(ND_LE, add(), node);
    else
      return node;
  }
}
```

## 修改汇编生成器

在 `x86-64` 上，比较是使用 `cmp` 指令完成的。下面的代码从堆栈中弹出两个整数，比较它们，如果它们相等，则将 RAX 设置为 1，否则设置为 0。

```
pop rdi
pop rax
cmp rax, rdi
sete al
movzb rax, al
```

前两行将值从堆栈中弹出。第 3 行比较那些弹出的值。比较结果去哪了？在 x86-64 上，比较指令的结果设置在一个特殊的“标志寄存器”中。`sete al` 取标志寄存器中 ZF 的值, 放到 AL 中。ZF 是零标志位，它记录相关指令执行后，其结果是否为 `0`，如果结果为 `0`，那么 `zf=1`；如果结果不为 `0`，那么 `zf=0`。

AL 实际上只是 RAX 低 8 位的另一个名称。因此，当 `sete` 在 AL 中设置值时，它也会自动更新 RAX。但是，通过 AL 更新 RAX 时，高 56 位保持不变，因此如果要将整个 RAX 设置为 `0` 或 `1`，则必须将高 56 位清零。这就是 `movzb` 指令的作用。如果 `sete` 指令可以直接写入 RAX 就好了，但是由于 `sete` 只能接受一个 8 位寄存器作为参数，所以比较指令使用两条指令来设置 RAX 中的值。

其他比较运算符可以通过使用不同的指令代替 `sete` 来实现。对 `<` 使用 `setl`，对 `<=` 使用 `setle`，对 `!=` 使用 `setne`。`>` 和 `>=` 不需要代码生成器支持，只需在解析器中交换两边并读取为 `<` 或 `<=`。

```c main.c
void gen(Node *node) {
  if (node->kind == ND_NUM) {
    printf("  push %d\n", node->val);
    return;
  }

  gen(node->lhs);
  gen(node->rhs);

  printf("  pop rdi\n");
  printf("  pop rax\n");

  switch (node->kind) {
  case ND_ADD:
    printf("  add rax, rdi\n");
    break;
  case ND_SUB:
    printf("  sub rax, rdi\n");
    break;
  case ND_MUL:
    printf("  imul rax, rdi\n");
    break;
  case ND_DIV:
    printf("  cqo\n");
    printf("  idiv rdi\n");
    break;
  case ND_EQ:
    printf("  cmp rax, rdi\n");
    printf("  sete al\n");
    printf("  movzb rax, al\n");
    break;
  case ND_NE:
    printf("  cmp rax, rdi\n");
    printf("  setne al\n");
    printf("  movzb rax, al\n");
    break;
  case ND_LT:
    printf("  cmp rax, rdi\n");
    printf("  setl al\n");
    printf("  movzb rax, al\n");
    break;
  case ND_LE:
    printf("  cmp rax, rdi\n");
    printf("  setle al\n");
    printf("  movzb rax, al\n");
    break;
  }

  printf("  push rax\n");
}
```

## 小结

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}
参考实现：
- [6ddba4b](https://github.com/rui314/chibicc/commit/6ddba4be5f63388607fc77fd786267b9ddcb14c9): Add ==, !=, <= and >= operators
- [bf64f7c](https://github.com/zhifeng-essen/chibicc/commit/bf64f7cc96f87444be1721d23c5cb538844a1f40): 加入比较运算符
{% raw %}</div></article>{% endraw %}
