---
title: 自制 C 编译器 - 06 - 实现一元加减运算
toc: true
date: 2022-08-07 21:40:34
tags: 
- chibicc
- compiler
categories: 
- 自制 C 编译器
---

执行减法的 `-` 运算符不仅可以写在两个项之间，例如 `5-3`，还可以写在单个项之前，例如 `-3`。类似地，`+` 运算符也可以通过省略左侧写成 `+3`。像这样只取一个术语的运算符称为一元运算符。

<!-- more -->

除了 `+` 和 `-` 之外，C 语言还有其他的一元运算符，例如接受指针的 & 和取消引用的 *，但在这一步中，我们将仅实现 `+` 和 `-`。

## 新语法的生成式

带有一元 +/- 的新语法如下所示：

```
expr    = mul ("+" mul | "-" mul)*
mul     = unary ("*" unary | "/" unary)*
unary   = ("+" | "-")? primary
primary = num | "(" expr ")"
```

## 修改解析器

上面的新语法添加了一个新的非终结符 `unary`。更改我们的解析器以遵循这个新语法：

```c main.c
Node *unary() {
  if (consume('+'))
    return primary();
  if (consume('-'))
    return new_node(ND_SUB, new_node_num(0), primary());
  return primary();
}
```

在解析阶段用 `x` 替换 `+x` 和用 `0-x` 替换 `-x`。因此，此步骤不需要更改代码生成器。

## 测试

添加测试：

```bash test.sh
assert 10 '-10+20'
assert 10 '- -10'
assert 10 '- - +10'
```

## 小结

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}
参考实现：
- [bb5fe99](https://github.com/rui314/chibicc/commit/bb5fe99dbad62c9516ec6a4bc64e444d09115e6d): Add unary plus and minus
- [4246009](https://github.com/zhifeng-essen/chibicc/commit/4246009f6a24e9c47fa7e7146e2cb86b9ada01d6): 实现一元加减运算
{% raw %}</div></article>{% endraw %}
