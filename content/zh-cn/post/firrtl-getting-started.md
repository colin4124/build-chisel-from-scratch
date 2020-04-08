---
title: "第二章：开始构建 Firrtl"
date: 2019-12-17T20:26:33+08:00
draft: false
---

上一篇讲如何构建基本 Chisel 框架，生成二选一多路选择器的 **FIRRTL**。本章也从零构建 **Firrtl** ，来满足生成对应 **Verilog** 的需求。

# Mill 环境搭建

删掉原来的 **firrtl** 依赖，增加 `firrtl` 的 `CommonChiselModule` 并让 `chiselFrontend` 和 `chisel3` 的模块依赖 `moduleDeps` 都依赖它。 完整的 `build.sc` 如下

```scala
import ammonite.ops._

import mill._
import mill.scalalib._

trait CommonChiselModule extends ScalaModule {
  def scalaVersion = "2.12.10"
}

object chisel3 extends CommonChiselModule {
  object firrtl extends CommonChiselModule
  object chiselFrontend extends CommonChiselModule {
    def moduleDeps = Seq(firrtl)
  }

  def moduleDeps = Seq(firrtl, chiselFrontend)
  def millSourcePath = super.millSourcePath / ammonite.ops.up
}
```

# Firrtl IR

```shell
$ mkdir -p firrtl/src/ir
$ touch firrtl/src/ir/IR.scala
```

新建存放 **FIRRTL** **IR** 相关代码的 `IR.scal` 文件。

```scala
// play-chisel/firrtl/src/ir/IR.scala
package firrtl
package ir

abstract class FirrtlNode

trait HasName {
  val name: String
}

trait IsDeclaration extends HasName
```

`FirrtlNode` 是 **IR** 的一个基本（根）节点，每种类型的 **IR** 都继承它。

`HasName` 特质定义了字符串类型的名字属性。

`IsDeclaration` 特质归类那些用于声明的 **IR** 类型。

## 基本操作

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class PrimOp extends FirrtlNode
```

具体的实现是在 `play-chisel/firrtl/src/PrimOps.scala`

```shell
$ touch firrtl/src/PrimOps.scala
```

```scala
// play-chisel/firrtl/src/PrimOps.scala
package firrtl

import firrtl.ir._

object PrimOps {
  case object Not extends PrimOp { override def toString = "not" }
  case object And extends PrimOp { override def toString = "and" }
  case object Or extends PrimOp { override def toString = "or" }

  private lazy val builtinPrimOps: Seq[PrimOp] =
    Seq(Not, And, Or)
  private lazy val strToPrimOp: Map[String, PrimOp] = {
    builtinPrimOps.map { case op : PrimOp=> op.toString -> op }.toMap
  }
  def fromString(op: String): PrimOp = strToPrimOp(op)
}
```

这里实现了三个位操作：与或非。

`builtinPrimOps` 数组是存放了定义好的基本操作，`strToPrimOp` 把定义好的基本操作对象跟它的字符串表示关联成映射关系 **Map**，`fromString` 根据字符串名字找到对应的基本操作对象。

## 表达式

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class Expression extends FirrtlNode {
  def tpe: Type
}

case class Reference(name: String, tpe: Type) extends Expression with HasName
case class DoPrim(op: PrimOp, args: Seq[Expression], consts: Seq[BigInt], tpe: Type) extends Expression
```

这里定义了两种表达式：引用和运算。

引用类型包含了引用名字和引用类型两个信息，运算表达式包含了操作符、参数、常量和类型。

## 语句
```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class Statement extends FirrtlNode

case class DefNode(name: String, value: Expression) extends Statement with IsDeclaration
case class Block(stmts: Seq[Statement]) extends Statement
case class Connect(loc: Expression, expr: Expression) extends Statement
```

这里定义了三种语句：节点语句、语句块和连接语句。

节点语句包含了名字和对应的表达式，语句块包含语句列表，连接语句包含了连接的左值和右值表达式。

## 宽度

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class Width extends FirrtlNode

object IntWidth {
  private val maxCached = 1024
  private val cache = new Array[IntWidth](maxCached + 1)
  def apply(width: BigInt): IntWidth = {
    if (0 <= width && width <= maxCached) {
      val i = width.toInt
      var w = cache(i)
      if (w eq null) {
        w = new IntWidth(width)
        cache(i) = w
      }
      w
    } else new IntWidth(width)
  }
}
class IntWidth(val width: BigInt) extends Width
case object UnknownWidth extends Width
```

`object IntWidth` 会缓存住 0 到 1024 的数值，这样每次遇到这个范围的数值，只会创建一次对应宽度数值的 `class IntWidth`。

`UnknownWidth` 没有宽度值。

## 类型

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class Type extends FirrtlNode
abstract class GroundType extends Type

case class UIntType(width: Width) extends GroundType
case object UnknownType extends Type
```

这里定义了两种类型：整型类型和未知类型。

## 方向

```scala
// play-chisel/firrtl/src/ir/IR.scala
sealed abstract class Direction extends FirrtlNode

case object Input extends Direction
case object Output extends Direction
```

这里定义了两种方向：输入和输出。

## 端口
```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Port(
  name      : String,
  direction : Direction,
  tpe       : Type
) extends FirrtlNode with IsDeclaration
```

端口包含了端口的名字、方向和类型。

## 模块

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class DefModule extends FirrtlNode with IsDeclaration {
  val name : String
  val ports : Seq[Port]
}

case class Module(name: String, ports: Seq[Port], body: Statement) extends DefModule
```

`DefModule` 是基本的模块类型，包含模块的名字和端口列表。模块分为两种：包含模块体定义的一般模块，以及不包含模块体定义的外部模块（这里先不做介绍）。

## 电路

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Circuit(modules: Seq[DefModule], main: String) extends FirrtlNode
```

电路包含模块列表和模块的名字。

# 把 Chisel IR 转成 Firrtl IR


**Chisel** 默认既生成 **FIRRTL** `.fir` 文件也会生成 **Verilog** `.v` 文件。但 **Chisel** 并不是调用 **FIRRTL** 解析 `.fir` 文件来生成 **Verilog** 文件的，而是内部把 **Chisel** 的电路 **IR** 转成 **Firrtl** 的电路 **IR**，省去了 **FIRRTL** 读取 `.fir` 文件再解析成 **Firrtl** 的电路 **IR** 过程。

```shell
$ touch chiselFrontend/src/internal/firrtl/Converter.scala
```

新建 `Converter.scala` 文件。

```scala
// play-chisel/chiselFrontend/src/internal/firrtl/Converter.scala
package chisel3.internal.firrtl

import scala.annotation.tailrec
import scala.collection.immutable.Queue

import chisel3._
import firrtl.{ir => fir}

object Converter {
  def convert(circuit: Circuit): fir.Circuit =
    fir.Circuit(circuit.components.map(convert), circuit.name)
  ...
}
```

`import firrtl.{ir => fir }` 改名是为了跟 **Chisel** 的 `ir` 区分开来。

`def convert(circuit: Circuit)` 方法是将 **Chisel** 的电路 **IR** 转成 **Firrtl** 的电路 **IR**。

`circuit.components.map(convert)` 是遍历 **Chisel** 的模块 **IR**，一一转成 **Firrtl** 的模块 **IR**，存到 **Firrtl** 的电路 **IR** `fir.Circuit` 中。

```scala
// play-chisel/chiselFrontend/src/internal/firrtl/Converter.scala
object Converter {
  def convert(component: Component): fir.DefModule = component match {
    case ctx @ DefModule(_, name, ports, cmds) =>
      fir.Module(name, ports.map(p => convert(p)), convert(cmds.toList))
  }
  ...
}
```

`def convert(component:Component)` 方法是将 **Chisel** 的模块 **IR** 转成 **Firrtl** 的模块 **IR**。

目前 **Chisel** 的模块 **IR** 类型只有 **DefModule** 一种。把它的名字、端口列表和命令列表提取出来，分别转换成对应的 **Firrtl**  **IR**。

```scala
// play-chisel/chiselFrontend/src/internal/firrtl/Converter.scala
object Converter {
  def convert(port: Port, topDir: SpecifiedDirection = SpecifiedDirection.Unspecified): fir.Port = {
    val resolvedDir = SpecifiedDirection.fromParent(topDir, port.dir)
    val dir = resolvedDir match {
      case SpecifiedDirection.Unspecified | SpecifiedDirection.Output => fir.Output
      case SpecifiedDirection.Flip | SpecifiedDirection.Input => fir.Input
    }
    val tpe = extractType(port.id)
    fir.Port(port.id.getRef.name, dir, tpe)
  }

  def extractType(data: Data): fir.Type = data match {
    case d: UInt => fir.UIntType(convert(d.width))
  }
  ...
}
```

`def convert(port: Port...)` 方法是将 **Chisel** 的端口 **IR** 转成 **Firrtl** 的端口 **IR**。

前面调用该方法的时候 `topDir` 的参数没有给，默认是 `SpecifiedDirection.Unspecified`。

`val dir` 是端口在 **Firrtl** 这边的端口方向。

`val tpe` 是端口在 **Firrtl** 这边的类型。

`def extractType` 方法根据数据的类型和方向得到 **Firrtl** 的类型 **IR**。

`fir.Port` 记录下端口的名字、方向和类型。

```scala
// play-chisel/chiselFrontend/src/internal/firrtl/Converter.scala
object Converter {
  def convert(width: Width): fir.Width = width match {
    case UnknownWidth() => fir.UnknownWidth
    case KnownWidth(value) => fir.IntWidth(value)
  }
  ...
}
 ```
 
 `def convert(width: Width)` 方法是将 **Chisel** 的宽度 **IR** 转成 **Firrtl** 的宽度 **IR**。

```scala
// play-chisel/chiselFrontend/src/internal/firrtl/Converter.scala
import scala.annotation.tailrec
import scala.collection.immutable.Queue

object Converter {
  def convert(cmds: Seq[Command]): fir.Statement = {
    @tailrec
    def rec(acc: Queue[fir.Statement])(cmds: Seq[Command]): Seq[fir.Statement] = {
      if (cmds.isEmpty) {
        acc
      } else convertSimpleCommand(cmds.head) match {
        case Some(stmt) =>
          rec(acc :+ stmt)(cmds.tail)
        case None => rec(acc)(cmds)
      }
    }
    fir.Block(rec(Queue.empty)(cmds))
  }

  def convertSimpleCommand(cmd: Command): Option[fir.Statement] = cmd match {
    case e: DefPrim[_] =>
      val args = e.args map convert
      val expr = fir.DoPrim(convert(e.op), args, Seq(), fir.UnknownType)
      Some(fir.DefNode(e.name, expr))
    case Connect(loc, exp) =>
      Some(fir.Connect(convert(loc), convert(exp)))
    case _ => None
  }
  ...
}
```

`def convert(cmds: Seq[Command])` 方法是将 **Chisel** 的命令 **IR** 列表一一转成 **Firrtl** 的语句 **IR**。

`def rec` 是一个递归函数，递归调用 `convertSimpleCommand` 方法转换每一个 **Chisel** 的命令 **IR**，然后添加到 `acc` 队列里，最后返回该队列。

`def convertSimpleCommand` 方法是将每个 **Chisel** 的命令 **IR** 转成 **Firrtl** 的语句 **IR**。

目前这里的 **Chisel** 命令 **IR** 只有运算 `DefPrim` 和连接 `Connect` 两种。

`DefPrim` 的会先把参数、操作类型转成对应的 **Firrtl** 的参数、操作类型 **IR**，然后再转成 **Firrtl** 的表达式 **IR**，最后再放到 **Firrtl** 的 `DefNode` **IR** 里。 **注意：** **Chisel** 的 `DefPrim` 命令 **IR** 对应到 **FIRRTL** 里的是一条 **DoPrim** 表达式 **IR** ，需要放在 **DefNode** 才成为一条 **Firrtl** 的语句 **IR**。

`Connect` 的会把左、右值转成对应的 **FIRRTL** 的表达式 **IR**，最后再放到 **Firrtl** 的 `Connect` **IR** 里。

```scala
// play-chisel/chiselFrontend/src/internal/firrtl/Converter.scala
object Converter {
  def convert(op: PrimOp): fir.PrimOp = firrtl.PrimOps.fromString(op.name)

  def convert(arg: Arg): fir.Expression = arg match {
    case Node(id) =>
      convert(id.getRef)
    case Ref(name) =>
      fir.Reference(name, fir.UnknownType)
    case ModuleIO(mod, name) =>
      fir.Reference(name, fir.UnknownType)
  }
}
```

`def convert(op: PrimOp)` 方法是将 **Chisel** 的运算符 **IR** 转成 **Firrtl** 的 **IR** 。

`def convert(arg: Arg` 方法是将 **Chisel** 的参数转成 **Firrtl** 的表达式。

`Node` 是将携带的 **Chisel** 元素提前出来再进行转换。

`Ref` 和 `ModuleIO` 则都是转成 **Firrtl** 的 `Reference` 引用类型 **IR**。

```scala
// play-chisel/src/Main.scala
object Main extends App {
  val (circuit, _) = Builder.build(Module(new Mux2))
  ...
  val firrtl = Converter.convert(circuit)
  println(firrtl)
}
```

调用 `Converter.convert` 方法把 **Chisel** 的电路 **IR** 转换成 **Firrtl**，运行输出看下效果吧。

```shell
$ mill chisel3.run
```
{{% admonition tip 源码01 %}}
{{% /admonition %}}

[play-chisel/tree/chap02-01](https://github.com/colin4124/play-chisel/tree/chap02-01)

# 美化输入的 IR 格式

```shell
$ touch src/PrintIR.scala
```

```scala
// play-chisel/src/PrintIR.scala
package playchisel

import firrtl.{ir => fir}

object PrintIR {
  def tab(l: Int) = (" " * 2) * l
  def m_str(name: String) = s"Module: ${name}"
  def p_str(p: fir.Port) = s"Port: ${p.name} ${dir_str(p.direction)} ${type_str(p.tpe)}"
  def dir_str(d: fir.Direction) = d match {
    case fir.Input  => "Input"
    case fir.Output => "Output"
  }
  def type_str(d: fir.Type) = d match {
    case fir.UIntType(w)  => s"UInt(${w_str(w)})"
    case fir.UnknownType => "UnknownType"
  }
  def w_str(w: fir.Width) = w match {
    case fir.IntWidth(i) => s"$i"
    case fir.UnknownWidth => "UnknownWidth"
  }
  def stmt_str(s: fir.Statement, l: Int): String = {
    s match {
      case fir.DefNode(n, v) => s"${tab(l)}DefNode: $n\n${tab(l+1)}${e_str(v, l+1)}"
      case fir.Block(s) => s"${tab(l)}Block\n" + (s map { x => stmt_str(x, l+1) } mkString "\n")
      case fir.Connect(left, right) =>
        s"${tab(l)}Connect\n${tab(l+1)}${e_str(left, l+1)} ${e_str(right, l+1)}"
    }
  }
  def e_str(e: fir.Expression, l: Int): String = {
    e match {
      case fir.Reference(n, t) => s"Reference(${n}: ${type_str(t)})"
      case fir.DoPrim(op, args, const, tpe) =>
        s"DoPrim\n${tab(l+1)}${op}\n${tab(l+1)}"      +
        (args map {x => e_str(x, l+1)} mkString ", ") +
        s"\n${tab(l+1)}${type_str(tpe)}"
    }
  }

  def print_fir(ast: fir.Circuit) = {
    println(s"Circuit: ${ast.main}")
    ast.modules foreach { m =>
      print(tab(1))
      println(m_str(m.name))
      m.ports foreach { p =>
        print(tab(2))
        println(p_str(p))
      }
      m match {
        case fir.Module(_, _, body) => println(stmt_str(body, 2))
        case _ =>
      }
    }
  }
}
```

```scala
// play-chisel/src/Main.scala
import PrintIR._

object Main extends App {
  ...
  print_fir(firrtl)
}
```

```scala
// play-chisel/firrtl/src/ir/IR.scala
object IntWidth {
  ...
  def unapply(w: IntWidth): Option[BigInt] = Some(w.width)
}
```

```shell
$ mill chisel3.run
```
{{% admonition tip 源码02 %}}
{{% /admonition %}}

[play-chisel/tree/chap02-02](https://github.com/colin4124/play-chisel/tree/chap02-02)

# 搭建 Firrtl 基本框架

```shell
$ touch firrtl/src/Utils.scala
```

```scala
// play-chisel/firrtl/src/Utils.scala

package firrtl

object Utils {
  def getThrowable(maybeException: Option[Throwable], first: Boolean): Throwable = {
    maybeException match {
      case Some(e: Throwable) => {
        val t = e.getCause
        if (t != null) {
          if (first) {
            getThrowable(Some(t), first)
          } else {
            t
          }
        } else {
          e
        }
      }
      case None | null => null
    }
  }

  def throwInternalError(message: String = "", exception: Option[Exception] = None) = {
    val throwable = getThrowable(exception, true)
    val string = if (message.nonEmpty) message + "\n" else message
    error("Internal Error! %sPlease file an issue at https://github.com/ucb-bar/firrtl/issues".format(string), throwable)
  }

  def error(str: String, cause: Throwable = null) = throw new FirrtlInternalException(str, cause)
}
```

```shell
$ touch firrtl/src/FirrtlException.scala
```

```scala
// play-chisel/firrtl/src/FirrtlException.scala
package firrtl

private[firrtl] class FirrtlInternalException(message: String, cause: Throwable = null)
    extends Exception(message, cause)
```

修改下 `Main.scala`，调用 `Firrtl` 的 `VerilogCompiler` 把 **Firrtl** **IR** 表示的电路转成 **Verilog**。

```scala
import firrtl.{VerilogCompiler, CircuitState, ChirrtlForm}

import java.io.{File, FileWriter}

object Main extends App {
  ...
  val state = CircuitState(firrtl, ChirrtlForm)
  val compiler = new VerilogCompiler
  val res = compiler.compile(state)
}
```

`CircuitState` 是存放 **Firrtl** 电路 **IR** 和当前电路 **IR** 处于哪个编译阶段的状态。

`compiler.compile` 方法是将其编译成 **Verilog**。

## 电路状态

```shell
$ touch firrtl/src/Compiler.scala
```

```scala
// play-chisel/firrtl/src/Compiler.scala

package firrtl

import firrtl.ir.Circuit
import firrtl.Utils.throwInternalError
import firrtl.options.TransformLike

case class CircuitState(
  circuit: Circuit,
  form: CircuitForm,
)

sealed abstract class CircuitForm(private val value: Int) extends Ordered[CircuitForm] {
  def compare(that: CircuitForm): Int = this.value - that.value
}

final case object ChirrtlForm extends CircuitForm(value = 3)
final case object HighForm extends CircuitForm(2)
final case object MidForm extends CircuitForm(1)
final case object LowForm extends CircuitForm(0)
final case object UnknownForm extends CircuitForm(-1) {
  override def compare(that: CircuitForm): Int = { sys.error("Illegal to compare UnknownForm"); 0 }
}
```

`CircuitState` 包含了电路 **IR** `circuit: Circuit` 和电路当前的状态 `form: CircuitForm` 。

电路的状态是可以比较的对象，因为继承了 `Ordered[CircuitForm]`，比较的方法是对比内部的整型数值 `private val value: Int`。

高级的形式是低级形式语法的超集，高级往低级转成的过程加入更多的限制条件。由大到小、由高级的形式到低级的 **Verilog** 形式依次是：

- `ChirrtlForm` **Chisel** 生成的 **IR**；
- `HighForm` 由 `ChirrtlForm` 转成的标准 **FIRRTL** 语法，去掉了 **Chisel** 扩展的语法；
- `MidForm` 比 `HighForm` 多加了三个限制 **所有的宽度必须显式地标明**、**所有的 when 语句必须移除** 和 **每个元素只有一个赋值来源**；
- `LowForm` 比 `MidForm` 多加了两个限制 **所有的集合类型 vector/bundle 必须移除** 和 **所有隐氏的截位必须显式**。
- `UnknownForm` 是个特殊的形式，表明当前的操作不会改变当前的形式。

因此 `UnknownForm` 是不具备比较性的。

## Compiler

```shell
$ touch firrtl/src/LoweringCompiler.scala
```

```scala
// play-chisel/firrtl/src/LoweringCompiler.scala

package firrtl

import CompilerUtils.getLoweringTransforms

class VerilogCompiler extends Compiler {
  val emitter = new VerilogEmitter
  def transforms: Seq[Transform] = getLoweringTransforms(ChirrtlForm, LowForm) ++
    Seq(new LowFirrtlOptimization) ++
    Seq(emitter)
}
```

`VerilogCompiler` 继承自 `Compiler`，需要经过一系列的转换 `def transforms`，最后再发射（生成) **Verilog** `Seq(emitter)`。

**Chisel** 生成的 **FIRRTL IR** 是 **ChirrtlForm**，通过 `getLoweringTransforms` 转成 **LowForm**。然后再经过 `LowFirrtlOptimization` 优化处理后，由 `emitter` （也就是 `VerilogEmitter`）生成 **Verilog**。

看下 `Compiler`。

```scala
// play-chisel/firrtl/src/Compiler.scala
trait Compiler {
  def emitter: Emitter
  def transforms: Seq[Transform]

  require(transforms.size >= 1,
          s"Compiler transforms for '${this.getClass.getName}' must have at least ONE Transform! " +
            "Use IdentityTransform if you need an identity/no-op transform.")

  def compile(state: CircuitState): CircuitState = {
    val finalState = transforms.foldLeft(state) { (in, xform) =>
      xform.runTransform(in)
    }
    finalState
  }
}
```

`Compiler` 规定了子类必须要有 `def emitter` 生成最终的文件和 `def transforms` 一系列的转换，并且不能为空，至少得有一个。

`def compile(state: CircuitState)` 方法是将当前的电路状态 `state` 按顺序地一个传给下一个 `transforms` 列表里具体的转换对象，返回最终的状态。

接下来看下 `Transform` 是怎么定义的，转成的过程如何。

## 转换

```shell
$ mkdir -p firrtl/src/options
$ touch firrtl/src/options/Phase.scala
```

```scala
//  play-chisel/firrtl/src/options/Phase.scala
package firrtl.options

trait TransformLike[A] extends {
  def name: String

  def transform(a: A): A
}
```
`trait TransformLike` 是一个转换的基类，它不指定具体的类型，而是泛型变量 `A`。

`transform` 方法是将 `a: A` 处理过之后，尽管改变了其中的内容，但还是返回同样的类型。

```scala
// play-chisel/firrtl/src/Compiler.scala
abstract class Transform extends TransformLike[CircuitState] {
  def name: String = this.getClass.getName
  def inputForm: CircuitForm
  def outputForm: CircuitForm

  protected def execute(state: CircuitState): CircuitState
  def transform(state: CircuitState): CircuitState = execute(state)

  private[firrtl] def prepare(state: CircuitState): CircuitState = state

  final def runTransform(state: CircuitState): CircuitState = {
    val result = execute(prepare(state))

    CircuitState(result.circuit, result.form)
  }
}
trait Emitter extends Transform {
  def outputSuffix: String
}
```

`Transform` 是将一个电路状态 `CircuitState` 转换成另一个电路状态。

`inputForm` 和 `outputForm` 是指定转换的输入和输出的形式分别是什么。

`def runTransform` 、`def execute`和`def prepare` 都是一些进行转换的方法，只是执行顺序不同。

`Emitter` 是一种特殊的转换，一般是放在最后生成文件。

```scala
// play-chisel/firrtl/src/Compiler.scala
trait SeqTransformBased {
  def transforms: Seq[Transform]
  protected def runTransforms(state: CircuitState): CircuitState =
    transforms.foldLeft(state) { (in, xform) => xform.runTransform(in) }
}
abstract class SeqTransform extends Transform with SeqTransformBased {
  def execute(state: CircuitState): CircuitState = {
    val ret = runTransforms(state)
    CircuitState(ret.circuit, outputForm)
  }
}
```

`SeqTransform` 是转换序列，经过封装之后，`def execute` 就是执行了一系列放在 `def transforms` 里的转换对象。

```scala
// play-chisel/firrtl/src/Compiler.scala
object CompilerUtils {
  def getLoweringTransforms(inputForm: CircuitForm, outputForm: CircuitForm): Seq[Transform] = {
    if (outputForm >= inputForm) {
      Seq.empty
    } else {
      inputForm match {
        case ChirrtlForm =>
          Seq(new ChirrtlToHighFirrtl) ++ getLoweringTransforms(HighForm, outputForm)
        case HighForm =>
          Seq(new IRToWorkingIR, new ResolveAndCheck, new HighFirrtlToMiddleFirrtl) ++
              getLoweringTransforms(MidForm, outputForm)
        case MidForm => Seq(new MiddleFirrtlToLowFirrtl) ++ getLoweringTransforms(LowForm, outputForm)
        case LowForm => throwInternalError("getLoweringTransforms - LowForm")
        case UnknownForm => throwInternalError("getLoweringTransforms - UnknownForm")
      }
    }
  }
}
```

`object CompilerUtils` 里的 `getLoweringTransforms` 是根据当前的输入和输出形式，生成一些转换对象，需要经过它们才能完成转换。

`ChirrtlForm` 需要经过 `ChirrtlToHighFirrtl` 才能到 `HighForm` 形式，`HighForm` 需要经过 `IRToWorkingIR`、`ResolveAndCheck` 和 `HighFirrtlToMiddleFirrtl` 才能到 `MidForm` 形式；`MidForm` 需要经过 `MiddleFirrtlToLowFirrtl` 才能到 `LowForm` 形式。

当输出形式的级别大于等于输入的级别时就停下来了。

这里罗列下这些形式的定义，有些形式的 `def transforms = Seq()` 表示暂时不实现转换，先占个位置，没有起到实际的作用。

```scala
// play-chisel/firrtl/src/LoweringCompiler.scala
sealed abstract class CoreTransform extends SeqTransform

class ChirrtlToHighFirrtl extends CoreTransform {
  def inputForm = ChirrtlForm
  def outputForm = HighForm
  def transforms = Seq()
  // def transforms = Seq(
  // passes.CInferTypes,
  // )
}
class IRToWorkingIR extends CoreTransform {
  def inputForm = HighForm
  def outputForm = HighForm
  def transforms = Seq()
  // def transforms = Seq(passes.ToWorkingIR)
}
class ResolveAndCheck extends CoreTransform {
  def inputForm = HighForm
  def outputForm = HighForm
  def transforms = Seq()
  // def transforms = Seq(
  //   passes.ResolveKinds,
  //   passes.ResolveFlows,
  // )
}
class HighFirrtlToMiddleFirrtl extends CoreTransform {
  def inputForm = HighForm
  def outputForm = MidForm
  def transforms = Seq()
}
class MiddleFirrtlToLowFirrtl extends CoreTransform {
  def inputForm = MidForm
  def outputForm = LowForm
  def transforms = Seq()
}
class LowFirrtlOptimization extends CoreTransform {
  def inputForm = LowForm
  def outputForm = LowForm
  def transforms = Seq()
}
```

`CoreTransform` 这里只是 `SeqTransform` 的一个别称，可以看作是一个分组，每个分类里包含具体的转换对象 `def transforms`。

目前这里仅介绍 `ChirrtlToHighFirrtl` 分类下的 `passes.CInferTypes`，`IRToWorkingIR` 分类下的 `passes.ToWorkingIR` 和 `ResolveAndCheck` 分类下的 `passes.ResolveKinds` 和 `passes.ResolveFlows`。（先注释掉，后面讲到对应的时候再拿掉注释）。

```shell
$ mkdir -p firrtl/src/passes
$ touch firrtl/src/passes/Passes.scala
```

```scala
// play-chisel/firrtl/src/passes/Passes.scala
package firrtl.passes

import firrtl._
import firrtl.ir._
import firrtl.Utils._
import firrtl.Mappers._

trait Pass extends Transform {
  def inputForm: CircuitForm = UnknownForm
  def outputForm: CircuitForm = UnknownForm
  def run(c: Circuit): Circuit
  def execute(state: CircuitState): CircuitState = {
    val result = (state.form, inputForm) match {
      case (_, UnknownForm) => run(state.circuit)
      case (UnknownForm, _) => run(state.circuit)
      case (x, y) if x > y =>
        error(s"[$name]: Input form must be lower or equal to $inputForm. Got ${state.form}")
      case _ => run(state.circuit)
    }
    CircuitState(result, outputForm)
  }
}
```

```shell
$ touch firrtl/src/Emitter.scala
```

```scala
// play-chisel/firrtl/src/Emitter.scala
package firrtl

class VerilogEmitter extends SeqTransform with Emitter {
  def inputForm = LowForm
  def outputForm = LowForm
  val outputSuffix = ".v"
  def transforms = Seq()
}
```

## IR 的迭代

先以一个小的例子讲解隐式转换，后面会需要到这方面的知识。

随便在哪里新建一个文件 `foo.scala`，写入以下内容：

```scala
object Foo extends App {
  class Bar(val x: Int)

  def printBar(b: Bar) = println(b.x)

  object Car {
    implicit def fromInt(i: Int) = new Bar(i)
  }

  printBar(10)
}
```

运行：

```shell
$ scala foo.scala
```

输出以下错误信息：

```shell
foo.scala:10: error: type mismatch;
 found   : Int(10)
 required: Main.Bar
  printBar(10)
           ^
```

可以看到，当隐式转换放在一个 `object` 里，和 `printBar(10)` 作用域就不在一起了，编译器也就无法找到隐式声明进行隐式转换。

这次做个小修改，把 `object Car` 改成 `object Bar`。再运行一遍，发现结果对了，输出了 `10`。

```scala
10
```

发现编译器在寻找隐式转换的时候，会去转换到的目标类型的 `object` 作用域里寻找。这个例子里，`printBar` 的参数类型是 `Bar`，而传递的参数是 `Int` 类型，编译器会去寻找把 `Int` 转成 `Bar` 类型的隐式函数。它会去 `Bar` 类型的 `object Bar` 里面寻找，找到了 `implicit def fromInt` 方法。

再扩展个稍微复杂的例子。

对自定义类 `Statement` 重载 `map` 方法。`Statement` 类有两个属性，`Int` 整型类型表示的数值 `num`和 `String` 字符串类型表示的字符串 `str`。 重载的 `map` 方法根据传给 `map` 方法的参数类型，自己找到对应的属性进行操作，例如针对 `Int` 类型的函数就改变 `num` 属性，针对 `String` 类型的函数就改变 `str` 属性。重载的 `map` 方法不在 `Statement` 类里面定义，而是通过隐式类的方式。


```scala
object Foo extends App {
  def addTwo(a: Int): Int = a + 2
  def addStr(a: String): String = a + " world"

  case class Statement(num: Int, str: String) {
    def mapInt(f: Int => Int): Statement = this.copy(num = f(this.num))
    def mapStr(f: String => String): Statement = this.copy(str = f(this.str))
  }

  private trait StmtMagnet {
    def map(stmt: Statement): Statement
  }
  private object StmtMagnet {
    implicit def forInt(f: Int => Int): StmtMagnet = new StmtMagnet {
      override def map(stmt: Statement): Statement = stmt mapInt f
    }
    implicit def forStr(f: String => String): StmtMagnet = new StmtMagnet {
      override def map(stmt: Statement): Statement = stmt mapStr f
    }
  }
  implicit class StmtMap(val _stmt: Statement) extends AnyVal {
    def map[T](f: T => T)(implicit magnet: (T => T) => StmtMagnet): Statement = magnet(f).map(_stmt)
  }

  val stmt = Statement(5, "Hello")

  println("Before: ")
  println(stmt)
  val newStmt = stmt map addTwo map addStr
  println("After: ")
  println(newStmt)
}
```

`case class Statement(num: Int, str: String)` 类除了定义两个属性之外，还分别定义了针对这个两个属性进行操作的方法 `def mapInt` 和 `def mapStr`。

`def mapInt` 方法用函数 `f` 改变 `num` 属性，`def mapStr` 方法用函数 `f` 改变 `str` 属性。

```scala
implicit class StmtMap(val _stmt: Statement) extends AnyVal {
  def map[T](f: T => T)(implicit magnet: (T => T) => StmtMagnet): Statement = magnet(f).map(_stmt)
}
```

当执行 `stmt map addTwo` 的时候， `Statement` 类型没有定义 `map` 方法，此时编译器就会去找隐式类 `implicit class` 构造函数的参数类型是 `Statement` 的。找到之后，就把 `stmt` 作为 `val _stmt` 传参进隐式类里，再找名字叫 `map` 的方法来调用。 `addTwo` 方法作为 `map` 方法的参数 `f`。

`(implicit magnet: (T => T) => StmtMagnet)` 是隐式的参数，编译器会在调用 `def map` 时自动加上。例如这里的 `stmt map addTwo`，`T` 就会自动推导成 `Int` 类型，也就是转成

```scala 
def map[Int](addTwo: Int => Int)(implicit magnet: (Int => Int) => StmtMagnet): Statement = 
  magnet(addTwo).map(stmt)
```

`magnet(addTwo)` 是执行了参数 `implicit magnet: (T=>T) => StmtMagnet`，相当于调用了函数 `magnet`，参数是 `addTwo`，返回的结果类型是 `StmtMagnet`。编译器会去寻找能把 `addTwo`的 `Int => Int` 函数类型转成 `StmtMagnet` 的方法。

编译器在 `object StmtMagnet`里找到类型匹配的隐式方法 `implicit def forInt`，那么就会调用该方法生成一个新的 `StmtMagnet` 类型的对象，它重载了 `map` 方法，变成调用 `Statement` 类型的参数 `stmt` 的 `mapInt` 方法。

`magnet(addTwo)` 仅仅是生成了重载 `map` 方法后的 `Statement` 对象，还得调用该方法 `magnet(addTwo).map(stmt)`。相当于 `stmt mapInt addTwo`。

其实做了这么多事情，跟直接写 `stmt mapInt addTwo` 是一样的。

```scala
  val newStmt = stmt map addTwo map addStr
  // 等价于
  val newStmt = stmt mapInt addTwo mapStr addStr
```

好处是一旦定义好之后，每次写 `map` 就好了，不管是 `Int` 类型还是 `String` 类型。坏处就是读起来不直观，需要自己推导出，当 `map` 针对的 `Int` 类型时，实际上是调用 `mapInt` 方法，针对 `String` 类型时，实际上调用的是 `mapStr` 方法。

运行结果：

```shell
Before: 
Statement(5,Hello)
After: 
Statement(7,Hello world)
```

在对 **Firrtl** 的 **IR** 进行迭代处理过程中，大量地运用这种隐式转换的方法。

### Map 迭代

新建 `Mapper.scala` 文件。

```shell
$ touch firrtl/src/Mapper.scala
```

下面是 `Statement` **IR** 的 `def map` 方法实现。

#### Statement IR

```scala
// play-chisel/firrtl/src/Mapper.scala
package firrtl

import language.implicitConversions
import firrtl.ir._

object Mappers {
  // ********** Stmt Mappers **********
  private trait StmtMagnet {
    def map(stmt: Statement): Statement
  }
  private object StmtMagnet {
    implicit def forStmt(f: Statement => Statement): StmtMagnet = new StmtMagnet {
      override def map(stmt: Statement): Statement = stmt mapStmt f
    }
    implicit def forExp(f: Expression => Expression): StmtMagnet = new StmtMagnet {
      override def map(stmt: Statement): Statement = stmt mapExpr f
    }
    implicit def forType(f: Type => Type): StmtMagnet = new StmtMagnet {
      override def map(stmt: Statement) : Statement = stmt mapType f
    }
    implicit def forString(f: String => String): StmtMagnet = new StmtMagnet {
      override def map(stmt: Statement): Statement = stmt mapString f
    }
  }
  implicit class StmtMap(val _stmt: Statement) extends AnyVal {
    def map[T](f: T => T)(implicit magnet: (T => T) => StmtMagnet): Statement = magnet(f).map(_stmt)
  }
}
```

当对一个 `Statement` 类型的 **IR** 进行 `map` 的时： `stmt map f`，要明白一点，背后实际上根据不同的类型调用了不同的方法。

- `String` 类型对应 `stmt mapString f` ；
- `Type` 类型对应 `stmt mapType f` ；
- `Expression` 类型对应 `stmt mapExpr f` ；
- `Statement` 类型对应 `stmt mapStmt f`。

看下对应的这些方法，在 `Statement` **IR** 里是怎么定义的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class Statement extends FirrtlNode {
  def mapStmt   (f: Statement => Statement)   : Statement
  def mapExpr   (f: Expression => Expression) : Statement
  def mapType   (f: Type => Type)             : Statement
  def mapString (f: String => String)         : Statement
}
```

抽象类 `Statement` 规定了子类必须要有这四种方法，注意这里不管是什么方法，最后的返回值都是 `Statement` 类型。换句话说，这些方法只是更改了 `Statement` 里的某种属性，然后返回更改过的 `Statement` 对象。

目前继承 `Statement` 的子类有：`DefNode`、`Block` 和 `Connect` 三种，下面分别看下它们是如何实现这些方法的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class DefNode(name: String, value: Expression) extends Statement with IsDeclaration {
  def mapStmt(f: Statement => Statement): Statement = this
  def mapExpr(f: Expression => Expression): Statement = DefNode(name, f(value))
  def mapType(f: Type => Type): Statement = this
  def mapString(f: String => String): Statement = DefNode(f(name), value)
}
```

`DefNode` 只有 `String` 和 `Expression` 两种类型的属性 `name` 和 `value`。

因此 `mapStmt` 和 `mapType` 没有匹配类型的属性，什么也没改变就返回了自身。

`mapExpr` 和 `mapString` 方法是各自更改了对应类型的属性 `f(value)` 和 `f(name)`。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Block(stmts: Seq[Statement]) extends Statement {
  def mapStmt(f: Statement => Statement): Statement = Block(stmts map f)
  def mapExpr(f: Expression => Expression): Statement = this
  def mapType(f: Type => Type): Statement = this
  def mapString(f: String => String): Statement = this
}
```

`Block` 只有一个存放 `Statement` 序列的属性 `stmts`。

由于 `def mapStmt` 方法的参数 `f` 是针对一个 `Statement` 的函数，要对 `stmts` 进行 `map` 迭代，一一代入 `f` 方法，整体更改 `stmts` 属性。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Connect(loc: Expression, expr: Expression) extends Statement {
  def mapStmt(f: Statement => Statement): Statement = this
  def mapExpr(f: Expression => Expression): Statement = Connect(f(loc), f(expr))
  def mapType(f: Type => Type): Statement = this
  def mapString(f: String => String): Statement = this
}
```

`mapExpr` 是会调用函数 `f` 改变两个 `Expression` 属性：`loc: Expression` 和 `expr: Expression`，其他的方法什么也不做返回自身。

#### Expression IR

```scala
// play-chisel/firrtl/src/Mapper.scala
object Mappers {
  ...
  // ********** Expression Mappers **********
  private trait ExprMagnet {
    def map(expr: Expression): Expression
  }
  private object ExprMagnet {
    implicit def forExpr(f: Expression => Expression): ExprMagnet = new ExprMagnet {
      override def map(expr: Expression): Expression = expr mapExpr f
    }
    implicit def forType(f: Type => Type): ExprMagnet = new ExprMagnet {
      override def map(expr: Expression): Expression = expr mapType f
    }
    implicit def forWidth(f: Width => Width): ExprMagnet = new ExprMagnet {
      override def map(expr: Expression): Expression = expr mapWidth f
    }
  }
  implicit class ExprMap(val _expr: Expression) extends AnyVal {
    def map[T](f: T => T)(implicit magnet: (T => T) => ExprMagnet): Expression = magnet(f).map(_expr)
  }
}
```
当对一个 `Expression` 类型的 **IR** 进行 `map` 方法时（ `expr map f`），实际上是根据方法 `f` 的类型不同调用的方法：

- `Expression => Expression` 类型对应 `expr mapExpr f`；
- `Type => Type` 类型对应 `expr mapType f`；
- `Width => Width` 类型对应 `expr mapWidth f`。

看下对应的这些方法，在 `Expression` **IR** 里是怎么定义的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class Expression extends FirrtlNode {
  def mapExpr(f: Expression => Expression): Expression
  def mapType(f: Type => Type): Expression
  def mapWidth(f: Width => Width): Expression
}
```

抽象类 `Expression` 规定了子类必须要有这三种方法，注意这里不管是什么方法，最后的返回值都是 `Expression` 类型。换句话说，这些方法只是更改了 `Expression` 里的某种属性，然后返回更改过的 `Expression` 对象。

目前继承 `Expression` 的子类有：`Reference` 和 `DoPrim` 两种，下面分别看下它们是如何实现这些方法的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Reference(name: String, tpe: Type) extends Expression with HasName {
  def mapExpr(f: Expression => Expression): Expression = this
  def mapType(f: Type => Type): Expression = this.copy(tpe = f(tpe))
  def mapWidth(f: Width => Width): Expression = this
}
```

`Reference` 只有 `tpe: Type` 属性可以更改，因此 `mapExpr` 和 `mapWidth` 没有匹配类型的属性，什么也没改变就返回了自身。

`mapType` 方法调用 `f` 函数更改了 `tpe` 属性。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class DoPrim(op: PrimOp, args: Seq[Expression], consts: Seq[BigInt], tpe: Type) extends Expression {
  def mapExpr(f: Expression => Expression): Expression = this.copy(args = args map f)
  def mapType(f: Type => Type): Expression = this.copy(tpe = f(tpe))
  def mapWidth(f: Width => Width): Expression = this
}
```

`DoPrim` 可以更改的属性有 `args: Seq[Expression` 和 `tpe: Type`，因此 `mapWidth` 没有匹配类型的属性，什么也没改变就返回了自身。

`mapExpr` 和 `mapType` 方法则分别更改 `args` 和 `tpe` 两个属性。

#### Type IR

```scala
// play-chisel/firrtl/src/Mapper.scala
object Mappers {
  ...
  // ********** Type Mappers **********
  private trait TypeMagnet {
    def map(tpe: Type): Type
  }
  private object TypeMagnet {
    implicit def forType(f: Type => Type): TypeMagnet = new TypeMagnet {
      override def map(tpe: Type): Type = tpe mapType f
    }
    implicit def forWidth(f: Width => Width): TypeMagnet = new TypeMagnet {
      override def map(tpe: Type): Type = tpe mapWidth f
    }
  }
  implicit class TypeMap(val _tpe: Type) extends AnyVal {
    def map[T](f: T => T)(implicit magnet: (T => T) => TypeMagnet): Type = magnet(f).map(_tpe)
  }
}
```
当对一个 `Type` 类型的 **IR** 进行 `map` 方法时（ `tpe map f`），实际上是根据方法 `f` 的类型不同调用的方法：

- `Type => Type` 类型对应 `expr mapType f`；
- `Width => Width` 类型对应 `expr mapWidth f`。

看下对应的这些方法，在 `Type` **IR** 里是怎么定义的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class Type extends FirrtlNode {
  def mapType(f: Type => Type): Type
  def mapWidth(f: Width => Width): Type
}
```

抽象类 `Type` 规定了子类必须要有这两种方法，注意这里不管是什么方法，最后的返回值都是 `Type` 类型。换句话说，这些方法只是更改了 `Type` 里的某种属性，然后返回更改过的 `Type` 对象。

目前继承 `Type` 的子类有 **GroundType** 和 **UnknownType** ：

- **GroundType**
  - **UIntType**
- **UnknownType**

下面分别看下它们是如何实现这些方法的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class GroundType extends Type {
  def mapType(f: Type => Type): Type = this
}
```

**GroundType** 子类是没有类型属性的，因此 `mapType` 方法什么也没修改，就返回了自身。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class UIntType(width: Width) extends GroundType {
  def mapWidth(f: Width => Width): Type = UIntType(f(width))
}
```

`UIntType` 的 `mapWidth` 会调用函数 `f` 修改宽度属性。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case object UnknownType extends Type {
  def mapType(f: Type => Type): Type = this
  def mapWidth(f: Width => Width): Type = this
}
```

**UnknownType** 没有任何属性，所以方法都是返回自身。

#### Width IR

```scala
// play-chisel/firrtl/src/Mapper.scala
object Mappers {
  ...
  // ********** Width Mappers **********
  private trait WidthMagnet {
    def map(width: Width): Width
  }
  private object WidthMagnet {
    implicit def forWidth(f: Width => Width): WidthMagnet = new WidthMagnet {
      override def map(width: Width): Width = width match {
        case mapable: HasMapWidth => mapable mapWidth f // WIR
        case other => other // Standard IR nodes
      }
    }
  }
  implicit class WidthMap(val _width: Width) extends AnyVal {
    def map[T](f: T => T)(implicit magnet: (T => T) => WidthMagnet): Width = magnet(f).map(_width)
  }
}
```

当对一个 `Width` 类型的 **IR** 进行 `map` 方法时（ `tpe map f`），实际上是根据方法 `f` 的类型不同调用的方法：

- `Width => Width` 类型对应 `width mapWidth f`。

看下对应的这些方法，在 `Width` **IR** 里是怎么定义的。

不是每种 `Width` 类型都有 `mapWidth` 方法。只有继承了 `HasMapWidth` 的才有。

```shell
$ touch firrtl/src/WIR.scala
```

```scala
// play-chisel/firrtl/src/WIR.scala
package firrtl

import Utils._
import firrtl.ir._

private[firrtl] sealed trait HasMapWidth {
  def mapWidth(f: Width => Width): Width
}

case class MaxWidth(args: Seq[Width]) extends Width with HasMapWidth {
  def mapWidth(f: Width => Width): Width = MaxWidth(args map f)
}
```

`MaxWidth` 是有 `mapWidth` 方法的，就是调用函数 `f` 改变它 `args: Seq[Width]` 属性里的每一个 `Width` 类型的元素。

#### Module IR

```scala
// play-chisel/firrtl/src/Mapper.scala
object Mappers {
  ...
  // ********** Module Mappers **********
  private trait ModuleMagnet {
    def map(module: DefModule): DefModule
  }
  private object ModuleMagnet {
    implicit def forStmt(f: Statement => Statement): ModuleMagnet = new ModuleMagnet {
      override def map(module: DefModule): DefModule = module mapStmt f
    }
    implicit def forPorts(f: Port => Port): ModuleMagnet = new ModuleMagnet {
      override def map(module: DefModule): DefModule = module mapPort f
    }
    implicit def forString(f: String => String): ModuleMagnet = new ModuleMagnet {
      override def map(module: DefModule): DefModule = module mapString f
    }
  }
  implicit class ModuleMap(val _module: DefModule) extends AnyVal {
    def map[T](f: T => T)(implicit magnet: (T => T) => ModuleMagnet): DefModule = magnet(f).map(_module)
  }
}
```

当对一个 `Module` 类型的 **IR** 进行 `map` 方法时（ `module map f`），实际上是根据方法 `f` 的类型不同调用的方法：

- `Statement => Statement` 类型对应 `module mapStmt f`。
- `Port => Port` 类型对应 `module mapPort f`。
- `String => String` 类型对应 `module mapString f`。

看下对应的这些方法，在 `Module` **IR** 里是怎么定义的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class DefModule extends FirrtlNode with IsDeclaration {
  ...
  def mapStmt(f: Statement => Statement): DefModule
  def mapPort(f: Port => Port): DefModule
  def mapString(f: String => String): DefModule
}
```

抽象类 `DefModule` 规定了子类必须要有这三种方法，注意这里不管是什么方法，最后的返回值都是 `DefModule` 类型。换句话说，这些方法只是更改了 `DefModule` 里的某种属性，然后返回更改过的 `DefModule` 对象。

目前继承 `DefModule` 类型的只有 `Module`。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Module(name: String, ports: Seq[Port], body: Statement) extends DefModule {
  def mapStmt(f: Statement => Statement): DefModule = this.copy(body = f(body))
  def mapPort(f: Port => Port): DefModule = this.copy(ports = ports map f)
  def mapString(f: String => String): DefModule = this.copy(name = f(name))
}
```

`mapStmt` 调用函数 `f` 改变的是 `body: Statement` 属性，`mapPort` 改变的是 `ports: Seq[Port]` 属性， `mapString` 改变的是 `name: String` 属性。

#### Circuit IR

```scala
// play-chisel/firrtl/src/Mapper.scala
object Mappers {
  ...
  // ********** Circuit Mappers **********
  private trait CircuitMagnet {
    def map(module: Circuit): Circuit
  }
  private object CircuitMagnet {
    implicit def forModules(f: DefModule => DefModule): CircuitMagnet = new CircuitMagnet {
      override def map(circuit: Circuit): Circuit = circuit mapModule f
    }
    implicit def forString(f: String => String): CircuitMagnet = new CircuitMagnet {
      override def map(circuit: Circuit): Circuit = circuit mapString f
    }
  }
  implicit class CircuitMap(val _circuit: Circuit) extends AnyVal {
    def map[T](f: T => T)(implicit magnet: (T => T) => CircuitMagnet): Circuit = magnet(f).map(_circuit)
  }
}
```

当对一个 `Circuit` 类型的 **IR** 进行 `map` 方法时（ `circuit map f`），实际上是根据方法 `f` 的类型不同调用的方法：

- `DefModule => DefModule` 类型对应 `module mapModule f`。
- `String => String` 类型对应 `module mapString f`。

看下对应的这些方法，在 `Circuit` **IR** 里是怎么定义的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Circuit(modules: Seq[DefModule], main: String) extends FirrtlNode {
  def mapModule(f: DefModule => DefModule): Circuit = this.copy(modules = modules map f)
  def mapString(f: String => String): Circuit = this.copy(main = f(main))
}
```

`Circuit` 的 `mapModule` 是调用函数 `f` 修改 `modules: Seq[DefModule]` 属性里的每一个 `DefModule` 元素，`mapString` 是修改 `main: String` 属性。

### for 迭代

`foreach` 的迭代方式跟 `map` 的类似，都是会根据 `f` 方法的参数类型隐式转成对应的方法。

目前只有 `Statement` **IR** 需要到 `foreach`。

#### Statement IR

```shell
$ mkdir -p firrtl/src/traversals
$ touch firrtl/src/traversals/Foreachers.scala
```

```scala
// play-chisel/firrtl/src/traversals/Foreachers.scala
package firrtl.traversals

import firrtl.ir._
import language.implicitConversions

object Foreachers {

  /** Statement Foreachers */
  private trait StmtForMagnet {
    def foreach(stmt: Statement): Unit
  }
  private object StmtForMagnet {
    implicit def forStmt(f: Statement => Unit): StmtForMagnet = new StmtForMagnet {
      def foreach(stmt: Statement): Unit = stmt foreachStmt f
    }
    implicit def forExp(f: Expression => Unit): StmtForMagnet = new StmtForMagnet {
      def foreach(stmt: Statement): Unit = stmt foreachExpr f
    }
    implicit def forType(f: Type => Unit): StmtForMagnet = new StmtForMagnet {
      def foreach(stmt: Statement) : Unit = stmt foreachType f
    }
    implicit def forString(f: String => Unit): StmtForMagnet = new StmtForMagnet {
      def foreach(stmt: Statement): Unit = stmt foreachString f
    }
  }
  implicit class StmtForeach(val _stmt: Statement) extends AnyVal {
    // Using implicit types to allow overloading of function type to foreach, see StmtForMagnet above
    def foreach[T](f: T => Unit)(implicit magnet: (T => Unit) => StmtForMagnet): Unit = magnet(f).foreach(_stmt)
  }
}
```
当对一个 `Statement` 类型的 **IR** 进行 `foreach` 方法时（ `stmt foreach f`），实际上是根据方法 `f` 的类型不同调用的方法：

- `Statement => Unit` 类型对应 `stmt foreachStmt f`。
- `Expression => Unit` 类型对应 `stmt foreachExpr f`。
- `Type => Unit` 类型对应 `stmt foreachType f`。
- `String => Unit` 类型对应 `stmt foreachStr f`。

看下对应的这些方法，在 `Statement` **IR** 里是怎么定义的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class Statement extends FirrtlNode {
  ...
  def foreachStmt(f: Statement => Unit)  : Unit
  def foreachExpr(f: Expression => Unit) : Unit
  def foreachType(f: Type => Unit)       : Unit
  def foreachString(f: String => Unit)   : Unit
}
```

抽象类 `Statement` 规定了子类必须要有这四种方法，注意这里不管是什么方法，最后的返回值都是 `Unit` 类型。换句话说，这些方法只会以这些属性作为参数，触发相应的动作，但不会影响这些属性。

目前继承 `Statement` 的子类有：`DefNode`、`Block` 和 `Connect` 三种，下面分别看下它们是如何实现这些方法的。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class DefNode(name: String, value: Expression) extends Statement with IsDeclaration {
  ...
  def foreachStmt(f: Statement => Unit): Unit = Unit
  def foreachExpr(f: Expression => Unit): Unit = f(value)
  def foreachType(f: Type => Unit): Unit = Unit
  def foreachString(f: String => Unit): Unit = f(name)
}
```

`DefNode` 只有 `String` 和 `Expression` 两种类型的属性 `name` 和 `value`。

因此 `foreachStmt` 和 `foreachType` 没有匹配类型的属性，什么也不做返回空 `Unit`。

`foreachExpr` 和 `foreachString` 方法是各自调用了函数 `f` 传入对应类型的属性 `f(value)` 和 `f(name)`。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Block(stmts: Seq[Statement]) extends Statement {
  ...
  def foreachStmt(f: Statement => Unit): Unit = stmts.foreach(f)
  def foreachExpr(f: Expression => Unit): Unit = Unit
  def foreachType(f: Type => Unit): Unit = Unit
  def foreachString(f: String => Unit): Unit = Unit
}
```

`Block` 只有一个存放 `Statement` 序列的属性 `stmts`。

由于 `def foreachStmt` 方法的参数 `f` 是针对一个 `Statement` 的函数，要对 `stmts` 进行 `map` 迭代，一一代入每个元素作为参数执行 `f` 方法。

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class Connect(loc: Expression, expr: Expression) extends Statement {
  ...
  def foreachStmt(f: Statement => Unit): Unit = Unit
  def foreachExpr(f: Expression => Unit): Unit = { f(loc); f(expr) }
  def foreachType(f: Type => Unit): Unit = Unit
  def foreachString(f: String => Unit): Unit = Unit
}
```

`foreachExpr` 是会分别传入两个 `Expression` 属性：`loc: Expression` 和 `expr: Expression` 先后执行函数 `f`。


## 推导类型

```shell
$ touch firrtl/src/passes/InferTypes.scala
```

```scala
// play-chisel/firrtl/src/passes/InferTypes.scala
package firrtl.passes

import firrtl._
import firrtl.ir._
import firrtl.Utils._
import firrtl.Mappers._

object CInferTypes extends Pass {
  type TypeMap = collection.mutable.LinkedHashMap[String, Type]

  def run(c: Circuit): Circuit = {
    def infer_types(m: DefModule): DefModule = {
      val types = new TypeMap
      m map infer_types_p(types) map infer_types_s(types)
    }

    c copy (modules = c.modules map infer_types)
  }
}
```

`c copy (modules = ...)` 是改变了`run` 方法的参数 `c: Circuit` 里 `modules` 属性，其他属性不变，然后作为方法的结果返回。

新的 `modules` 值是 `c.modules map infer_types`，这段代码把模块列表里的每个模块都调用方法 `infer_types` 来推导类型。

`def infer_types` 方法的类型是 `DefModule => DefModule`，改变的是 `c: Circuit` 电路里的模块属性 `c.modules`。

`def infer_types` 方法里声明了一个哈希表 `types: TypeMap`，存放名字及其对应的类型。

`m map infer_types_p(types)` 调用 `infer_types_p` 方法改变模块 `port` 属性的类型，然后返回修改过的模块，再调用 `map infer_type_s(types)` 方法修改模块的语句列表 `body` 属性。


```scala
object CInferTypes extends Pass {
  def run(c: Circuit): Circuit = {
    def infer_types_p(types: TypeMap)(p: Port): Port = {
      types(p.name) = p.tpe
      p
    }
  }
}
```

`infer_types_p` 是收集已知的端口类型，按名字放入 `types` 哈希表里，并返回没有修改过的 `port` 自身。

```scala
object CInferTypes extends Pass {
  def run(c: Circuit): Circuit = {
    def infer_types_s(types: TypeMap)(s: Statement): Statement = s match {
      case sx: DefNode =>
        val sxx = (sx map infer_types_e(types)).asInstanceOf[DefNode]
        types(sxx.name) = sxx.value.tpe
        sxx
      case sx => sx map infer_types_s(types) map infer_types_e(types)
    }
  }
}
```

`def infer_types_s` 方法是接受一个语句 **IR** 参数，然后返回修改过的语句 **IR**。一开始的时候，传入的参数是 `body: Statement`，是一个 `Block` **IR**，它的属性 `stmts` 是一个语句列表。所以它满足的 `case sx => sx ...` 这个条件。

`sx map infer_types_s(types)` 会遍历 `body` 的每个语句（`Block(stmts map f)`），作为参数传入 `infer_types_s(types)` 方法里。 `map infer_types_e(types)` 对 `body` 这种 `Block` 类型的 **IR** 是没起什么作用的，因为它的 `mapExpr` 方法是什么也不做返回自身。

`Block` 语句列表里还有可能是 `DefNode` 和 `Connect` 类型的**IR**。

如是 `DefNode`，符合的条件是 `case sx: DefNode =>`， `(sx map infer_types_e(types))` 返回的是修改了 `sx` 之后新的值放到 `sxx`，（`sx` 本身并没有改变）。由于 `sx` 的 `map` 实际上调用的 `mapExpr` 方法返回值类型是 `Statement`，而后面的 `sxx.value.tpe` 需要的是 `DefNode` 类型（`DefNode` 才有 `value` 属性），所以需要用 `asInstanceOf[DefNode]` 转换类型。

`types(sxx.name) = sxx.value.tpe` 是收集该 `DefNode` 的类型信息，然后返回修改过的新语句 `sxx`。

如果是 `Connect` **IR**，符合的条件是 `case sx => ...`。 `sx map infer_types_s(types)` 调用的是它的 `mapStmt` 方法，什么也不做返回自身，然后再 `map infer_types_e(types)`，调用它的 `mapExpr` 方法修改 `loc` 和 `expr` 属性。

接下来看下 `infer_types_e` 如何修改 `Expression` **IR**。

```scala
object CInferTypes extends Pass {
  type TypeMap = collection.mutable.LinkedHashMap[String, Type]

  def run(c: Circuit): Circuit = {
    def infer_types_e(types: TypeMap)(e: Expression) : Expression =
      e map infer_types_e(types) match {
         case (e: Reference) => e copy (tpe = types.getOrElse(e.name, UnknownType))
         case (e: DoPrim) => PrimOps.set_primop_type(e)
      }
  }
}
```

`infer_types_e` 方法先对参数递归调用 `e map infer_types_e(types)` 是为了保证像 `DoPrim` **IR** 包含了 `Expression` 的属性 `args: Seq[Expression]` 也能被修改。

如果是引用类型 `Reference` 的 **IR**，说明之前可能已经声明过了，就去收集好的类型哈希表 `types` 里面找，找不到就是未知类型了 `UnknownType`。

如果是运算类型 `DoPrim` 的 **IR**，则根据运算操作的类型和运算操作数的类型共同决定。

```scala
// play-chisel/firrtl/src/PrimOps.scala
import firrtl.Utils.max

object PrimOps {
  def MAX (w1:Width, w2:Width) : Width = (w1, w2) match {
    case (IntWidth(i), IntWidth(j)) => IntWidth(max(i,j))
    case _ => MaxWidth(Seq(w1, w2))
  }

  def set_primop_type (e:DoPrim) : DoPrim = {
    def t1 = e.args.head.tpe
    def t2 = e.args(1).tpe
    def w1 = getWidth(e.args.head.tpe)
    def w2 = getWidth(e.args(1).tpe)
    e copy (tpe = e.op match {
      case Not => t1 match {
        case (_: UIntType) => UIntType(w1)
        case _ => UnknownType
      }
      case And | Or => (t1, t2) match {
        case (_: UIntType, _: UIntType) => UIntType(MAX(w1, w2))
        case _ => UnknownType
      }
    })
  }
}
```

这个例子里只有 `Not`、`And`和`Or`三种运算类型。

如果是 `Not` 一元运算，表达式类型是操作数（只有一个）的类型。

如果是 `And` 和 `Or` 二元运算，表达式类型是两个操作数的类型，宽度是两者之间最大值。

```scala
// play-chisel/firrtl/src/Utils.scala

import firrtl.ir._

object Utils {
  def max(a: BigInt, b: BigInt): BigInt = if (a >= b) a else b
}

object getWidth {
  def apply(t: Type): Width = t match {
    case t: GroundType => t.width
    case _ => Utils.error(s"No width: $t")
  }
  def apply(e: Expression): Width = apply(e.tpe)
}
```

```scala
// play-chisel/firrtl/src/ir/IR.scala

abstract class GroundType extends Type {
  val width: Width
}
```

```scala
// play-chisel/firrtl/src/LoweringCompiler.scala

class ChirrtlToHighFirrtl extends CoreTransform {
  def transforms = Seq(
    passes.CInferTypes,
  )
}
```

```scala
// play-chisel/src/Main.scala

object Main extends App {
  ...
  val firrtl = Converter.convert(circuit)

  println("======FIRRTL======")
  print_fir(firrtl)

  val state = CircuitState(firrtl, ChirrtlForm)
  val compiler = new VerilogCompiler
  val res = compiler.compile(state)
  println("======Compiling...======")
  println("======After Infer Types======")
  print_fir(res.circuit)
}
```

```shell
$ mill chisel3.run
```
{{% admonition tip 源码03 %}}
{{% /admonition %}}

[play-chisel/tree/chap02-03](https://github.com/colin4124/play-chisel/tree/chap02-03)



## 转成工作 IR

工作 **IR** 比 **IR** 携带更多的信息。

除了上一节介绍了一些 **WIR** 相关的代码，还有如下的定义：

```scala
// play-chisel/firrtl/src/WIR.scala
trait Kind
case object PortKind extends Kind
case object NodeKind extends Kind
case object UnknownKind extends Kind

trait Flow
case object SourceFlow extends Flow
case object SinkFlow extends Flow
case object UnknownFlow extends Flow

case class WRef(name: String, tpe: Type, kind: Kind, flow: Flow) extends Expression {
  def serialize: String = name
  def mapExpr(f: Expression => Expression): Expression = this
  def mapType(f: Type => Type): Expression = this.copy(tpe = f(tpe))
  def mapWidth(f: Width => Width): Expression = this
}
```

**WIR** 多两种新的属性：类别 **Kind** 和 方向 **Flow**。

**WRef** 还扩展了 **Expression** **IR**，新增了 `kind: Kind` 和 `flow: Flow` 两种新类型的属性。

**WrappedExpression** 携带一个 `Expression` **IR** 的属性 `val e1`。

```scala
// play-chisel/firrtl/src/passes/Passes.scala
object ToWorkingIR extends Pass {
  def toExp(e: Expression): Expression = e map toExp match {
    case ex: Reference => WRef(ex.name, ex.tpe, UnknownKind, UnknownFlow)
    case ex => ex
  }

  def toStmt(s: Statement): Statement = s map toExp match {
    case sx => sx map toStmt
  }

  def run (c:Circuit): Circuit =
    c copy (modules = c.modules map (_ map toStmt))
}
```

这里只是简单地把 `Reference` 变成 `WRef` 类型，携带了额外的 **Kind**，和 **Flow** 两个属性。由于信息不够多，所以先放未知类型，留个后面的流程去解析具体的种类 **Kind** 和方向 **Flow**。

```scala
// play-chisel/firrtl/src/LoweringCompiler.scala

class IRToWorkingIR extends CoreTransform {
  ...
  def transforms = Seq(passes.ToWorkingIR)
}
}
```

```scala
// play-chisel/src/PrintIR.scala
import firrtl._

object PrintIR {
  def e_str(e: fir.Expression, l: Int): String = {
    e match {
      ...
      case WRef(n, t, k, f) => s"WRef(${n}: ${type_str(t)} ${k_str(k)} ${f_str(f)})"
    }
  }
  def k_str(k: Kind): String = k match {
    case PortKind    => "PortKind"
    case NodeKind    => "NodeKind"
    case UnknownKind => "UnknownKind"
  }
  def f_str(f: Flow): String = f match {
    case SourceFlow  => "SourceFlow"
    case SinkFlow    => "SinkFlow"
    case UnknownFlow => "UnknownFlow"
  }
}
```

```shell
$ mill chisel3.run
```
{{% admonition tip 源码04 %}}
{{% /admonition %}}

[play-chisel/tree/chap02-04](https://github.com/colin4124/play-chisel/tree/chap02-04)

## 解析种类

```shell
$ touch firrtl/src/passes/Resolves.scala
```

```scala
// play-chisel/firrtl/src/passes/Resolves.scala
package firrtl.passes

import firrtl._
import firrtl.ir._
import firrtl.Mappers._

object ResolveKinds extends Pass {
  type KindMap = collection.mutable.LinkedHashMap[String, Kind]
  ...
  def resolve_kinds(m: DefModule): DefModule = {
    val kinds = new KindMap
    (m map find_port(kinds)
       map find_stmt(kinds)
       map resolve_stmt(kinds))
  }

  def run(c: Circuit): Circuit =
    c copy (modules = c.modules map resolve_kinds)
}
```

这里先去模块的端口和语句中寻找已知种类的信息，存到种类哈希表 `val kinds = new KindMap` 里面。最后根据这个种类哈希表来解析未知种类的语句。

```scala
// play-chisel/firrtl/src/passes/Resolves.scala
package firrtl.passes

object ResolveKinds extends Pass {
  def find_port(kinds: KindMap)(p: Port): Port = {
    kinds(p.name) = PortKind ; p
  }

  def find_stmt(kinds: KindMap)(s: Statement):Statement = {
    s match {
      case sx: DefNode => kinds(sx.name) = NodeKind
      case _ =>
    }
    s map find_stmt(kinds)
  }
  ...
}
```

`find_port` 方法把端口名添加到种类哈希表里作为端口种类。

`find_stmt` 遍历一遍语句属性，发现 `DefNode` 语句就把它添加到种类哈希表，作为 `NodeKind` 类型。


```scala
// play-chisel/firrtl/src/passes/Resolves.scala

object ResolveKinds extends Pass {
  def resolve_expr(kinds: KindMap)(e: Expression): Expression = e match {
    case ex: WRef => ex copy (kind = kinds(ex.name))
    case _ => e map resolve_expr(kinds)
  }
  def resolve_stmt(kinds: KindMap)(s: Statement): Statement =
    s map resolve_stmt(kinds) map resolve_expr(kinds)
  ...
}
```

`resolve_expr` 方法找到上一个步骤留下未知种类的 `WRef` **IR**，然后去查找种类哈希表，获得对应的种类信息。

```scala
// play-chisel/firrtl/src/LoweringCompiler.scala

class ResolveAndCheck extends CoreTransform {
  ...
  def transforms = Seq(
    passes.ResolveKinds,
  )
}
```

```shell
$ mill chisel3.run
```
{{% admonition tip 源码05 %}}
{{% /admonition %}}

[play-chisel/tree/chap02-05](https://github.com/colin4124/play-chisel/tree/chap02-05)

## 解析方向

```scala
// play-chisel/firrtl/src/passes/Resolves.scala
object ResolveFlows extends Pass {
  ...
  def resolve_flow(m: DefModule): DefModule = m map resolve_s

  def run(c: Circuit): Circuit =
    c copy (modules = c.modules map resolve_flow)
}
```

这里解析方向只针对模块的语句，不需遍历端口了。

```scala
// play-chisel/firrtl/src/passes/Resolves.scala
object ResolveFlows extends Pass {
  def resolve_s(s: Statement): Statement = s match {
    case Connect(loc, expr) =>
      Connect(resolve_e(SinkFlow)(loc), resolve_e(SourceFlow)(expr))
    case sx => sx map resolve_e(SourceFlow) map resolve_s
  }
  ...
}
```

如果是 `Connect` 语句，左值 `loc` 的方向是 `SinkFlow` ，右值 `expr` 的方向是 `SourceFlow`。

其他类型的语句是 `SourceFlow`。

```scala
// play-chisel/firrtl/src/passes/Resolves.scala
object ResolveFlows extends Pass {
  def resolve_e(g: Flow)(e: Expression): Expression = e match {
    case ex: WRef => ex copy (flow = g)
    case _ => e map resolve_e(g)
  }
  ...
}
```

`resolve_e` 方法是设置表达式的方向为它的 `g: Flow` 参数。

```scala
// play-chisel/firrtl/src/LoweringCompiler.scala

class ResolveAndCheck extends CoreTransform {
  ...
  def transforms = Seq(
    passes.ResolveKinds,
    passes.ResolveFlows,
  )
}
```

```shell
$ mill chisel3.run
```
{{% admonition tip 源码06 %}}
{{% /admonition %}}

[play-chisel/tree/chap02-06](https://github.com/colin4124/play-chisel/tree/chap02-06)

## 生成 Verilog

`VerilogEmitter` 负责生成 **Verilog**。

```scala
// play-chisel/firrtl/src/Emitter.scala
package firrtl

import java.io.Writer
import scala.collection.mutable

import firrtl.ir._
import firrtl.passes._
import firrtl.traversals.Foreachers._
import firrtl.PrimOps._
import Utils._

case class EmitterException(message: String) extends PassException(message)

class VerilogEmitter extends SeqTransform with Emitter {
    ...
    def emit(state: CircuitState, writer: Writer): Unit = {
      val circuit = state.circuit
      circuit.modules.foreach {
        case m: Module =>
          val renderer = new VerilogRender(m)(writer)
          renderer.emit_verilog()
          println(writer.toString)
        case _ => // do nothing
      }
    }

    override def execute(state: CircuitState): CircuitState = {
      val writer = new java.io.StringWriter
      emit(state, writer)
      state
    }
}
```

`execute` 方法调用 `emit` 方法，传递电路状态参数和 `writer` 参数。

`def emit(state: CircuitState, write: Writer)` 方法根据参数电路状态 `state`，把生成的 **Verilog** 放到参数 `writer` 里。

然后循环调用 `VerilogRender` 的 `emit_verilog` 方法生成电路里每个模块对应的 **Verilog**。

```scala
// play-chisel/firrtl/src/Emitter.scala
class VerilogEmitter extends SeqTransform with Emitter {
  ...
  class VerilogRender(m: Module)(implicit writer: Writer) {
    ...
    def emit_verilog(): DefModule = {
      build_netlist(m.body)
      build_ports()
      build_streams(m.body)
      emit_streams()
      m
    }
  }
}
```

`VerilogRender` 类的构造器有两个参数 `(m: Module)` 和 `(implicit writer: Writer)`，`implicit` 的好处是，当类里面调用的方法需要到 `writer: Writer` 这个参数时，可以省略。

### 构建网表

```scala
// play-chisel/firrtl/src/Emitter.scala
class VerilogEmitter extends SeqTransform with Emitter {
  ...
  class VerilogRender(m: Module)(implicit writer: Writer) {
    val netlist = mutable.LinkedHashMap[WrappedExpression, Expression]()
    def build_netlist(s: Statement): Unit = {
      s.foreach(build_netlist)
      s match {
        case sx: Connect => netlist(sx.loc) = sx.expr
        case sx: DefNode =>
          val e = WRef(sx.name, sx.value.tpe, NodeKind, SourceFlow)
          netlist(e) = sx.value
        case _ =>
      }
    }
    ...
  }
}
```

`netlist` 存的是 `WrappedExpression` 为键， `Expression` 为值的哈希表。

如果是 `Connect` 语句，则赋值语句的左值 `sx.loc` 为键，右值 `sx.expr` 为值。

如果是 `DefNode` 语句，键是 `WRef`，值是 `DefNode` 的值 `sx.value`。

```scala
// play-chisel/firrtl/src/Emitter.scala
class VerilogEmitter extends SeqTransform with Emitter {
  ...
  class VerilogRender(m: Module)(implicit writer: Writer) {
    ...
    val portdefs = mutable.ArrayBuffer[Seq[Any]]()
    val declares = mutable.ArrayBuffer[Seq[Any]]()
    val assigns = mutable.ArrayBuffer[Seq[Any]]()
    def declare(b: String, n: String, t: Type): Unit = t match {
      case tx =>
        declares += Seq(b, " ", tx, " ", n,";")
    }

    def assign(e: Expression, value: Expression): Unit = {
      assigns += Seq("assign ", e, " = ", value, ";")
    }
    ...
  }
}
```

`portdefs`、`declares` 和 `assigns` 分别用于存放端口定义，声明和赋值语句。

`def declare` 和 `def assign` 方法分别往各自的数组里添加信息。

### 构建端口

```scala
// play-chisel/firrtl/src/Emitter.scala
class VerilogEmitter extends SeqTransform with Emitter {
  ...
  class VerilogRender(m: Module)(implicit writer: Writer) {
    ...
    // Turn ports into Seq[String] and add to portdefs
    def build_ports(): Unit = {
      def padToMax(strs: Seq[String]): Seq[String] = {
        val len = if (strs.nonEmpty) strs.map(_.length).max else 0
        strs map (_.padTo(len, ' '))
      }

      // Turn directions into strings (and AnalogType into inout)
      val dirs = m.ports map { case Port(name, dir, tpe) =>
        (dir, tpe) match {
          case (Input, _) => "input "
          case (Output, _) => "output"
        }
      }
      // Turn types into strings, all ports must be GroundTypes
      val tpes = m.ports map {
        case Port(_, _, tpe: GroundType) => stringify(tpe)
        case port: Port => error(s"Trying to emit non-GroundType Port $port")
      }

      // dirs are already padded
      (dirs, padToMax(tpes), m.ports).zipped.toSeq.zipWithIndex.foreach {
        case ((dir, tpe, Port(name, _, _)), i) =>
          if (i != m.ports.size - 1) {
            portdefs += Seq(dir, " ", tpe, " ", name, ",")
          } else {
            portdefs += Seq(dir, " ", tpe, " ", name)
          }
      }
    }
    ...
  }
}
```

```scala
def padToMax(strs: Seq[String]): Seq[String] = {
  val len = if (strs.nonEmpty) strs.map(_.length).max else 0
  strs map (_.padTo(len, ' '))
}
```

先获取整个数组字符串中最大的长度，然后为每个字符串拼接空格达到最大的长度，用作对齐。

```scala
// Turn directions into strings (and AnalogType into inout)
val dirs = m.ports map { case Port(name, dir, tpe) =>
  (dir, tpe) match {
    case (Input, _) => "input "
    case (Output, _) => "output"
  }
}
```

获取每个端口的方向，输入为 `"input"`，输出为 `"output"`。

```scala
// Turn types into strings, all ports must be GroundTypes
val tpes = m.ports map {
  case Port(_, _, tpe: GroundType) => stringify(tpe)
  case port: Port => error(s"Trying to emit non-GroundType Port $port")
}
```

获取每个端口类型对应的字符串。

```scala
// dirs are already padded
(dirs, padToMax(tpes), m.ports).zipped.toSeq.zipWithIndex.foreach {
  case ((dir, tpe, Port(name, _, _)), i) =>
    if (i != m.ports.size - 1) {
      portdefs += Seq(dir, " ", tpe, " ", name, ",")
    } else {
      portdefs += Seq(dir, " ", tpe, " ", name)
    }
}
```

开始生成端口的 **Verilog** 定义，最后一个端口定义的结尾没有逗号 `,`。

### 构建语句

```scala
def build_streams(s: Statement): Unit = {
  s.foreach(build_streams)
  s match {
    case sx@Connect(loc@WRef(_, _, PortKind, _), expr) =>
      assign(loc, expr)
    case sx: DefNode =>
      declare("wire", sx.name, sx.value.tpe)
      assign(WRef(sx.name, sx.value.tpe, NodeKind, SourceFlow), sx.value)
    case _ =>
  }
}
```

`s.foreach(build_streams)` 先确保内嵌语句列表的 **Statement**（比如 `Block`）也能被方法 `build_streams` 遍历到。

如果是 `Connect` 语句，直接把左值和右值添加到赋值语句列表。

如果是 `DefNode` 语句，先添加到声明语句列表，然后再添加到赋值语句列表。

### 生成语句

```scala
def emit_streams(): Unit = {
  emit(Seq("module ", m.name, "("))
  for (x <- portdefs) emit(Seq(tab, x))
  emit(Seq(");"))

  if (declares.isEmpty && assigns.isEmpty) emit(Seq(tab, "initial begin end"))
  for (x <- declares) emit(Seq(tab, x))
  for (x <- assigns) emit(Seq(tab, x))
  emit(Seq("endmodule"))
}
```

这里是真正生成 **Verilog** 语句的方法，先声明模块名字，然后是端口列表，声明语句和赋值语句，最后是模块的结束。

### 辅助方法

```scala
val tab = "  "
def stringify(tpe: GroundType): String = tpe match {
  case (_: UIntType) =>
    val wx = bitWidth(tpe) - 1
    if (wx > 0) s"[$wx:0]" else ""
  case _ => throwInternalError(s"trying to write unsupported type in the Verilog Emitter: $tpe")
}
def emit(x: Any)(implicit w: Writer): Unit = { emit(x, 0) }
def emit(x: Any, top: Int)(implicit w: Writer): Unit = {
  def cast(e: Expression): Any = e.tpe match {
    case (t: UIntType) => e
    case _ => throwInternalError(s"unrecognized cast: $e")
  }
  x match {
    case (e: DoPrim) => emit(op_stream(e), top + 1)
    case (e: WRef) => w write e.serialize
    case (t: GroundType) => w write stringify(t)
    case (s: String) => w write s
    case (i: Int) => w write i.toString
    case (i: Long) => w write i.toString
    case (i: BigInt) => w write i.toString
    case (s: Seq[Any]) =>
      s foreach (emit(_, top + 1))
      if (top == 0) w write "\n"
    case x => throwInternalError(s"trying to emit unsupported operator: $x")
  }
}

def op_stream(doprim: DoPrim): Seq[Any] = {
  def cast_as(e: Expression): Any = e.tpe match {
    case (t: UIntType) => e
    case _ => throwInternalError(s"cast_as - unrecognized type: $e")
  }
  def a0: Expression = doprim.args.head
  def a1: Expression = doprim.args(1)

  def checkArgumentLegality(e: Expression) = e match {
    case _: WRef =>
    case _ => throw EmitterException(s"Can't emit ${e.getClass.getName} as PrimOp argument")
  }

  doprim.args foreach checkArgumentLegality
  doprim.op match {
    case Not => Seq("~ ", a0)
    case And => Seq(cast_as(a0), " & ", cast_as(a1))
    case Or => Seq(cast_as(a0), " | ", cast_as(a1))
  }
}
```

`stringify` 方法是把类型转成对应的字符串。

`emit` 生成最终的 **Verilog** 字符串，写入 `Writer`。

`op_stream` 根据运算语句生成对应的 **Verilog** 语句。

```scala
// play-chisel/firrtl/src/WIR.scala
case class WRef(name: String, tpe: Type, kind: Kind, flow: Flow) extends Expression {
  def serialize: String = name
  ...
}

class WrappedExpression(val e1: Expression)
```

```scala
// play-chisel/firrtl/src/Utils.scala
import language.implicitConversions

object Utils {
  implicit def toWrappedExpression (x:Expression): WrappedExpression = new WrappedExpression(x)
}

object bitWidth {
  def apply(dt: Type): BigInt = widthOf(dt)
  private def widthOf(dt: Type): BigInt = dt match {
    case GroundType(IntWidth(width)) => width
    case t => Utils.error(s"Unknown type encountered in bitWidth: $dt")
  }
}
```

```scala
// play-chisel/firrtl/src/passes/Passes.scala
package firrtl.passes

class PassException(message: String) extends FirrtlUserException(message)
```

```scala
// play-chisel/firrtl/src/FirrtlException.scala
import scala.util.control.NoStackTrace

class FirrtlUserException(message: String, cause: Throwable = null)
    extends RuntimeException(message, cause) with NoStackTrace
```

```scala
// play-chisel/firrtl/src/ir/IR.scala
object GroundType {
  def unapply(ground: GroundType): Option[Width] = Some(ground.width)
}
```

删掉之前打印的信息。

```scala
// play-chisel/src/Main.scala
object Main extends App {
  ...
  val firrtl = Converter.convert(circuit)

  val state = CircuitState(firrtl, ChirrtlForm)
  val compiler = new VerilogCompiler
  compiler.compile(state)
}
```

```shell
$ mill chisel3.run
```
{{% admonition tip 源码07 %}}
{{% /admonition %}}

[play-chisel/tree/chap02-07](https://github.com/colin4124/play-chisel/tree/chap02-07)

## 增加转换过程的详细信息

```scala
// play-chisel/firrtl/src/Compiler.scala
import debug.PrintIR._

abstract class SeqTransform extends Transform with SeqTransformBased {
  def execute(state: CircuitState): CircuitState = {
    val ret = runTransforms(state)
    if (transforms.nonEmpty) {
      println(s"======After ${this.getClass.getSimpleName}======")
      print_fir(ret.circuit)
      println("===================")
    }
    CircuitState(ret.circuit, outputForm)
  }
}
```

删掉 `src/Main.scala` 里的 `import PrintIR._`

```shell
$ mv src/PrintIR.scala firrtl/src/PrintIR.scala
```

```scala
// play-chisel/firrtl/src/PrintIR.scala
package debug
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码08 %}}
{{% /admonition %}}

[play-chisel/tree/chap02-08](https://github.com/colin4124/play-chisel/tree/chap02-08)


# 自定义

**Chisel** 默认生成的 **Verilog** 可能不符合想要的代码风格，自定义这章讲一些如何自定义生成的 **Verilog**。

## 合并冗余的节点

```scala
io.out := (io.sel & io.in1) | (~io.sel & io.in0)
```

会生成带中间变量的形式：

```
node _T = and(sel, in1)
node _T_1 = not(sel)
node _T_2 = and(_T_1, in0)
node _T_3 = or(_T, _T_2)
out <= _T_3
```

```verilog
wire  _T;
wire  _T_1;
wire  _T_2;
assign _T = sel & in1;
assign _T_1 = ~ sel;
assign _T_2 = _T_1 & in0;
assign out = _T | _T_2;
```

这种生成的代码很不直观，希望生成没有中间节点的形式：

```verilog
assign out = (sel & in1) | ((~ sel) & in0);
```

新建一个合并冗余节点的 `Pass`。

先收集每个中间节点对应的表达式，存到 ~HashMap~ 里；然后把中间节点都替换成对应的表达式；最后删掉中间节点的声明语句。

```shell
$ touch firrtl/src/passes/MergeNodes.scala
```

```scala
// play-chisel/firrtl/src/passes/MergeNodes.scala
package firrtl.passes

import firrtl._
import firrtl.ir._
import firrtl.Mappers._

object MergeNodes extends Pass {
  // 新建空的 HashMap
  type RefMap = collection.mutable.LinkedHashMap[String, Expression]

  // 解析表达式，如果是节点类型的引用，去 HashMap 里找到对应表达式
  def resolve_e(refs: RefMap)(e: Expression): Expression = e match {
    case WRef(name, _, kind, _) if kind == NodeKind => refs(name)
    case _ => e map resolve_e(refs)
  }

  // 解析语句，如果是节点定义的语句，把节点名字和表达式记录下来
  def resolve_s(refs: RefMap)(s: Statement): Statement = s match {
    case d@DefNode(name, value) =>
      val res = d map resolve_e(refs) map resolve_s(refs)
      res match {
        case DefNode(_, v) => refs(name) = v
      }
      res
    case sx =>
      sx map resolve_e(refs) map resolve_s(refs)
  }

  def merge_nodes(m: DefModule): DefModule = {
    val refs = new RefMap
    m map resolve_s(refs)
  }

  // 过滤语句列表，去掉定义节点的语句
  def remove_s(s: Statement): Statement = s match {
    case b@Block(stmts) =>
      b.copy(stmts = stmts filter {
               s => s match {
                 case _: DefNode => false
                 case _ => true
               }}) map remove_s
    case _ => s
  }

  def remove_nodes(m: DefModule): DefModule = {
    m map remove_s
  }

  def run(c: Circuit): Circuit = {
    val res = c copy (modules = c.modules map merge_nodes)
    res copy (modules = res.modules map remove_nodes)
  }
}
```

在 **LowFirrtlOptimization** 阶段加入合并冗余节点的 **Pass**。

```scala
// play-chisel/firrtl/src/LoweringCompiler.scala
class LowFirrtlOptimization extends CoreTransform {
  ...
  def transforms = Seq(
    passes.MergeNodes,
  )
}
```

修改下生成运算操作的参数。 ~checkArgumentLegality~ 检查参数的合法性新增了表达式类型（即中间节点对应的表达式），之前只能是一个引用（WRef，即中间节点的名字）。判断是不是同一个运算操作类型，用于是否需要加括号确保运算的优先级。

```scala
def op_stream(doprim: DoPrim): Seq[Any] = {
  ...
  def is_same_op(a: Expression) = a match {
    case d: DoPrim if (d.op != doprim.op) => false
    case _ => true
  }

  def checkArgumentLegality(e: Expression) = e match {
    case _: WRef | _: Expression =>
    case _ => throw EmitterException(s"Can't emit ${e.getClass.getName} as PrimOp argument")
  }

  doprim.op match {
    case Not =>
      if (is_same_op(a0)) Seq("~ ", a0)
      else Seq("~ ", "(", a0, ")")
    case And =>
      val a0_seq = if (is_same_op(a0)) Seq(cast_as(a0)) else Seq("(", cast_as(a0), ")")
      val a1_seq = if (is_same_op(a1)) Seq(cast_as(a1)) else Seq("(", cast_as(a1), ")")
      a0_seq ++ Seq(" & ") ++ a1_seq
    case Or =>
      val a0_seq = if (is_same_op(a0)) Seq(cast_as(a0)) else Seq("(", cast_as(a0), ")")
      val a1_seq = if (is_same_op(a1)) Seq(cast_as(a1)) else Seq("(", cast_as(a1), ")")
      a0_seq ++ Seq(" | ") ++ a1_seq
  }
}
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码 custom 01 %}}
{{% /admonition %}}

[play-chisel/tree/custom-01](https://github.com/colin4124/play-chisel/tree/custom-01)
