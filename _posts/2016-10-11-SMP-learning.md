---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

### 概述 

当某种语言可以自我审视，进而具备了自我修改的能力，我们就可以认为这种语言支持反射(reflection)了。反射的历史很长，在PO, OOP，FP等语言中都可以见到。
当某些语言在设计之初就纳入了反射当作不可以分离的特性时，还有很多其它语言在逐步的演进并强化反射的能力。

反射首先会面临到如何描述自身的问题(老外叫reify)。我们可以初步的把一段代码分为两类：

1. **static program elements.** like classs, methods, or expressions etc.
2. **dynamic program elements.** like the current continuation or execution events such as method invocations and field access etc.

其实更通常的做法是划分为两种类型：compile-time 和 runtime，依据就反射执行的阶段。compile-time是极具威力的，尤其对于开发代码转换(transformers)和生成(gennerators)两种
功能。runtime较为典型的用例是桥接语义或者组件懒绑定过程中。runtime反射不代表slower, 却可能带来更简洁的语法和运行时效率的同时提升。

static program elements的reify是在runtime，dynamic program elements的reify是在compile-time.

好了，scala在2.10之前并没有带进来自己的反射能力，只好依附java原生反射。问题也很多，例如functions, trait, case class 和 types 不能完全可逆。
以至于在反射之后，存在类型，高阶类型等变成了后缀带$的普通类，用起来工作量大，极易出错。还有一个问题就是类型擦除，java反射苍白无力。。。

好在scala对于反射一直在进化，2.10版本引入了反射库，补短java反射的基础之上，自称提供了更加强大的工具，两个方面：

1. **完全支持scala变态的类型系统**
2. **提供compile-time反射能力，scala称之为macros, 一种reify代码片段我认为这是核打击中海射能力，例如在scala中内嵌DSL。**

