---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

## 一 Overview

***

Symbols描述了名字（name)与实体（entity）例如class，method之间的联系。在scala中，任何有名字的实体都有对应的Symbol。

#### Symbols的分类及关系设计

Symbols首先分成Term和type, term下有Module和Method, Type下有Class。在实现类中分的更细。
举例：`type X = List[Int]`会被描述成一个`AliasTypeSymbol`，继承TypeSymbolApi，但不属于`ClassSymbolApi`。

在接口层，其实不算多，6个：

```scala
    trait Symbols { self: Universe =>
    
        type Symbol >: Null <: AnyRef with SymbolApi
        
        //The type of type symbols representing type, class, and trait declarations, as well as type parameters.
        type TypeSymbol >: Null <: TypeSymbolApi with Symbol
        
        //The type of term symbols representing val, var, def, and object declarations as well as packages and value parameters.
        type TermSymbol >: Null <: TermSymbolApi with Symbol
        
        //The type of method symbols representing def declarations.
        type MethodSymbol >: Null <: MethodSymbolApi with Symbol
        
        //The type of module symbols representing object declarations.
        type ModuleSymbol >: Null <: ModuleSymbolApi with Symbol
        
        //The type of class symbols representing class and trait declaration.
        type ClassSymbol >: Null <: ClassSymbolApi with Symbol
        
        type NoSymbol: Symbol
        
        //The API of symbols
        trait SymbolApi {...}
        
        //The API of term symbols
        trait TermSymbolApi {...}
        
        //The API of type symbols
        trait TypeSymbolApi {...}
        
        //The API of method symbols
        trait MethodSymbolApi extends TermSymbolApi {...}
        
        //The API of module symbols
        trait ModuleSymbolApi extends TermSymbolApi {...}
        
        //The API of class symbols
        trait　ClassSymbolApi extends TypeSymbolApi {...}
    }
```

我们暴露的不是`xxxApi`而是类型别名。
作者两个诉求点：`trait XXXApi`的继承关系要保证，接口`type XXXSymbols`继承关系要保证。第一个比较好理解，第二个实际上要求实现类要有对应的继承关系。
需要指出的是类型别名中`With Symbol`必不可少，虽然`trait XXXApi`继承关系已经描述清晰，但是在alias这一层是完全可以把`Symbol`定义成自定义子类。
因此这种写法扩展性很强，在scala reflect包中常见，是scala类型系统强大的一种体现。

* 接口定义隔离化
* API类型别名化

##### 1. Hierarchy

层级关系，举例：参数symbol的owner是方法，方法symbol的owner是class，trait或者object，class的owner是package等等，最顶层是*NoSymbol*。
注意不像官方文档说描述的，访问*NoSymbol*的owner并不会抛出异常，而是返回*NoSymbol*自身。

NoSymbol实现源码（internal/Symbols.scala）：

```scala
    class NoSymbol protected[Symbols]() extends Symbol(null, NoPosition, nme.NO_NAME) {
    ...
    
    override def owner: Symbol = {
        devWarningDumpStack("NoSymbol.owner", 15)
        this //返回自身，是不是应该向scala官方反馈一下？
    }
    ...
    }
```
    
示例代码：

```scala
   package mq.test
   
   object boot extends App {
   
     import scala.reflect.runtime.universe._
   
     var symbol = typeOf[Entice].member(TermName("showSymbolHierarchy"))
   
     while (symbol != NoSymbol) {
       println(symbol + " : " + symbol.getClass.getSuperclass)
       symbol = symbol.owner
     }
   
     class Entice { def showSymbolHierarchy(i: Int) = ??? }
   }
```

输出结果是：

    method showSymbolHierarchy : class scala.reflect.internal.Symbols$MethodSymbol
    class Entice : class scala.reflect.internal.Symbols$ClassSymbol
    object boot : class scala.reflect.internal.Symbols$ModuleClassSymbol
    package test : class scala.reflect.internal.Symbols$PackageClassSymbol
    package mq : class scala.reflect.internal.Symbols$PackageClassSymbol
    package <root> : class scala.reflect.internal.Mirrors$Roots$RootClass

画出来继承关系就看出来了：object boot是type symbol。我还是比较诧异的，没有正确理解type与term的区别。上述除了Method Symbol 其它都是type symbol。
                                                                 
                                      *TypeSymbolApi*                        
                                           |                       
                                           |                       
                          *Symbol*---------|                               
                              |            |                                         
                              |--------------------------|                                   
                              |                          |          
                    TypeSymbol(*TypeSymbol*)      *ClassSymbolApi*
                              |                          |
                              |                          |
                               ---------------------------
                                            |
                                            |
                                    ClassSymbol(*ClassSymbol*)
                                            |
                                            |
                                     ModuleClassSymbol
                                            |
                                            |
                                     PackageClassSymbol
    
#### 2. common information

`SymbolApi`包含名，类型转换，属性，函数式拼接辅助。其中关于属性可以帮助理解Symbol，例如读取类型，查看伴生类型，查看注解等。

```scala
trait SymbolApi {

    ...

    //--------names-------------
    type NameType >: Null <: Name
    
    def name: NameType
    
    def fullName: String
    
    def pos: Position
    
    //--------types-------------
    def isType: Boolean = false
    
    def asType: TypeSymbol = throw new ScalaReflectionException(s"$this is not a type")
    
    def isTerm: Boolean = false
    
    def asTerm: TermSymbol = throw new ScalaReflectionException(s"$this is not a term")
    
    def isMethod: Boolean = false
    
    def isConstructor: Boolean = false
    
    def as Method: MethodSymbol = {...}
    
    def isOverloadedMethod = false
    
    def isModule: Boolean = false
    
    def asModule: ModuleSymbol = throw new ScalaReflectionException(s"this is not a module")
    
    def isClass: Boolean = false
    
    def asClass: ClassSymbol = throw new ScalaReflectionException(s"this is not a class")
    
    def associatedField: scala.reflect.io.AbstractFile // Source file if this symbol is created during this compilation run
    
    //--------attributes-------------
    def annotations: List[Annotation]
    
    def compaion: Symbol
    
    def infoIn(site: Type): Type
    
    def info: Type
    
    def overrides: List[Symbol]
    
    def alternatives: List[Symbol]
    
    //--------tests-------------
    
    def isSymthetic: Boolean
    
    def isImplementation: Boolean
    
    def isPrivateThis: Boolean
    
    ...
    
    //--------functional-------------
    
    def orElse(alt: => Symbol): Symbol
```
    
TermSymApi等等都是一些独有的信息封装，我不多记了，不过一些东西还是比较新鲜的。类型系统在reflection中有两个方面表达：TypeSymbol和Type。
TypeSymbol描述了外域，Type描述了內域。外域的体现在：

```scala

trait TypeSymbolApi extends SymbolApi { this: TypeSymbol =>
    
    def isContravariant: Boolean
    
    def isCovariant: Boolean
    
    def isAliasType: Boolean
    
    def isAbstractType: Boolean
    
    ...
    } 
```

    
    
    
    
    
    
    
    