---
title: 自制 C 编译器 - 05 - 实现四则运算
toc: true
date: 2022-08-02 23:13:20
tags: 
- chibicc
- compiler
categories: 
- 自制 C 编译器
---

为了实现四则运算，需要在语言中添加乘法、除法和括号，即 `*`、`/` 和 `()`。但是这样做有一个很大的技术挑战——如何处理运算符优先级？

<!-- more -->
## 产生式

大多数编程语言的语法是使用所谓的产生式（production rule）定义的。

请考虑以下产生式：

```
expr = num ("+" num | "-" num)*
```

该规则实际上描述了加减表达式的语法。

产生式是表达语法的非常强大的工具，运算符优先级也可以在产生式中表示：

```
expr = mul ("+" mul | "-" mul)*
mul  = num ("*" num | "/" num)*
```

`mul` 是乘法和除法的产生式规则，执行加法和减法的 `expr` 使用 `mul` 作为一个组件。在这个语法中，乘除法先行的规则自然地用语法树来表达。

生成文法也可以写成递归的，以下是在算术运算中添加了括号的语法产生式：

```
expr    = mul ("+" mul | "-" mul)*
mul     = primary ("*" primary | "/" primary)*
primary = num | "(" expr ")"
```

## 语法分析器/解析器（Parser）

定义抽象语法树的节点类型：

```c
typedef enum {
  ND_ADD, // +
  ND_SUB, // -
  ND_MUL, // *
  ND_DIV, // /
  ND_NUM, // 整数
} NodeKind;

typedef struct Node Node;

// 抽象语法树节点类型
struct Node {
  NodeKind kind;
  Node *lhs;
  Node *rhs;
  int val;       // 仅在 kind 为 ND_NUM 时使用
};
```

创建新节点的函数：

```c
Node *new_node(NodeKind kind, Node *lhs, Node *rhs) {
  Node *node = calloc(1, sizeof(Node));
  node->kind = kind;
  node->lhs = lhs;
  node->rhs = rhs;
  return node;
}

Node *new_node_num(int val) {
  Node *node = calloc(1, sizeof(Node));
  node->kind = ND_NUM;
  node->val = val;
  return node;
}
```

使用这些函数和数据类型编写一个解析器：

```c
Node *expr() {
  Node *node = mul();

  for (;;) {
    if (consume('+'))
      node = new_node(ND_ADD, node, mul());
    else if (consume('-'))
      node = new_node(ND_SUB, node, mul());
    else
      return node;
  }
}
```

`consume` 是之前定义的函数，当此时的 Token 与参数匹配时返回 `true`，并推进到下一个标记；否则返回 `false`。

可以看到产生式 `expr = mul ("+" mul | "-" mul)*` 直接映射到函数调用和循环。

接着定义 `expr` 函数使用的 `mul` 函数。由于 `*` 和 `/` 也是左结合运算符，所以可以用相同的模式编写。

```c
Node *mul() {
  Node *node = primary();

  for (;;) {
    if (consume('*'))
      node = new_node(ND_MUL, node, primary());
    else if (consume('/'))
      node = new_node(ND_DIV, node, primary());
    else
      return node;
  }
}
```

上面代码中的函数调用关系直接对应产生式 `mul = primary ("*" primary | "/" primary)*`。

最后定义 `primary` 函数：

```c
Node *primary() {
  // 如果下一个 Token 是 `(`，则应该是 `"(" expr ")"`
  if (consume('(')) {
    Node *node = expr();
    expect(')');
    return node;
  }

  // 否则应该是一个 num
  return new_node_num(expect_number());
}
```

不熟悉递归的程序员可能会对上述递归函数感到困惑。说实话，即使对我这个应该很熟悉递归的人来说，这段代码的运行也感觉像是一种魔法。即使你知道递归代码是如何工作的，它也会让人觉得有点奇怪，但它可能就是这样。

如上所述，将一个产生式规则映射到一个函数的解析技术称为“递归下降解析”。上面的解析器只预取一个 Token 并决定调用或返回哪个函数，这种只向前看一个 Token 的递归下降解析器称为 LL(1) 解析器。此外，可以编写 LL(1) 解析器的文法称为 LL(1) 文法。

## 栈机（Stack Machine）

栈机中的算术指令对堆栈顶部的元素进行操作。例如 ADD 指令从栈顶弹出两个元素，相加，然后将结果压入栈中。

SUB、MUL 和 DIV 指令与 ADD 类似，将堆栈顶部的两个元素替换为减、乘或除它们的结果。

例如，可以使用这些指令计算 `2*3+4*5`：

```
// 计算 2*3
PUSH 2
PUSH 3
MUL

// 计算 4*5
PUSH 4
PUSH 5
MUL

// 计算 2*3 + 4*5
ADD
```

转换为 x86-64 汇编：

```
// 计算 2*3 并将结果压入堆栈
push 2
push 3

pop rdi
pop rax
mul rax, rdi
push rax

// 计算 4*5 并将结果压入堆栈
push 4
push 5

pop rdi
pop rax
mul rax, rdi
push rax

// 将栈顶的两个值相加
// 即计算 2*3+4*5
pop rdi
pop rax
add rax, rdi
push rax
```

以下生成器函数是该方法的 C 语言实现：

```c
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
  }

  printf("  push rax\n");
}
```

`div` 为无符号除法，`idiv` 为有符号除法。idiv 进行的是 `128 / 64` 位除法，即被除数为 `128` 位、除数为 `64` 位。`64` 位操作系统中寄存器大小当然只有 `64` 位，因此，`idiv` 使用 `rdx:rax` 作为被除数。即 `rdx` 中的值作为高 `64` 位、`rax` 中的值作为低 `64` 位。`cqo` 指令的作用是扩展双字，即从 64 位扩展到 `128` 位，具体是将 `rax` 寄存器的值进行扩展，高位置于 `rdx` 中。所以上面的代码在调用 `idiv` 之前调用了 `cqo`。

## 编译器实现

更改编译器的 `main` 函数以使用新创建的解析器和代码生成器：

```c main.c
int main(int argc, char **argv) {
  if (argc != 2) {
    error("引数の個数が正しくありません");
    return 1;
  }

  // トークナイズしてパースする
  user_input = argv[1];
  token = tokenize(user_input);
  Node *node = expr();

  // アセンブリの前半部分を出力
  printf(".intel_syntax noprefix\n");
  printf(".globl main\n");
  printf("main:\n");

  // 抽象構文木を下りながらコード生成
  gen(node);

  // スタックトップに式全体の値が残っているはずなので
  // それをRAXにロードして関数からの返り値とする
  printf("  pop rax\n");
  printf("  ret\n");
  return 0;
}
```

## 测试

在这个阶段，带有加法、减法、乘法和除法以及括号的表达式应该可以正确编译。让我们添加一些测试：

```bash test.sh
assert 47 '5+6*7'
assert 15 '5*(9-6)'
assert 4 '(3+5)/2'
```

## 小结

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}
参考实现：
- [3c1e383](https://github.com/rui314/chibicc/commit/3c1e3831009edff2ea237d3e59680ba9d4bb2e14): Add *, / and ()
- [780775b](https://github.com/zhifeng-essen/chibicc/commit/780775be7447b9f11112625f53dffa71f23bc448): 实现四则运算
{% raw %}</div></article>{% endraw %}

