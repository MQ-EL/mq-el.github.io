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

### Environment&Universes（为了文档化&交流，老外都喜欢定义名词，我是真不知道该怎么翻译。）

让我们现在脑板（脑中白板不花钱，方便省事）简单比划一下：

* *Environment*代表scala反射最抽象的认知（scala目前的反射系统首先按照运行阶段分成两类：runtime and compile-time，对应的API有相同，也有不同）。
嗯，如果是我该如何设计基本结构呢？这个想起来比较简单，OOP的继承关系就好了。例如：*Environment*在API中的表现形式是`scala.reflect.api.Universe`。不同的`Universe`代表着不同的*Environment*。
`scala.reflect.runtime.universe`对应runtime reflection，`scala.reflect.macros.Universe`对应compile-time reflection。

* 我们要操作方法字段，创建对象，探查类型等，这些 *entity* 怎么搞？OK，*entity*在反射世界里就是*Mirror*，并且*Mirror*还提供接口来操作*entity*，是不是量子纠缠？

* 我们在操作AST的时候，在抽取注解的时候，在串改类型的时候，是不是需要对应的数据结构来描述被操作对象？

* 反射包的使用都是API，用户不需要知道实现，我们需要封装内部实现。

OK，基本可以理解反射包的基本结构了：
    
    scala.reflect/
    +--api //通用用户层接口
    +--internal/ //内部实现
       +--annotations
       +--pickling
       +--settings
       +--tpe
       +--transform
       +--util
    +--io
    +--macros //compile-time用户层接口
       +--blackbox
       +--whitebox
    +runtime //runtime用户层接口
    
#### Note:

* 下载[源码](https://github.com/scala/scala/tree/2.12.x/src/reflect/scala/reflect)，然后依据readme导入Intellij：
  
  >IDE Setup
  > 
  > You may use IntelliJ IDEA (see src/intellij/README.md), the Scala IDE for Eclipse (see src/eclipse/README.md), or ENSIME (see this page on the ENSIME site).
  > 
  > In order to use IntelliJ's incremental compiler:
  > 
  > run dist/mkBin in sbt to get a build and the runner scripts in build/quick/bin
  > run "Build" - "Make Project" in IntelliJ
  > Now you can edit and build in IntelliJ and use the scripts (compiler, REPL) to directly test your changes. 
  >You can also run the scala, scalac and partest commands in sbt. Enable "Ant mode" (explained above) to 
  >prevent sbt's incremental compiler from re-compiling (too many) files before each partest invocation.