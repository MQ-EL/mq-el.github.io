---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

#### Extractor Macros

2.11.X之后才有此功能

需要提前说的是，*extractor*在[PR](https://github.com/scala/scala/pull/2848)提交之后，对返回值的要求松了一些，
本来要求返回值必须是Option[T],现在还可以返回具备如下条件的对象：

```
def isEmpty: Boolean
def get: T
```

或者返回多个：

```
  def isEmpty = false
  def get     = this
  def _1      = pre
  def _2      = sym
  def _3      = args
```

我只想说一句，2013年的pr，书上还按照Option的语法来写，怪不得看到quasiquotes的源码的时候，看着抽取器返回`Any`,太很郁闷了（知道这个知识点漏了，就是找不着官方说明文档）。
現在，我們可以這樣描述，如果是Extractor Macros的話，返回值可以是Any，compiler在macro展開之前不會檢查。

```
package scala.reflect
package api

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
      def unapply(scrutinee: Any): Any = macro ??? //!!!!!!!!!!!!!!!!!!!!!!!!神马，是誰允許返回值是Any的!!!!!!!!!!!!!
    }
    object q extends api
    object tq extends api
    object cq extends api
    object pq extends api
    object fq extends api
  }
}
```

如果不清晰，如許我粘一下愚蠢但是清晰的代碼：

```
//單獨類文件，編譯次序2
object A extends App {
  val f = new F
  f match {
    case D(x) ⇒ println("HAHA" + x) // out: HAHAF(1)
    case _ ⇒ println("WOWO")
  }
}

object D {
  def unapply(x: F): Any = macro Exa.impl
}

//單獨類文件，編譯次序2
class F {
  def isEmpty: Boolean = false
  def get = 1
}
object Exa {
  def impl(c: Context)(x: c.Tree) = {
    import c.universe._
    q"""
        new {
          class Match(x: scandra.io.F){
            def isEmpty = false
            def get = x
          }
          def unapply(x: scandra.io.F) = new Match(x)
        }.unapply($x)
      """
  }
}
```

我们实现了一个在编译阶段才确定的抽取逻辑。

#### 用例展示

目前看到的使用案例，就是在quasiquotes的實現上。当我们需要隐士的类型变换（shapeshifting）的时候，可以考虑此方法。
把每个类都可以看成某种形状的话，类之间的转换我们可以称之为*shapeshifing*。从此角度来看*polymorphic*的一个重要特征是变量或者类型的*shape*在编译期间才确定，允许shape多样性。
我感觉*shapeless*类库，目标是解放在coding过程中对shape的束缚。

有很多小例子，我们可以参考，

1. [t5903a](https://github.com/scala/scala/tree/00624a39ed84c3fd245dd9df7454d4cec4399e13/test/files/run/t5903a)
其精妙之处在于$x和$y的引用，数量可以无限多，抽取出来的类型要求是_1,_2等等数量上要一致。感觉到了shape的动态了吗？

2. [t5903b](https://github.com/scala/scala/tree/00624a39ed84c3fd245dd9df7454d4cec4399e13/test/files/run/t5903b),[t5903c](),[t5903d](https://github.com/scala/scala/tree/00624a39ed84c3fd245dd9df7454d4cec4399e13/test/files/run/t5903d)展现了对于类型多态，返回类型及，普通类型的支持。

