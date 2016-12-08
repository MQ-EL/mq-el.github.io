---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

#### Macro Annotations

macro annotations依赖*Macro Paradise*编译器插件，在两个阶段上使用，编译注解定义和注解展开（引用）时段，之后feel free。


macro annotation带来了非常强大的源码处理能力，对标注对象进行更改，顶层及内嵌的定义都可以。

1. 修饰class，object和任意的定义，包括变量，方法及类型定义。但是类型是不可以的。
2. 允许创建或者修改对应的伴生object，这个扩展性较大。

举例：

```
import scala.reflect.macros.Context
import scala.language.experimental.macros
import scala.annotation.StaticAnnotation
import scala.annotation.compileTimeOnly

//compileTimeOnly检查identity在typer之后是否存在，如果存在则抛出异常，这样可以变相保证paradise是否正确使用。
@compileTimeOnly("enable macro paradise to expand macro annotations")
class identity extends StaticAnnotation {
  // macroTransform 和 annottees: Any* 必须一字不差。
  def macroTransform(annottees: Any*): Any = macro ??? 
}
```

*annottees*为什么是多个的原因在于：

```
@compileTimeOnly("enable macro paradise to expand macro annotations")
class Identity extends StaticAnnotation {
  def macroTransform(annottees: Any*): Any = macro IdentityMacro.impl
}

object IdentityMacro {
  def impl(c: Context)(annottees: c.Expr[Any]*): c.Expr[Any] = {
    import c.universe._
    val inputs = annottees.map(_.tree).toList

    println(inputs)
    c.Expr(q""" class D""")
  }
}
```


1. 如果我们标注了一个class，并且这个class有伴生object，他俩都会被传入。反之则不然

```
@Identity class F
object F

//output
Warning:scalac: List(class F extends scala.AnyRef {
  def <init>() = {
    super.<init>();
    ()
  }
}, object F extends scala.AnyRef {
  def <init>() = {
    super.<init>();
    ()
  }
})

```

2. 如果class/method/type member的参数被标注后，参数为被标注对象，owner，伴生对象。

```
class F(@Identity  i: Int)
object F

//output
Warning:scalac: List(<paramaccessor> private[this] val i: Int = _, class F extends scala.AnyRef {
  <paramaccessor> private[this] val i: Int = _;
  def <init>(i: Int) = {
    super.<init>();
    ()
  }
}, object F extends scala.AnyRef {
  def <init>() = {
    super.<init>();
    ()
  }
})
```

3. 如果展开之后出现了同名的class和module，那么他俩是伴生的了。
4. 有意思的是如果注解修饰的内嵌级别的元素，返回却是top级别的替换。

```
class F { def f(@Identity  i: Int)= i }
object F

//f方法没有了，多了个内嵌F$D类
```