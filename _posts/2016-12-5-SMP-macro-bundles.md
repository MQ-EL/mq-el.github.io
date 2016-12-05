---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

#### Macro Bundles

Macro bundles在2.11.X出现。
在2.10.X的时候，只能在方法级别的颗粒度定义macro implementations，例如：

```
class Queryable[T] {
 def map[U](p: T => U): Queryable[U] = macro QImpl.map[T, U]
}

object QImpl {
 //方法级别颗粒度
 def map[T: c.WeakTypeTag, U: c.WeakTypeTag]
        (c: Context)
        (p: c.Expr[T => U]): c.Expr[Queryable[U]] = ...
}
```

在compiler看到*macro*关键字后，直接调用对应的方法，直截了当，但在实践过程中发现一些不便之处：

1. 不利于模块化复杂情况下的macros。2.10.X经常看到的写法是，逻辑集中写在辅助traits里，implementations定义用来简单包皮，桥接双方。
2. macros的context参数是path-dependent，有时迫不得已implementation定义与逻辑实现写在[一起](http://docs.scala-lang.org/overviews/macros/overview.html#writing_bigger_macros),
或者需要通过refinement type或者type constructor(类型抽取出来成参数)，麻烦一点。

上述是设计Macro Bundles的直接动机。它通过类定义级别的颗粒度来解决上述问题，只需要定义构造函数参数是*Context*类型的类，那么模块化比之前容易了。

```
import scala.reflect.macros.blackbox.Context
//class级别颗粒度
class Impl(val c: Context) {
  def mono = c.literalUnit
  def poly[T: c.WeakTypeTag] = c.literal(c.weakTypeOf[T].toString)
}
object Macros {
  def mono = macro Impl.mono
  def poly[T] = macro Impl.poly[T]
}
```

在使用上和方法级别差别不大，指定好bundles的名字和方法就可以了。