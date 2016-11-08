---
layout: post
category : learning
tagline: "scala meta-programming"
tags : [scala, metaprogramming]
---
{% include JB/setup %}

#### Trees Hierarchy

Scala的AST用来描述程序代码，在编译期间或者反射期间的被操作对象。主要应用在如下三个方面：

1. Scala annotations, 用来描述注解类的构造器
2. reify，把用户层代码转换成对应的AST，后期会被Quasiquotes取代
3. runtime 和 compile time反射时，编译器会把对应的程序代码转换成AST进行操作。

总体来说，节点种类还是较为简洁的，见下图：

        |   0  |        1       |          2         |        3        |         4        |       5 
                
        Tree <-=                                                   
               |                                     |<-- TypeApply
               |                |<-- GenericApply <--|<-- Apply
               |                |                                  
               |                |<-- Super
               |                |<-- Typed                         
               |                |<-- New                           
               |                |<-- Throw                         
               |                |<-- Try                           
               |                |<-- Return                        
               |                |<-- Match                         
               |                |<-- If                            
               |                |<-- AssignOrNamedArg              
               |<-O EmptyTree   |<-- Assign                        
               |                |<-- UnApply                       
               |                |<-- Star                          
               |                |<-- Alternative                   
               |<-- TermTree <--|<-- Block                         
               |                |                                  
               |                |<--=----------------------=      
               |                    |<-- Function          |      
               |                    |<-- This              |      
               |                |<--=                      |      
               |                |                          |      
               |<-- SymTree  <--|<-- Template        |<-- LabelDef     |<-- PackageDef                    
               |                |<-- Import          |                 |                   |<-- ClassDef 
               |                |                    |                 |<-- ImplDef <------|<-- ModuleDef
               |                |<--=                |<-- MemberDef <--|                                  
               |                    |                |                 |                                  
               |                    |                |                 |<-- ValOrDefDef <--|<-- ValDef   
               |                    | <-- DefTree <--|<-- Bind         |                   |<-- DefDef   
               |                    |                                  |<-- TypeDef                      
               |                    |                                                    
               |                    | <-- RefTree <--|<-- Select                       
               |<-- NameTree <------=                |<-- Ident                                           
               |                                     |                                                    
               |                                     |<--=                                             
               |                                         |                                             
               |                                         |                                             
               |<-- TypTree  <--|<-------------------- SelectFromTypeTree                              
               |                |                       
               |                |<-- SingletonTypeTree  
               |                |<-- CompoundTypeTree   
               |                |<-- AppliedTypeTree    
               |                |<-- TypeBoundsTree     
               |                |<-- ExistentialTypeTree
               |                |<-- TypeTree           
               |<-- CaseDef                             
               |                                        
               |<-- Annotated

对于一类Trees：

* **对于DefTree和RefTree来说SymTree里的symbol.name与NameTree里的name有什么区别吗???**
应该是一样：

```
    package test
    
    import scala.reflect.runtime.universe._
    import scala.tools.reflect.ToolBox
    
    object Entice extends App {
    
      val r = reify { val x: List[Int] = Nil} 
    
      val tb = runtimeMirror(getClass.getClassLoader).mkToolBox()
    
      val ValDef = tb.typecheck(r.tree).asInstanceOf[Block].stats.head.asInstanceOf[NameTree]
    
      println(ValDef.name == ValDef.symbol.name) //true :)
    }
```

* 主要分为三大类，TermTree：描述各种术词（Term），例如方法调用（Apply），新建对象（New），赋值（Assign），TypTree：描述程序中显示声明的类型，例如List[Int]对应AppliedTypeTree, 
SymTree：描述带名字的引用和定义，例如定义一个常量（val），或者引用一个变量或者方法（Ident）。看如下示例：

```
    val x: List[Int] = Nil
    def f: Int => Int = x => x
   
    correspond to :
    
    Block(
        List(
            ValDef(
                Modifiers(), //mods
                TermName("x"), //name
                AppliedTypeTree(Select(Ident(scala.package), TypeName("List")), List(Ident(scala.Int))), //tpt
                Ident(scala.collection.immutable.Nil) //rhs: right-hand side tree
            ), 
            DefDef(
                Modifiers(), //mods
                TermName("f"), //name
                List(), //tparams: type parameters
                List(), //vparams: parameter lists of the method
                AppliedTypeTree(Ident(scala.Function1), List(Ident(scala.Int), Ident(scala.Int))), // tpt: the type ascribed to the definition
                Function(List(ValDef(Modifiers(PARAM), TermName("x"), TypeTree(), EmptyTree)), Ident(TermName("x"))) //rhs
            )
        ),
        Literal(Constant(()))
    )
```

#### AST的打印

我们看到universe混进了`scala.reflect.api.Printers`, 所有文本形式输出都在里面。
*show/showCode/toString*方法类输出的java级别源码，*showRaw*方法类输出的AST。

#### AST的遍历
如果对AST进行操作，有两种方式：1. pattern matching 和 2. 使用Traverser

* pattern matching
    这是比较自然的方式，可以取到某个特定的属性或者树节点，适合较为简单的操作：
    
```scala
    scala> val tree = Apply(Select(Apply(Select(Ident(TermName("x")), TermName("$plus")), List(Literal(Constant(2)))), TermName("$plus")), List(Literal(Constant(3))))
    tree: scala.reflect.runtime.universe.Apply = x.$plus(2).$plus(3)
    
    scala> val Apply(fun, arg :: Nil) = tree
    fun: scala.reflect.runtime.universe.Tree = x.$plus(2).$plus
    arg: scala.reflect.runtime.universe.Tree = 3
    
    scala> showRaw(fun)
    res3: String = Select(Apply(Select(Ident(TermName("x")), TermName("$plus")), List(Literal(Constant(2)))), TermName("$plus"))
```

* 某些复杂操作例如查询某种特定类型的数量，就需要遍历了。默认提供深度遍历。

Traverser定义的接口我们都可以复写，实现我们需要的功能，而且不需要自己再造遍历逻辑的轮子：

```scala
    class Traverser {
        protected[scala] var currentOwner: Symbol = rootMirror.RootClass
        
        /* Traverse someting which Trees contain, but which isn't a Tree itself. */
        def traverseName(name: Name): Unit = ()
        def traverseConstant(c: Constant): Unit = ()
        def traverseImportSelector(sel: ImportSelector): Unit = ()
        def traverseModifiers(mods: Modifiers): Unit = traverseAnnotations(mods.annotations)
        
        /* Traverse a single tree. */
        def traverse(tree: Tree): Unit = iteraverse(this, tree)
        def traversePattern(pat: Tree): Unit = traverse(pat)
        def traverseGuard(guard: Tree): Unit = traverse(guard)
        def traverseTypeAscription(tpt: Tree): Unit = traverse(tpt)
        
        /* Traverses a list of trees. */
        ...
        
        /* Traverse a list of lists of trees. */
        ...

        /* Traverses a list of trees with a given owner symbol. */
        def traverseStats(stats: List[Tree], exprOwner: Symbol) {
            stats foreach (stat =>
                if (exprOwner != currentOwner) atOwner(exprOwner)(traverse(stat))
                else traverse(stat)
            )
        }
        
        ...
    }
    
    protected def itraverse(traverser: Traverser, tree: Tree): Unit = ... //遍历逻辑在此处实现。为了模式匹配的时候，为了提升速度，匹配的都是各个节点的实现类。
```

#### 树的创建
运行时编译和macro可能都需要创建树来描述程序逻辑代码，一版有三种方式可以使用：

1. reify方法（后续会被quoasiquotes取代）
2. parse方法，在toolbox包中
3. 手动构建（不建议）

* reify，是“hygenic”的，因为转换后的AST保留了绑定信息。例如`reify(println(2)).tree).tree`转
换成`Apply(Select(Select(This(TypeName("scala")), TermName("Predef")), TermName("println")), List(Literal(Constant(2))))`。
目的是在ASTs拼接过程中，不会丢失原义；还有一种限制叫typeable，要求reify的参数必须是完整的具备类型的scala语句，不能是半句话。reify返回值类型是Expr，封装了AST，并且提供了util，例如splice，用于Exprs之间拼接。

```scala
    val x = reify(2)
    reify(println(x.splice))
```

* toolbox是强大的工具，可以用来typecheck，compile，execute。当然也可以把string解析成AST。使用的时候需要依赖scala-compile.jar库。
与reify主要的不同是没有typeable限制，`runtimeMirror(getClass.getClassLoader).mkToolBox().parse("println(2)")`返回的是
`Apply(Ident(TermName("println")), List(Literal(Constant(2))))`。println并没有限制路径，最终结果不一定是scala.Predef。某种程度上是丢失了健壮性。

AST还包含了两个比较重要的信息：Symbol和tpe。可以使用toolbox的typeCheck来填充AST，举例：

```
    val tree = reify{"test".length}.tree
    val tb = runtimeMirror(getClass.getClassLoader).mkToolBox()
    val ttree = tb.typeCheck(tree)
    ttree.tpe// tb.u.Type = Int
    ttree.symbol // tb.u.Symbol = method length
    