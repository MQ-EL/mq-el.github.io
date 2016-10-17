---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

## 一 前言

***

### 1 背景

> 话说scala 2.10带来了反射，好多积极的农民利用它开始造机器人，这样庄稼不用自己来种了。
> 后来发现造机器人的工具（scala reflect api）用的不顺手，很多零件需要自己搞（手工构造和分解AST树），需要更多的技能和知识才能造出来（需要知道AST节点如何创造，源码是如何用AST表示的）。
> 伪命题：减轻局部工作量，会引起工作总量的提升，并且需要额外的工作去梳理组织结构，尽量保持简单清晰。

2.10版带来的问题主要有两点：

* `reify`方法主要问题在于只能操作*typechecked trees*。对于没有类型检测过的tree无能为力了，因此只能借助于手动构造tree了，细节较多，需要注意的地方也多，因此需要对AST深入的了解。
* 缺乏对*tree*高层拆解（deconstruction）。因此当我们想分割一个较为复杂的树时，需要对AST直接*pattern match*，如果遇到抽取器或者通配符，情况会更加复杂。

为了解决手动构造/拆解AST带来的不方便，2.11引入了**quasiquotes**。对于AST的操作， 虽然**quasiquotes**在当前版本还是受限制，但是最终变成全能型工具。

### 2 Quasiquotes ( `Q` )

`Q`是采用SI（string interpolation）技巧来写的scala标准代码：

```scala
    q"here.comes { some => Scala(code) }"
```

返回结果是对应内部字符串的无类型AST，好比被compiler parser后的AST如下：

```scala
    Apply ( 
        Select (Ident ( TermName (" here ")), TermName (" comes ")),
        List ( 
                Function ( 
                    List (ValDef ( Modifiers ( PARAM ), TermName (" some ") , TypeTree () , EmptyTree )), 
                    Apply (
                        Ident ( TermName (" Scala ")), 
                        List ( Ident ( TermName (" code ")))
                    )    
                )
        )
    )
```

可以看到`Q`优势比较明显了，并且对于AST可以不用去深入了解就可以搞定（各位看客，AST还是要认真学习，全面深入的学习才可以进阶）。

我们还可以利用ST语法（$name 或者 ${expr}）进行变量拼接（splicing）: 

```scala
    val a = q"a quasiquote"
    q"println($a)"
```

最后一句转换成：

```scala
    Apply (
        Ident(TermName("println")),
        List(
            Select(
                Ident(TermName("a")),
                TermName("quasiquote")
            )
        ) 
    )
```

`Q`还可以进行拆解（deconstuction）匹配：

```scala
    val q"$a + $b" = q"1 + 2"
```

如果把a(b)打印出来：`a: String = Literal(Constant(1))`，到这里往外扩一点，*match case*也能用上了：

```scala
    q"foo(10)" match {
        case q"${_}($x)" => x
    }
```

### 3 Splicees and cardinality ( 老外的单词比较好用，splice改成splicee就表示被拼接者，我找不到精简词汇来翻译, 求助:( )

拼接变量（把变量塞进`Q`里）分为如下几种大类，全面的描述参考[scala之声](http://docs.scala-lang.org/overviews/quasiquotes/syntax-summary)：

* **Term and type names。** 举例：`q"def $foo = $a.$b"`，*foo* 和*b* 只能是term names, *a* 是identifier（可以标识一个tree）。
转换之后变为：`DefDef( Modifiers(), foo, List(), List(), TypeTree(), Select(Ident(a), b))`

* **Trees。** 较为自然的用法。可以把几个trees（Tree, List[Tree], List[List[Tree]]等，这里会涉及cardinality)拼接成一个tree, 
举例：`q"$foo($bar)"`可以转换成`Apply(foo, List(bar))`，当*foo*和*bar*都是Tree的时候。

* **Modifiers and Flags。** 对于构造和析构定义树比较重要（DefDef, ValDef, ClassDef等等）。*Modifiers*用来描述定义的元数据，包含*flags*和*annotations*。
*Flags*描述了访问权限，CC(case class)，laziness， abstractioness of members等等。举例：`q"$mod val $name: $tpe = $rhs"`对应着`ValDef(mods, name, tpe, rhs)`。
那么对于*Flags*到底有哪些呢，让我们展一眼源码，其中有些在源码开发阶段是禁止使用的：

```scala
package scala
package reflect
package internal //内部包，compiler也引用相关类型，可以说是souces与compiler的桥梁。

import scala.language.implicitConversions

trait FlagSets extends api.FlagSets { self: SymbolTable =>

  type FlagSet = Long
  implicit val FlagSetTag = ClassTag[FlagSet](classOf[FlagSet])

  implicit def addFlagOps(left: FlagSet): FlagOps =
    new FlagOpsImpl(left)

  private class FlagOpsImpl(left: Long) extends FlagOps {
    def | (right: Long): Long = left | right
  }

  val NoFlags: FlagSet = 0L

  object Flag extends FlagValues {
    val TRAIT         : FlagSet = Flags.TRAIT
    val INTERFACE     : FlagSet = Flags.INTERFACE
    val MUTABLE       : FlagSet = Flags.MUTABLE
    val MACRO         : FlagSet = Flags.MACRO
    val DEFERRED      : FlagSet = Flags.DEFERRED
    val ABSTRACT      : FlagSet = Flags.ABSTRACT
    val FINAL         : FlagSet = Flags.FINAL
    val SEALED        : FlagSet = Flags.SEALED
    val IMPLICIT      : FlagSet = Flags.IMPLICIT
    val LAZY          : FlagSet = Flags.LAZY
    val OVERRIDE      : FlagSet = Flags.OVERRIDE
    val PRIVATE       : FlagSet = Flags.PRIVATE
    val PROTECTED     : FlagSet = Flags.PROTECTED
    val LOCAL         : FlagSet = Flags.LOCAL
    val CASE          : FlagSet = Flags.CASE
    val ABSOVERRIDE   : FlagSet = Flags.ABSOVERRIDE
    val BYNAMEPARAM   : FlagSet = Flags.BYNAMEPARAM
    val PARAM         : FlagSet = Flags.PARAM
    val COVARIANT     : FlagSet = Flags.COVARIANT
    val CONTRAVARIANT : FlagSet = Flags.CONTRAVARIANT
    val DEFAULTPARAM  : FlagSet = Flags.DEFAULTPARAM
    val PRESUPER      : FlagSet = Flags.PRESUPER
    val DEFAULTINIT   : FlagSet = Flags.DEFAULTINIT
    val ENUM          : FlagSet = Flags.JAVA_ENUM
    val PARAMACCESSOR : FlagSet = Flags.PARAMACCESSOR
    val CASEACCESSOR  : FlagSet = Flags.CASEACCESSOR
    val SYNTHETIC     : FlagSet = Flags.SYNTHETIC
    val ARTIFACT      : FlagSet = Flags.ARTIFACT
    val STABLE        : FlagSet = Flags.STABLE
  }
}
```

额，29种啊。好了，让我们练练, 打开REPL：

```scala
    val universe = scala.reflect.runtime.universe
    import universe._
    import universe.Flag._
    
    q"$IMPLICIT val x = 1"
```

输出`res0: universe.ValDef = implicit val x = 1`。`$mod`的地方要求参数类型为*scala.reflect.runtime.universe.FlagSet*或者*...universe.Modifiers*，不是*...universe.Tree*。
那我们再展一眼*Modifiers*？

```scala
      // --- modifiers implementation ---------------------------------------
    
      /** @param privateWithin the qualifier for a private (a type name)
       *    or tpnme.EMPTY, if none is given.
       *  @param annotations the annotations for the definition.
       *    '''Note:''' the typechecker drops these annotations,
       *    use the AnnotationInfo's (Symbol.annotations) in later phases.
       */
      case class Modifiers(flags: Long,
                           privateWithin: Name,
                           annotations: List[Tree]) extends ModifiersApi with HasFlags {
    
        var positions: Map[Long, Position] = Map()
    
        def setPositions(poss: Map[Long, Position]): this.type = {
          positions = poss; this
        }
    
        /* Abstract types from HasFlags. */
        type AccessBoundaryType = Name
        type AnnotationType     = Tree
    
        def hasAnnotationNamed(name: TypeName) = {
          annotations exists {
            case Apply(Select(New(Ident(`name`)), _), _)     => true
            case Apply(Select(New(Select(_, `name`)), _), _) => true
            case _                                           => false
          }
        }
    
        def hasAccessBoundary = privateWithin != tpnme.EMPTY
        def hasAllFlags(mask: Long): Boolean = (flags & mask) == mask
        def hasFlag(flag: Long) = (flag & flags) != 0L
    
        def & (flag: Long): Modifiers = {
          val flags1 = flags & flag
          if (flags1 == flags) this
          else Modifiers(flags1, privateWithin, annotations) setPositions positions
        }
        def &~ (flag: Long): Modifiers = {
          val flags1 = flags & (~flag)
          if (flags1 == flags) this
          else Modifiers(flags1, privateWithin, annotations) setPositions positions
        }
        def | (flag: Int): Modifiers = this | flag.toLong
        def | (flag: Long): Modifiers = {
          val flags1 = flags | flag
          if (flags1 == flags) this
          else Modifiers(flags1, privateWithin, annotations) setPositions positions
        }
        def withAnnotations(annots: List[Tree]) =
          if (annots.isEmpty) this
          else copy(annotations = annotations ::: annots) setPositions positions
    
        def withPosition(flag: Long, position: Position) =
          copy() setPositions positions + (flag -> position)
    
        override def mapAnnotations(f: List[Tree] => List[Tree]): Modifiers = {
          val newAnns = f(annotations)
          if (annotations == newAnns) this
          else Modifiers(flags, privateWithin, newAnns) setPositions positions
        }
    
        override def toString = "Modifiers(%s, %s, %s)".format(flagString, annotations mkString ", ", positions)
      }
    
      object Modifiers extends ModifiersExtractor
```

看到了把，Modifier包括了`flags`, `privateWithin`, `annotations`，验证了我们之前描述。`privateWithin`是指`private/proctected[name]`中的*name*（scope qualifier）, 参考父类`HasFlags`sinppet：

```scala
  /** Access level encoding: there are three scala flags (PRIVATE, PROTECTED,
   *  and LOCAL) which combine with value privateWithin (the "foo" in private[foo])
   *  to define from where an entity can be accessed.  The meanings are as follows:
   *
   *  PRIVATE     access restricted to class only.
   *  PROTECTED   access restricted to class and subclasses only.
   *  LOCAL       can only be set in conjunction with PRIVATE or PROTECTED.
   *              Further restricts access to the same object instance.
   *
   *  In addition, privateWithin can be used to set a visibility barrier.
   *  When set, everything contained in the named enclosing package or class
   *  has access.  It is incompatible with PRIVATE or LOCAL, but is additive
   *  with PROTECTED (i.e. if either the flags or privateWithin allow access,
   *  then it is allowed.)
   *
   *  The java access levels translate as follows:
   *
   *  java private:     hasFlag(PRIVATE)                && !hasAccessBoundary
   *  java package:     !hasFlag(PRIVATE | PROTECTED)   && (privateWithin == enclosing package)
   *  java protected:   hasFlag(PROTECTED)              && (privateWithin == enclosing package)
   *  java public:      !hasFlag(PRIVATE | PROTECTED)   && !hasAccessBoundary
   */
  def privateWithin: AccessBoundaryType
```

* **Symbols**

* **Iterable of Trees** 需要特殊的*..*（基）前缀来平展。例如`q"$foo(..$bar)"`，表示`bar`是一个list[Tree]，应该平坦化之后拼接在一起，即`Apply(foo, bar)`。
如果`bar`是其他类型，编译期间会抛出异常。不同基允许在一起使用：`q"$foo(bar, ..$baz)`编译成`Apply(foo, bar :: baz)`。再细致点的例子：

```scala
    val ab = List(q"a", q"b")
    val c = q"c"
    val fabcab = q"f(..$ab, $c, ..$ab)" //fabcab: universe.Tree = f(a, b, c, a, b)
```

* **Iterable of Iterable of Trees** 当某个*function*声明多个参数（>1)，并且其中存在一个或多个List[Tree]类型的时候，非常有用：

```scala
    val argss = List(ab, List(c)) //arglists: List[List[universe.Ident]] = List(List(a, b), List(c))
    val fargss = q"f(...$argss)" //universe.Tree = f(a, b)(c)
```

* **Liftable types** `Q`提供了`Liftable`trait允许自定义实现：

```scala
    package scala
    package reflect
    package api

    /** A type class that defines a representation of `T` as a `Tree`.
       *
       *  @see [[http://docs.scala-lang.org/overviews/quasiquotes/lifting.html]]
       */
      trait Liftable[T] {
        def apply(value: T): Tree
      }
```

当然也有预定义好的，直接拿来用，举例：

```scala
    val ints = List(1, 2, 3)
    val f123 = q"f(..$ints)" // f123: universe.Tree = f(1, 2, 3)
```
    
### 4 Type `Q`

针对type也有类似的*ST*：*tq*。类似*q*可以构造析构类型树。

`val x: Foo[Bar]`中的`Foo[Bar]` ~> `tq"Foo[Bar]"` ~> `AppliedTypeTree(Ident(TypeName("Foo")), List(Ident(TypeName("Bar"))))`

调用`def foo[T] = ???`方法，`foo[Bar]` ~> `q"foo[Bar]"` ~> `TypeApply(Ident(TermName("Foo")), List(Ident(TypeName("Bar"))))`

那么我们怎么知道在源码级别，哪个应该用`tq`和`q`呢？其实在语法层面已经确定了（精通scala语法的可以考察这个点）, 源码中需要表示类型的地方都必须用*tq*。例如方法返回类型，变量类型，匿名lambda参数类型。

那么类型AST我们能创建哪些呢（10种）: *Empty Type*, *Type Identifier*, *Singleton Type*, *Type Projection*, *Applied Type*, *Annotated Type*, *Compound Type*, *Existential Type*, *Tuple Type*, *Function Type*。

今天初步记录了*Quasiquotes*的基本情况，但是有很多还未写上，例如*cq*，还有各种AST构造方法方式。我想基于目前`Q`的初步使用体验已经有了，下一步可以研究其实现策略，反过来再通读其官方文档！