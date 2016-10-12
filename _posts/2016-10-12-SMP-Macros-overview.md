---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

## 一 简述

***

可以认为Compile-time metaprogramming是在编译时期构造代码的一种逻辑（算法）。

    source -> compiler AST -> binary code
                       vs
    source -> compiler AST + macro -> binary code
    |  dev  |        compile        |   runtime  |

个人认为macros打开了scalac的大门，使开发者可以在编译过程中 'CRUD' *source code*,
至于最终的效果，大神们总结归纳了很多，详见[scala flavors](http://scalamacros.org/paperstalks/2013-07-17-WhatAreMacrosGoodFor.pdf)。

目前compile-time metaprogramming已被好多语言证明极其有用，例如：

* **language virtualization.** overloading/overriding semantics of the original programming language.
* **embedding of external domain-specific languages.** tight integration of external DSLs into the host language.
* **self-optimization.** self-application of optimizations based on analysis of the program's own code.
* **boilerplate generation.** automatizing repetitive patterns which cannot be readily abstracted away by the underlying language.

好了，到现在已经很清晰了：在scala中compile-time metaprogramming叫做macros。

那么macros如何进行工作的呢？
简单说来，编译器可以认知特定的方法（例如macro修饰符），并在编译时特殊时间点调用它。目前macros设计的版本中，调用时compiler会注入一个context，两个目的：

* **提供当前的AST，**我们可以查看前缀调用树等
* **针对编译时期设计的API，**例如parsing, type-checking 和 error reporting（可以提供编译报错信息）。我们可以更改代码，或者修改类型。

简要介绍一下macro定义的几种形式（flavors）：最基本的是*def macros*普通的方法在编译时期被调用，展开。同时scala meta团队还提供了*dynamic macros*, 
*string interpolation*, *implicit macros*, *type macros* and *macro annotations*。不同的flavors包含了不同的macros语法结构和使用方式。
在设计和实现flavors过程中，scala meta 团队有一个原则：充分发挥scala的丰（变）富（态）的语法和强大的类型系统。

对于应用，通过利用5种flavors，我们变着花样的玩起来：

* **enable language virtualization**
* **can implement a form of type providers**
* **can be used to automatically generate type class instances**
* **simplify type-level programming**
* **enable embedding of external domain-specify languages**
* **re-implement non-trivial language features such as code lifting and materialization of type class instances**

社区鼓励探索更多的flavors和application，我也在不断的摸索, 努力创造新的形式（看官别笑出声啊）。

## 二 来点茴香豆

***

最简单的一种flavor: *def macros*示例：

```scala
    def assert(cond: Boolean, msg: String) = macro assertImpl
    def assertImpl(c: Context) = {
        import c.universe._
        val q"assert($cond, $msg)" = c.macroApplication
        q"if (!$cond) raise($msg)"
    }
```

举例：`assert(2 + 2 == 4, "does not compute")` 在编译完成后转换成 `if (!(2 + 2 == 4）raise("does not compute")`。
**第一个好处**是inline了（self-optimization application)。**第二个好处**是参数msg延迟计算了，因此`false`条件下可以避免计算。
当然by-name类型参数（`=>`）也可以，但是想一想它的内部实现会造成一个匿名对象生成与销毁，性能可能最差。

assert方法是门面（老外叫facade），门面有讲究的，声明部分照常，实现部分要带macro修饰符。
在编译过程中，编译器会把调用assert的地方展开成assertImpl的返回值（一段代码即AST表达式）。*def macros*中的`context`提供了`macroApplication`接口，
返回`Tree`类型的值代表调用处扩展(expansion)之前的代码片段（开发阶段，但表示为AST）。参考如下源码：

```scala
    trait Enclosures {
      self: blackbox.Context =>
    
      /** The tree that undergoes macro expansion.
       *  Can be useful to get an offset or a range position of the entire tree being processed.
       */
      def macroApplication: Tree
      ...
```      

对于`q"assert($cond, $msg)"` 有没有发现神奇的地方？string interpolator还可以用于抽取器(extractor)，长见识了，来我们看下源码：

```scala
    trait Quasiquotes { self: Universe =>
    
      /** Implicit class that introduces `q`, `tq`, `cq,` `pq` and `fq` string interpolators
       *  that are also known as quasiquotes. With their help you can easily manipulate
       *  Scala reflection ASTs.
       *
       *  @see [[http://docs.scala-lang.org/overviews/quasiquotes/intro.html]]
       */
      implicit class Quasiquote(ctx: StringContext) {
        protected trait api {
          // implementation is hardwired to `dispatch` method of `scala.tools.reflect.quasiquotes.Quasiquotes`
          // using the mechanism implemented in `scala.tools.reflect.FastTrack`
          def apply[A >: Any](args: A*): Tree = macro ???
          def unapply(scrutinee: Any): Any = macro ???
        }
        object q extends api
        object tq extends api
        object cq extends api
        object pq extends api
        object fq extends api
      }
    }
```

Compiler转换过程：`q"assert($cond, $msg) = c.macroApplication` 
~> `StringContext("assert(", ",_", ")").q(cond, msg) = c.macroApplication`
~> `Quasiquote(StringContext("assert(", ",_", ")")).q(cond,msg) = c.macroApplication`
最终调用`unapply`方法。上面注释也提到`???`被编译器硬编码分发到[Quasiquotes](scala.tools.reflect.quasiquotes.Quasiquotes)。

#### Note：
* `y.x(...) //= z`中x可以理解为method call，instance apply/unapply, object apply/unapply。参见示例：

```scala
    object Sample extends App {
    
      val i = 22
    
      val objectApply = a"apply$i"
      val valApply = b"apply$i"
      val defApply = c"apply$i"
    
      println(s"Apply test result: objectSI = ${objectApply.i}, valSI = ${valApply.i}, defSI = ${defApply.i}")
    
      val a"unapply$objectUnapply" = new U(1)
      val b"unapply$valUnapply" = new U(1)
      val c"unapply$defUnapply" = new U(1)
    
      println(s"Unapply test result: objectSI = ${objectUnapply}, valSI = ${valUnapply}, defSI = ${defUnapply}")
    
      implicit class SCTypeClass(ctx: StringContext) {    
        object a extends api    
        val b = new U(1)    
        def c = new U(1)
            
        protected trait api {
          def apply(args: Int*): U = new U(args.length)    
          def unapply(scrutinee: U): Option[Int] = Some(scrutinee.i)
        }    
      }
    
      class U(val i: Int) {
        def apply(args: Int*) = new U(args.length)    
        def unapply(b: U): Option[Int] = Some(b.i)
      }    
    } 
```

output: `Apply test result: objectSI = 1, valSI = 1, defSI = 1
         Unapply test result: objectSI = 1, valSI = 1, defSI = 1`

*  unapply返回值必须是Option或者带有isEmpty()方法的对象，对于普通代码返回Any，编译器会报错啊！！！进一步猜测macros impl可以返回其子类，这样在编译期间展开后依然合法。今天没时间测试了。
*  关于quasiquotes学习，请参考scala之声（这隐喻不好，极易误解）讲解[quasiquotes](http://docs.scala-lang.org/overviews/quasiquotes/intro.html).



