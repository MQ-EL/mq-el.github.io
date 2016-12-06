---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

#### Implicit macros

##### 如何去除Type class逻辑重复的代码 

假设我们有一个`Showable`type class定义如下：

```
trait Showable[T] { def show(x: T): String}
def show[T](x: T)(implicit s: Showable[T]) = s.show(x)
```

方法`show`在调用的时候，*scalac*会在*scope*查找符合类型要求的elegant的变量，如果找不到编译报错。

```
implicit object IntShowable extends Showable[Int]{
    def show(x: Int): String = x.toString
}

show(42)//"42"
show("42")//compilation error
```

如果我们需要对`Double`类型的`Showable`模板进行定义，发现有段代码逻辑几乎一样，搬砖的节奏又要开始了:-(

```
implicit object DoubleShowable extends Showable[String]{
    def show(x: Double): String = x.toString
}

```

老外叫：**Proliferation of boilerplate** 极其类似的样板越来越多，开始自己埋雷了，日后修改可能会带来大麻烦。对于`Type Class`，这是众所周知的现实问题。

可行的解决思路有俩种：

1. type-level programming。请参考[文章](http://typelevel.org/blog/2013/06/24/deriving-instances-1.html)。
主要思想是依据类型转换成对应的product类型，然后依据吆半裙合并每一维度数据，最后再转换回来。中间有macro，转换工作量可以省去。

2. implicit materializer。

```
trait Showable[T]{ def show(x: T): String}
object Showable {
    implicit def materializeShowable[T]: Showable[T] = macro ...
}
```

这种思路是既然我们是依据类型进行逻辑处理，因此通过上述方式，我们可以获取类型T，进而生成对应的代码，类型的抽象免去了为每一个具体类型重复编写代码的工作。

这种隐士macros还可以无缝配合已有的implicit规则，

```
implicit def listShowable[T](implicit s: Showable[T]) = 
  new Showable[List[T]] {
    def show(x: List[T]) = x.map(s.show).mkString("List(", ",", ")")
  }

show(List(42)) //prints: List(42)
```

上述定义了List的show逻辑里使用了materializer逻辑。

#### Fundep materialization, 类型推断与隐式供参之间的互作用

```
trait Iso[T, U] {
  def to(t: T) : U
  def from(u: U) : T
}

case class Foo(i: Int, s: String, b: Boolean)

def conv[C, L](c: C)(implicit iso: Iso[C, L]): L = iso.to(c)

val tp  = conv(Foo(23, "foo", true))

```

在2.10.X版本，最后一行会出错，因为L会被推断成Nothing。在2.11.X版本，进行了[改进](https://github.com/scala/scala/commit/7b890f71ecd0d28c1a1b81b7abfe8e0c11bfeb71),
大体思路是这样：typechecker尽可能推断类型，如果有未推断出类型，保留此类型符号，进入macro展开，依据展开结果，进行二次推断。
shapeless中的Generic[T]的生成也是用此方法。