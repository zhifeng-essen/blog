---
title: Taichi 语言参考手册 v1.0.4
toc: true
date: 2022-08-03 16:19:51
tags:
- Taichi Lang
categories:
- Taichi 编程语言
---

最近在 [crowdin](https://crowdin.com/project/taichi-programming-language) 协助翻译了 Taichi 文档中的 [Language Reference](https://docs.taichi-lang.org/docs/reference/)，顺便贴在这里，内容做了一些微调。

<!-- more -->

## 介绍

由于 Taichi 是一个嵌入在 Python 中的 DSL，所以 Taichi 的语法是 Python 语法的一个子集。[Language Reference](https://docs.taichi-lang.org/docs/reference/) 也以 [Python 语言参考](https://docs.python.org/3/reference/) 为蓝本，完全沿用了 Python 的 [记号](https://docs.python.org/3/reference/introduction.html#notation) 和 [词法分析](https://docs.python.org/3/reference/lexical_analysis.html) 部分。

## 基本概念
### 值和类型

Taichi 中的每个表达式都会被计算为一个值，并且每个值都有一个类型。

由于 Taichi 涉及与 Python 交互和 [元编程](https://docs.taichi-lang.org/docs/meta)，所以有 “编译期求值” 和 “运行期求值”，相应地产生了两种值：“Python 值” 和 “Taichi 值”。Python 值只存在于编译期。在编译期求值后，所有余下的表达式将在运行期计算为 Taichi 值。

Taichi 值具有 Taichi 类型，它可以是：
- [基本类型](https://docs.taichi-lang.org/docs/type/#primitive-types)
- [复合类型](https://docs.taichi-lang.org/docs/type/#compound-types)
- [Ndarray 类型](https://docs.taichi-lang.org/docs/ndarray_android#a-definition-of-ndarray)
- [稀疏矩阵 builder 类型](https://docs.taichi-lang.org/docs/sparse_matrix)

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}
求值规则的非正式总结：
- Python 值 + Python 值 = Python 值
- Python 值 + Taichi 值 = Taichi 值
- Taichi 值 + Taichi 值 = Taichi 值
{% raw %}</div></article>{% endraw %}

### 变量和作用域

在 Taichi 中，变量可以通过以下方式定义：
- 函数形参
- 赋值语句

Taichi 是静态类型的，采用静态作用域。

### 二元运算的通用规则

1. 如果两个操作数都是 Python 值，会触发编译期求值，结果是一个 Python 值。

2. 如果只有一个操作数是 Python 值，那么 Python 值首先会被转换为默认类型的 Taichi 值。这样就转化成了两个操作数都是 Taichi 值的情况。

3. 如果两个操作数都是 Taichi 值，又可分为三种情况：
    - 两个基本类型值：返回值还是基本类型。
    - 一个基本类型值和一个复合类型值：基本类型值首先通过广播（broadcast）机制转换为复合类型值。返回值是复合类型。
    - 两个复合类型值：两个值需要有相同的形状（矩阵乘法例外），逐元素进行运算。返回值是复合类型。

## 表达式

### 原子

原子是表达式的最基本构成元素。最简单的原子是标识符和字面值。以圆括号、方括号或花括号包括的形式在语法上也被归类为原子。

```
atom      ::= identifier | literal | enclosure
enclosure ::= parenth_form | list_display | dict_display
```

#### 标识符（名称）

Taichi 中标识符（名称）的词法定义遵循 [Python](https://docs.python.org/3/reference/lexical_analysis.html#identifiers)。

```
identifier   ::=  xid_start xid_continue*
id_start     ::=  <all characters in general categories Lu, Ll, Lt, Lm, Lo, Nl, the underscore, and characters with the Other_ID_Start property>
id_continue  ::=  <all characters in id_start, plus characters in the categories Mn, Mc, Nd, Pc and others with the Other_ID_Continue property>
xid_start    ::=  <all characters in id_start whose NFKC normalization is in "id_start xid_continue*">
xid_continue ::=  <all characters in id_continue whose NFKC normalization is in "id_continue*">
```

求值时有三种情况：
- 如果名称是可见的，并且对应于 Taichi 作用域中定义的变量，那么求值结果是运行时该变量的值。
- 如果名称只在 Python 作用域中可见，即名称绑定在 Taichi 作用域之外，那么会触发编译期求值，将 Python 值绑定到名称。
- 如果名称是不可见的，那么会抛出 `TaichiNameError` 错误。

#### 字面量

Taichi 支持 [整数](https://docs.python.org/3/reference/lexical_analysis.html#integer-literals) 和 [浮点数](https://docs.python.org/3/reference/lexical_analysis.html#floating-point-literals) 字面量，其词法定义遵循 Python。

```
literal ::= integer | floatnumber

integer      ::=  decinteger | bininteger | octinteger | hexinteger
decinteger   ::=  nonzerodigit (["_"] digit)* | "0"+ (["_"] "0")*
bininteger   ::=  "0" ("b" | "B") (["_"] bindigit)+
octinteger   ::=  "0" ("o" | "O") (["_"] octdigit)+
hexinteger   ::=  "0" ("x" | "X") (["_"] hexdigit)+
nonzerodigit ::=  "1"..."9"
digit        ::=  "0"..."9"
bindigit     ::=  "0" | "1"
octdigit     ::=  "0"..."7"
hexdigit     ::=  digit | "a"..."f" | "A"..."F"

floatnumber   ::=  pointfloat | exponentfloat
pointfloat    ::=  [digitpart] fraction | digitpart "."
exponentfloat ::=  (digitpart | pointfloat) exponent
digitpart     ::=  digit (["_"] digit)*
fraction      ::=  "." digitpart
exponent      ::=  ("e" | "E") ["+" | "-"] digitpart
```

字面量在编译期被计算为 Python 值。

#### 带圆括号的形式

```
parenth_form ::= "(" [expression_list] ")"
```

对带圆括号的形式求值等于对其中的表达式列表求值。一对内容为空的圆括号在编译期被求值为一个空的元组。

#### 列表和字典的显示

Taichi 支持容器（仅限列表和字典）构造的 [显示](https://docs.python.org/3/reference/expressions.html#displays-for-lists-sets-and-dictionaries)。与 Python 相似，“显示”有两种形式：
- 显式地列出容器的项
- 提供一个“推导式”（一组循环和筛选指令）以计算出容器的项

```
list_display       ::= "[" [expression_list | list_comprehension] "]"
list_comprehension ::= assignment_expression comp_for
dict_display       ::= "{" [key_datum_list | dict_comprehension] "}"
key_datum_list     ::= key_datum ("," key_datum)* [","]
key_datum          ::= expression ":" expression
dict_comprehension ::= key_datum comp_for
comp_for           ::= "for" target_list "in" or_test [comp_iter]
comp_iter          ::= comp_for | comp_if
comp_if            ::= "if" or_test [comp_iter]
```

Taichi 的列表和字典显示语义大体上遵循 Python。注意，由于它们是在编译期求值的，所以 `comp_for` 中的所有表达式，和 `key_datum` 中的键需要被求值为 Python 值。

例如，以下代码片段中，`a` 可以成功定义而 `b` 不能，因为 `p` 无法在编译期求值得到 Python 值。

```python
@ti.kernel
def test(p: ti.i32):
    a = ti.Matrix([i * p for i in range(10)])   # valid
    b = ti.Matrix([i * p for i in range(p)])    # compile error
```

### 原型

原型代表编程语言中最紧密绑定的操作。

```
primary ::= atom | attributeref | subscription | slicing | call
```

#### 属性引用

```
attributeref ::= primary "." identifier
```

属性引用在编译期进行求值。`primary` 必须求值为带有以 `identifier` 为名称的属性的 Python 值。Taichi 中属性引用的常见使用场景包括 [field](https://docs.taichi-lang.org/docs/meta#field-metadata) 和 [矩阵](https://docs.taichi-lang.org/docs/meta#matrix--vector-metadata) 的元数据查询。

例如：

```python
x = ti.field(dtype=ti.f32, shape=(3, 3))

# Print field metadata in Python-scope
print("Field dimensionality is ", x.shape)
print("Field data type is ", x.dtype)

# Print field metadata in Taichi-scope
@ti.kernel
def print_field_metadata(x: ti.template()):
    print("Field dimensionality is ", len(x.shape))
    for i in ti.static(range(len(x.shape))):
        print("Size along dimension ", i, "is", x.shape[i])
    ti.static_print("Field data type is ", x.dtype)
```

#### 抽取

```
subscription ::= primary "[" expression_list "]"
```

如果 `primary` 被求值为 Python 值（如：一个列表或字典)，那么所有在 `expression_list` 中的表达式都需要被求值为 Python 值。并且抽取操作在编译期求值，这与 Python 一致。

否则，`primary` 是 Taichi 类型。除基本类型外，所有 Taichi 类型都支持抽取操作。可以参考这些类型的文档来查看抽取的用法。

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}
如果 `primary` 是 Taichi 矩阵类型，那么 `expression_list` 中所有表达式都需要被求值为 Python 值。这个限制可以通过设置 `ti.init(dynamic_index=True)` 去除。
{% raw %}</div></article>{% endraw %}

#### 切片

```
slicing      ::= primary "[" slice_list "]"
slice_list   ::= slice_item ("," slice_item)* [","]
slice_item   ::= expression | proper_slice
proper_slice ::= [expression] ":" [expression] [ ":" [expression] ]
```

目前，只有 `primary` 具有 Taichi 矩阵类型时才支持切片，并且在编译期进行求值。当 `slice_item` 的形式为：
- 单独的 `expression`：它需要被求值为 Python 值，除非你设置了 `ti.init(dynamic_index=True)`。
- `proper_slice`: 所有表达式（下界、上界和步长）都必须被求值为 Python 值。

#### 调用

```
call                 ::= primary "(" [positional_arguments] ")"
positional_arguments ::= positional_item ("," positional_item)*
positional_item      ::= assignment_expression | "*" expression
```

`primary` 必须被求值为以下之一：
- Taichi 函数。
- Taichi 内置函数。
- Taichi 基本类型。在这种情况下，`positional_arguments` 只能包含一项。如果此项被求值为 Python 值，那么这个基本类型作为字面量的类型注释，且此 Python 值将会转换成这个注释类型的 Taichi 值。否则，该基本类型作为 `ti.cast()` 的语法糖，而此项不能具有复合类型。
- Python 可调用对象。如果此对象不在 [静态表达式](https://docs.taichi-lang.org/docs/reference#static-expressions) 内部，会产生一个警告。

### 幂运算符

```
power ::= primary ["**" u_expr]
```

幂运算符与内置 `pow()` 函数具有相同的语义。

### 一元算术和位运算

```
u_expr ::= power | "-" power | "+" power | "~" power
```

类似于 [二元运算的通用规则](https://docs.taichi-lang.org/docs/reference#common-rules-of-binary-operations)，如果操作数是一个 Python 值，编译期求值被触发，结果生成一个 Python 值。如果操作数是一个 Taichi 值，有以下两种情况：
- 如果操作数是基本类型值，那么返回类型也是基本类型。
- 如果操作数是复合类型值，那么将逐元素进行运算，返回类型是相同形状的复合类型值。

见 [算术运算符](https://docs.taichi-lang.org/docs/operator#arithmetic-operators) 和 [位运算符](https://docs.taichi-lang.org/docs/operator#bitwise-operators)。注意，`~` 只能用于整数类型值。

### 二元算术运算

```
m_expr ::= u_expr | m_expr "*" u_expr | m_expr "@" m_expr | m_expr "//" u_expr | m_expr "/" u_expr | m_expr "%" u_expr
a_expr ::= m_expr | a_expr "+" m_expr | a_expr "-" m_expr
```

见 [二元运算的通用规则](https://docs.taichi-lang.org/docs/reference#common-rules-of-binary-operations)，[二元运算中的隐式类型转换](https://docs.taichi-lang.org/docs/type#implicit-type-casting) 和 [算术运算符](https://docs.taichi-lang.org/docs/operator#arithmetic-operators)。注意，`@` 运算符用于矩阵乘法，只能在矩阵类型参数上操作。

### 移位运算

```
shift_expr::= a_expr | shift_expr ( "<<" | ">>" ) a_expr
```

见 [二元运算的通用规则](https://docs.taichi-lang.org/docs/reference#common-rules-of-binary-operations)，[二元运算中的隐式类型转换](https://docs.taichi-lang.org/docs/type#implicit-type-casting) 和 [位运算符](https://docs.taichi-lang.org/docs/operator#bitwise-operators)。注意，两个操作数都要求是整数类型。

### 二元位运算

```
and_expr ::= shift_expr | and_expr "&" shift_expr
xor_expr ::= and_expr | xor_expr "^" and_expr
or_expr  ::= xor_expr | or_expr "|" xor_expr
```

见 [二元运算的通用规则](https://docs.taichi-lang.org/docs/reference#common-rules-of-binary-operations)，[二元运算中的隐式类型转换](https://docs.taichi-lang.org/docs/type#implicit-type-casting) 和 [位运算符](https://docs.taichi-lang.org/docs/operator#bitwise-operators)。注意，两个操作数都要求是整数类型。

### 比较运算

```
comparison    ::= or_expr (comp_operator or_expr)*
comp_operator ::= "<" | ">" | "==" | ">=" | "<=" | "!=" | ["not"] "in"
```

比较运算可以任意串连，例如 `x < y <= z` 等价于 `x < y and y <= z`。

#### 值比较

见 [二元运算的通用规则](https://docs.taichi-lang.org/docs/reference#common-rules-of-binary-operations)，[二元运算中的隐式类型转换](https://docs.taichi-lang.org/docs/type#implicit-type-casting) 和 [比较运算符](https://docs.taichi-lang.org/docs/operator#comparison-operators)。

#### 成员检测运算

成员检测运算的语义遵循 [Python](https://docs.python.org/3/reference/expressions.html#membership-test-operations)，但只在 [静态表达式](https://docs.taichi-lang.org/docs/reference#static-expressions) 中支持。

### 布尔运算

```
or_test  ::= and_test | or_test "or" and_test
and_test ::= not_test | and_test "and" not_test
not_test ::= comparison | "not" not_test
```

当运算符处于 [静态表达式](https://docs.taichi-lang.org/docs/reference#static-expressions) 内部，运算符的求值规则遵循 [Python](https://docs.python.org/3/reference/expressions.html#boolean-operations)。否则，这种行为取决于 `ti.init()` 的 `short_circuit_operator` 选项：
- 如果 `short_circuit_operators` 为 `False`（默认值），“逻辑与”将会被视为“按位与”，“逻辑或”将会被视为“按位或”。
- 如果 `short_circuit_operators` 为 `True`，通常的短路行为被采用，操作数必须是布尔值。因为 Taichi 还没有布尔类型，目前是以 `ti.i32` 作为一个临时替代。当且仅当一个 `ti.i32` 类型的值为 `0` 时，才被视为 `False`。

### 赋值表达式

```
assignment_expression ::= [identifier ":="] expression
```

赋值表达式将表达式赋给标识符（见 [赋值语句](https://docs.taichi-lang.org/docs/reference#assignment-statements)），同时返回表达式的值。

{% raw %}<article class="message is-info"><div class="message-body">{% endraw %}
此运算符自 Python 3.8 起支持。
{% raw %}</div></article>{% endraw %}

### 条件表达式

```
conditional_expression ::= or_test ["if" or_test "else" expression]
expression             ::= conditional_expression
```

表达式 `x if C y` 首先求值条件 `C` 而不是 `x`。如果 `C` 为真（见 [布尔运算](https://docs.taichi-lang.org/docs/reference#boolean-operations)），那么 `x` 被求值并返回其值；否则，`y` 被求值并返回其值。

### 静态表达式

```
static_expression ::= "ti.static(" positional_arguments ")"
```

静态表达式是指被 `ti.static()` 调用包裹的表达式。 `positional_parties` 是在编译期求值的，其中的项必须求值为 Python 值。

`ti.static()` 接收一个或多个参数。
- 当单个参数被传入时，它会返回这个参数。
- 当多个参数被传入时，它会返回一个顺序与传入次序相同的包含所有这些参数的元组。

静态表达式作为 Taichi 中的一种机制，能够触发许多元编程功能。例如 [编译期循环展开和编译期分支选择](https://docs.taichi-lang.org/docs/meta#compile-time-evaluations)。

静态表达式也可以用于 [为 Taichi field 和 Taichi 函数创建别名](https://docs.taichi-lang.org/docs/syntax_sugars#aliases)。

### 表达式列表

```
expression_list ::= expression ("," expression)* [","]
```

除非是列表显示的一部分，至少包含一个逗号的表达式列表在编译期被求值为一个元组。组件表达式按从左到右的顺序求值。

结尾逗号只在创建长度为 1 的元组时必须，在所有其他情况下是可选的。没有结尾逗号的单个表达式被求值为该表达式的值。

## 简单语句

简单语句由一个单独的逻辑行构成。多条简单语句可以存在于同一行内并以分号分隔。

```
simple_stmt ::= expression_stmt
                | assert_stmt
                | assignment_stmt
                | augmented_assignment_stmt
                | annotated_assignment_stmt
                | pass_stmt
                | return_stmt
                | break_stmt
                | continue_stmt
```

### 表达式语句

```
expression_stmt    ::= expression_list
```

表达式语句会对指定的表达式列表（也可能为单一表达式）进行求值。

### 赋值语句

```
assignment_stmt ::= (target_list "=")+ expression_list
target_list     ::= target ("," target)* [","]
target          ::= identifier
                    | "(" [target_list] ")"
                    | "[" [target_list] "]"
                    | attributeref
                    | subscription
```

赋值语句的递归定义基本遵循 [Python](https://docs.python.org/3/reference/simple_stmts.html#assignment-statements)，但有以下几点注意事项：
- 根据 [变量与作用域](https://docs.taichi-lang.org/docs/reference#variables-and-scope) 一节，如果一个目标是第一次出现的标识符，那么一个变量将用该名称定义，并从相应的右侧表达式推断类型。如果表达式求值为 Python 值，那么它首先会被转换为 [默认类型](https://docs.taichi-lang.org/docs/type#default-primitive-types-for-integers-and-floating-point-numbers) 的 Taichi 值。
- 如果一个目标是已存在的标识符，那么相应的右侧表达式必须求值为具有那个标识符所对应变量类型的 Taichi 值。 否则，将发生隐式类型转换。

#### 增强赋值语句

```
augmented_assignment_stmt ::= augtarget augop expression_list
augtarget                 ::= identifier | attributeref | subscription
augop                     ::= "+=" | "-=" | "*=" | "/=" | "//=" | "%=" |
                              "**="| ">>=" | "<<=" | "&=" | "^=" | "|="
```

不同于 [Python](https://docs.python.org/3/reference/simple_stmts.html#augmented-assignment-statements)，一些增强赋值语句（例如 `x[i] += 1`）在 Taichi 中 [自动具有原子性](https://docs.taichi-lang.org/docs/operator#supported-atomic-operations)。

#### 带标注的赋值语句

```
annotated_assignment_stmt ::= identifier ":" expression "=" expression
```

与一般 [赋值语句](https://docs.taichi-lang.org/docs/reference#assignment-statements) 不同的是：
- 只允许单个目标
- 如果标识符是第一次出现，那么一个变量将用该名称和类型注释（`:` 右边的表达式）定义。右侧表达式被转换为这个注释类型的 Taichi 值。
- 如果标识符是已存在的，那么类型注释必须与那个标识符所对应变量的类型相同。

### `assert` 语句

`assert` 语句是在程序中插入调试性断言的简便方式：

```
assert_stmt ::= "assert" expression ["," expression]
```

`assert` 语句只能在调试模式下运行（当 `ti.init()` 设置 `debug=True` 参数），否则它们等同于无操作。

简单形式 `assert expression`，当 `expression` 为假时引发 `TaichiAssertionError` 错误（`AssertionError` 的子类），并使用` expression` 作为错误消息。

扩展形式 `assert expression1, expression2`，当 `expression1` 为假时引发 `TaichiAssertionError` 错误，并使用 `expression2` 作为错误消息。`expression2` 必须是一个常量或格式化的字符串。格式化字符串中的变量必须是标量。

### `pass` 语句

```
pass_stmt ::= "pass"
```

`pass` 是一个空操作——当它被执行时，什么都不发生。它适合当语法上需要一条语句但并不需要执行任何代码时用来临时占位。

### `return` 语句

```
return_stmt ::= "return" [expression_list]
```

`return` 语句只能在 Taichi kernel 或 Taichi 函数中出现一次，并且它必须位于函数体的底部（Taichi 将来可能会放宽这一限制）。

如果 Taichi kernel 或 Taichi 函数有返回类型提示，那么它必须有 return 语句，且返回的值不能是 None。

如果 Taichi kernel 有 `return` 语句，且返回的值不是 `None`，那么它必须有返回类型提示。Taichi 函数的返回类型提示是可选的，但推荐使用（Taichi 将来可能会强制使用返回类型提示）。

一个 kernel 最多有一个返回值，这个值可以是标量，也可以是 `ti.Matrix` 或 `ti.Vector`，且返回值中的元素数量不超过 30（这个数字是一个实现细节，Taichi 将来可能会放宽这一限制）。

Taichi 函数的 `return` 语句可以有多个返回值，返回值可以是标量、`ti.Matrix`、`ti.Vector`、`ti.Struct` 或其它类型。

### `break` 语句

```
break_stmt ::= "break"
```

`break` 语句在语法上只会出现于 `for` 或 `while` 循环所嵌套的代码，它会终结最近的外层循环。

当最近的外层循环是并行的基于 `range/ndrange` 的 `for` 循环、基于结构体的 `for` 循环或基于网格的 `for` 循环时，`break` 语句不被允许。

### `continue` 语句
```
continue_stmt ::= "continue"
```

`continue` 语句在语法上只会出现于 `for` 或 `while` 循环所嵌套的代码，它会继续执行最近的外层循环的下一个轮次。

## 复合语句

一条复合语句由一个或多个“子句”组成。一个“子句”则包含一个句头和一个“句体”。特定复合语句的“子句头”都处于相同的缩进层级。每个“子句头”以一个作为唯一标识的关键字开始并以一个冒号结束。“子句体”是由一个“子句”控制的一组语句。

```
compound_stmt ::= if_stmt | while_stmt | for_stmt
suite         ::= stmt_list NEWLINE | NEWLINE INDENT statement+ DEDENT
statement     ::= stmt_list NEWLINE | compound_stmt
stmt_list     ::= simple_stmt (";" simple_stmt)* [";"]
```

Taichi 和 Python 的复合语句之间的差别是 Taichi 引入了编译期求值。如果“子句头”中的表达式是静态表达式，那么 Taichi 会根据表达式的求值结果在编译期替换复合语句。

### `if` 语句

`if` 语句用于有条件的执行：

```
if_stmt ::= "if" (static_expression | assignment_expression) ":" suite
            ("elif" (static_expression | assignment_expression) ":" suite)*
            ["else" ":" suite]
```

`elif` “子句” 是一个语法糖，相当于 `if` 语句内嵌于 `else` “子句”。

例如：

```python
if cond_a:
    body_a
elif cond_b:
    body_b
elif cond_c:
    body_c
else:
    body_d
```

等同于：

```python
if cond_a:
    body_a
else:
    if cond_b:
        body_b
    else:
        if cond_c:
            body_c
        else:
            body_d
```

Taichi 首先转换 `elif` “子句”，然后处理只带有一个 `if` “子句” 和可能的一个 `else` “子句”的 `if` 语句。

如果 `if` “子句”的表达式为真（真/假的定义见 [布尔运算符](https://docs.taichi-lang.org/docs/reference#boolean-operations)），那么就执行该“子句体”。否则，`else` “子句体”如果存在就会被执行。

表达式为静态表达式的 `if` 语句被称为静态 `if` 语句。静态 `if` “子句”的表达式在编译期求值，并在编译期按如下规则替换掉复合语句:
- 如果静态表达式为真，那么用 `if` “子句”的“子句体”替换这个静态 `if` 语句。
- 如果静态表达式为真，且 `else` “子句”存在，那么用 `else` “子句”的“子句体”替换这个静态 `if` 语句。
- 如果静态表达式为真，且 `else` “子句”不存在，那么用 `pass` 语句替换这个静态 `if` 语句。

### `while` 语句

`while` 语句用于在表达式保持为真的情况下重复地执行：

```
while_stmt ::= "while" assignment_expression ":" suite
```

这将重复地检验表达式，并且如果其值为真就执行“子句体”；如果表达式值为假（这可能在第一次检验时就发生）则终止循环。

“子句体”中的 [break 语句](https://docs.taichi-lang.org/docs/reference#the-break-statement) 在执行时将终止循环。“子句体”中的 [continue 语句](https://docs.taichi-lang.org/docs/reference#the-continue-statement) 在执行时将跳过“子句体”中的剩余部分并返回检验表达式。

### `for` 语句

Taichi 中的 `for` 语句用于遍历一系列数字、多维区间或 `field` 中元素的索引等。

```
for_stmt        ::= "for" target_list "in" iter_expression ":" suite
iter_expression ::= static_expression | expression
```

Taichi 不支持在 `for` 语句中使用 `else` “子句”。

位于最外层作用域的 `for` 循环可以被并行执行。并行化的 `for` 循环，其执行顺序是不确定的，且不能被 `break` 语句终止。

Taichi 使用 `ti.loop_config` 函数为紧跟其后的循环设置指令。 你可以在基于 range/ndrange 的 `for` 循环前设置 `ti.loop_config(serialize=True)` 以使其串行执行，并可以被 `break` 语句终止。

#### 基于 range 的 `for` 语句

基于 range 的 `for` 语句用来遍历一系列数字。

基于 range 的 `for` 语句的 `iter_expression` 必须形如 `range(start, stop)` 或 `range(stop)`，它们与 [Python 的 `range` 函数](https://docs.python.org/3/library/stdtypes.html#range) 有着相同的含义，只是不支持 `step` 参数。

基于 range 的 `for` 语句的 `target_list` 必须是一个不属于当前作用域的标识符。

当位于最外层作用域时，基于 range 的 `for` 循环默认会被并行化。

#### 基于 ndrange 的 `for` 语句

基于 ndrange 的 `for` 语句用来遍历多维区间。

基于 ndrange 的 `for` 语句的 `iter_expression` 必须是对 `ti.ndrange()` 的调用或对 `ti.grouped(ti.ndrange())` 的嵌套调用。
- 如果 `iter_expression` 是对 `ti.ndrange()` 的调用，那么它是一个一般形式的基于 ndrange 的 `for` 语句。
- 如果 `iter_expression` 是对 `ti.grouped(ti.ndrange())` 的嵌套调用，那么它是一个组合形式的基于 ndrange 的 `for` 语句。

你可以使用组合 `for` 循环来写 [独立于维度的程序](https://crowdin.com/backend/advanced/meta.md#dimensionality-independent-programming-using-grouped-indices)。

`ti.ndrange` 接收任意数量的参数。 第 `k` 个参数代表着第 `k` 维的遍历区间，且循环在每个维度的遍历区间的 [直积](https://en.wikipedia.org/wiki/Direct_product) 上进行遍历。

每个参数必须是一个整数或一个由两个整数组成的元组。
- 如果第 `k` 个参数为一个整数 `stop`，那么第 `k` 维的区间相当于 Python 中 `range(stop)` 的区间。
- 如果第 `k` 个参数为由两个整数 `(start, stop)` 组成的元组，那么第 `k` 维的区间相当于 Python 中 `range(start, stop)` 的区间。

`n` 维一般形式的基于 ndrange 的 `for` 语句的 `target_list` 必须是 `n` 个不属于当前作用域的标识符，且第 `k` 个标识符被赋以一个整数，作为第 `k` 维的循环变量。

`n` 维组合形式的基于 ndrange 的 `for` 语句的 `target_list` 必须是一个不属于当前作用域的标识符，且此标识符被赋以长度为 `n` 的 `ti.Vector`，其包含所有 `n` 维的循环变量。

当位于最外层作用域时，基于 ndrange 的 `for` 循环默认会被并行化。

#### 基于结构体的 `for` 语句

基于结构体的 `for` 语句用于遍历 Taichi field 中的活跃元素。

基于结构体的 `for` 语句的 `iter_expression` 必须是一个 Taichi field 或对 `ti.grouped(x)` 的调用，其中 `x` 是一个 `Taichi field`。
- 如果 `iter_expression` 是一个 Taichi field，那么它是一个一般形式的基于结构体的 `for` 语句。
- 如果 `iter_expression` 是一个对 `ti.grouped(x)` 的调用，其中 `x` 是一个 Taichi field，那么它是一个组合形式的基于结构体的 `for` 语句。

`n` 维 field 上的一般形式的基于结构体的 `for` 语句，其 `target_list` 必须是 `n` 个不属于当前作用域的标识符，且第 `k` 个标识符被赋以一个整数，作为第 `k` 维的循环变量。

`n` 维 field 上的组合形式的基于结构体的 `for` 语句，其 `target_list` 必须是一个不属于当前作用域的标识符，且此标识符被赋以长度为 `n` 的 `ti.Vector`，其包含所有 `n` 维的循环变量。

基于结构体的 `for` 语句必须位于 kernel 的最外层作用域，即使串行运行，`break` 语句也不能使其终止。

#### 基于静态表达式的 `for` 语句

基于静态表达式的 `for` 语句在编译期展开基于 range/ndrange 的 `for` 循环。

如果 `for` 语句的 `iter_expression` 是一个 `static_expression`，那么它是一个基于静态表达式的 `for` 语句。

`static_expression` 的 `positional_arguments` 必须满足与基于 range/ndrange 的 `for` 语句的 `iter_expression` 相同的要求。

例如：

```python
for i in ti.static(range(5)):
    print(i)
```

在编译期被展开为：

```python
print(0)
print(1)
print(2)
print(3)
print(4)
```
