---
layout: post
date: 2018-04-02T19:10:35+08:00
title: Programming Language Part B 课程笔记
tags: 读书笔记
---

本文是学习[Coursera Programming Language](https://www.coursera.org/learn/programming-languages-part-b/home/info)课程的学习笔记，文章内容及代码均取自课程材料。

# Interpreter or Compiler
实现编程语言的 workflow 如下图：

<img src="/assets/images/programming-language-b/illustration-1.png" width="800"/>

Parser 读取程序文本，检查 syntax，如果语法正确则输出 AST（abstract syntax tree）。如果该编程语言有 type checker，则将 AST 丢给 type checker 检查，通过 type check 后，就由 interpreter 或 compile 来运行程序并输出结果。

在实现编程语言 B 通常有下面两种办法：

* 使用另一种编程语言 A 来实现 interpreter（命名为 evaluator 或 executor 更恰当），输入 B 语言写的代码，输出结果
* 使用另一种编程语言 A 实现 compiler（命名为 translator 更恰当），将 B 翻译成第三种编程语言 C

# Skipping Parsing
如果基于编程语言 A 来实现编程语言 B，就可以跳过 parsing 阶段：**Have B programmers write ASTs directly in PL A**

# ML from a Racket perspective

ML is like a well-defined subset of Racket

# Racket from an ML Perspective

One way to describe Racket is that **it has “one big datatype”**：all values have this type.

我的理解由于 Racket 是 dynamic typing，所以 ML 程序员看来 Racket 只有一个类型（这样 static type check 都能成功）

* Constructors are applied implicitly (**values are tagged**)

    42 is really like Int 42
* Primitives implicitly check tags and extract data, raising errors for wrong constructors

    ```    
    fun car v = case v of Pair(a,b) => a | _ => raise …

    fun pair? v = case v of Pair _ => true | _ => false
    ```

# Weak Typing
There exist programs that, by definition, must pass static checking but then when run can "set the computer on fire"?

* Ease of language implementation: Checks left to the programmer
* Performance: Dynamic checks take time
* Lower level: Compiler does not insert information like array sizes, so it cannot do the checks

Racket is not weakly typed

* It just checks most things dynamically*
* Dynamic checking is the definition – if the implementation can analyze the code to ensure some checks are not needed, then it can optimize them away