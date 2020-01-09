---
title: "第一章 开始构建 Chisel"
date: 2019-11-30T16:14:50+08:00
draft: false
---

本章从一个最简单的多路选择器入手，讲解如何用 Mill 搭建项目的构建环境，开发满足生成多路选择器模块的 Chisel 基本框架。

# 目标

二选一的多路选择器用 **Chisel** 来写，代码如下：

```scala
class Mux2 extends RawModule {
  val sel = IO(Input(UInt(1.W)))
  val in0 = IO(Input(UInt(1.W)))
  val in1 = IO(Input(UInt(1.W)))
  val out = IO(Output(UInt(1.W)))
  io.out := (io.sel & io.in1) | (~io.sel & io.in0)
}
```

这章的目标就是生成如下的 **FIRRTL** `Mux2.fir`：
```firrtl
circuit Mux2 :
  module Mux2 :
    input sel : UInt<1>
    input in0 : UInt<1>
    input in1 : UInt<1>
    output out : UInt<1>

    node _T = and(sel, in1)
    node _T_1 = not(sel)
    node _T_2 = and(_T_1, in0)
    node _T_3 = or(_T, _T_2)
    out <= _T_3
```

最后再用 **FIRRTL** 生成如下的 **Verilog** `Mux2.v`：
```verilog
module Mux2(
  input   sel,
  input   in0,
  input   in1,
  output  out
);
  wire  _T;
  wire  _T_1;
  wire  _T_2;
  assign _T = sel & in1;
  assign _T_1 = ~ sel;
  assign _T_2 = _T_1 & in0;
  assign out = _T | _T_2;
endmodule
```

# Mill 环境搭建

[Mill 的安装方式](http://www.lihaoyi.com/mill/)

新建文件夹 `play-chisel` ，定义 `build.sc`。

```shell
$ mkdir play-chisel
$ cd play-chisel
$ touch build.sc
```

为了更清楚地表示路径，都会加上 `play-chisel` 来表示项目里的绝对路径，并在每个文件的顶部加上绝对路径的注释。

例如，`build.sc` 的绝对路径是 `play-chisel/build.sc`。

```scala
// play-chisel/build.sc
import ammonite.ops._
import mill._
import mill.scalalib._

trait CommonChiselModule extends ScalaModule {
  def scalaVersion = "2.12.10"
  override def ivyDeps = Agg(
    ivy"edu.berkeley.cs::firrtl:1.2.0"
  )
}

object chisel3 extends CommonChiselModule {
  object chiselFrontend extends CommonChiselModule
  def moduleDeps = Seq(chiselFrontend)
  def millSourcePath = super.millSourcePath / ammonite.ops.up
}
```

这里用的 **Scala** 版本是 `2.12.10`，依赖 **firrtl** 的 `1.2.0` 版本。

项目名字叫 **chisel3** ，在使用 **Mill** 进行编译或者是运行时，需要带上项目的名字。

- 编译源代码：`mill chisel3.compile`
- 编译并运行：`mill chisel3.run`

子项目是 **chiselFrontend**，因为用到宏 **Macro** ，需要把宏相关的放到一个单独的编译项目。需要先编译宏定义的项目之后，才去编译使用宏的项目。虽然目前不会用到宏，但还是保留了这样一个项目结构。

`def moduleDeps = Seq(chiselFrontend)` 定义了编译顺序。

即先编译 `chiselFrontend` (`play-chisel/chiselFrontend/src` 的源码) 再轮到 `chisel3`（`play-chisel/chisel3/src`的源码）。

源码默认是需要放在 `play-chisel/chisel3/src` 的目录里，但 `def millSourcePath = super.millSourcePath / ammonite.ops.up` 让寻找源代码的路径往上挪了一级。也就是说，原来 **Mill** 是去 `play-chisel/chisel3` 寻找 `src` 目录，现在改成去 `play-chisel` 找。

新建文件夹 `src`，编辑 `Main.scala`，用来存放程序的入口函数，相当于 **C** 语言的 `main` 函数。

```shell
$ mkdir src
$ touch src/Main.scala
```

**Scala** 是靠代码里定义了 `def main` 方法的对象或是继承了 `App` 的对象来作为程序执行的入口，下面是最简单的 *Hello, world* 。

```scala
// play-chisel/src/Main.scala
package playchisel

object Main extends App {
  println("Hello, world")
}
```

执行 *Mill* 构建项目并运行：

```shell
$ mill chisel3.run
```

{{% admonition tip 源码01 %}}
{{% /admonition %}}

[play-chisel/tree/chap01-01](https://github.com/colin4124/play-chisel/tree/chap01-01)


# 搭建基本的 Chisel 框架

这一小节主要是把 **Chisel** 需要的基本类型都简单地定义一遍，为了方便通过编译，基本上都是定义了类型，满足编译器的类型检查，不会做有实际意义的事情。

## Main.scala

`Main.scala` 做了四件事情：

1. 继承 `RawModule` 定义二选一的多路选择器；
2. 调用 `Builder.build` 方法构建出中间语言表示（**IR**）的电路；
3. 调用 `Emitter.emit` 方法解析 **IR** 生成 **FIRRTL** 字符串；
4. 例化 `FileWriter` 把字符串写入文件。

二选一的多路选择器 `Mux2` 只是简单地继承了 `RawModule`，具体要做的事情留到下一小节。

```scala
// play-chisel/src/Main.scala
package playchisel

import chisel3._
import chisel3.internal.Builder
import chisel3.internal.firrtl._

import java.io.{File, FileWriter}

class Mux2 extends RawModule

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Mux2))

  val emitted = Emitter.emit(circuit)

  val file = new File("Mux2.fir")
  val w = new FileWriter(file)
  w.write(emitted)
  w.close()
}
```

## Builder

新建文件夹 `chiselFrontend/src/internal`。

```shell
$ mkdir -p chiselFrontend/src/internal
$ touch chiselFrontend/src/internal/Builder.scala
```

`Builder.scala` 除了 `object Builder` 之外，还有 `trait HasId`。 **Chisel** 相关的元素类型都会继承于 `HasId`。

`object Builder` 定义了跟生成电路 **IR** 相关的变量和方法，目前先简单定义 `def build` 方法，只是满足返回值的类型检查。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
package chisel3.internal

import chisel3._
import chisel3.internal.firrtl._

private[chisel3] trait HasId

object Builder {
  def build[T <: RawModule](f: => T): (Circuit, T) = {
    println("Elaborating design...")
    val mod = f
    println("Done elaborating.")
    (Circuit("", Seq()), mod)
  }
}
```

`Builder.build` 方法的参数是 `bc: => T` ，也就是引用传递，不会在传递参数的时候立刻执行，而是等到使用这个参数的时候。因此，`Builder` 的属性初始化之后（这里目前只有输出一句 `"Elaborating design..."`)，它的参数才会被执行 `val mod = f` 。

返回值类型是`（Circuit, T)` , 也就是返回中间语言表示（**IR**）的电路 `Circuit` 以及模块自身，类型必须是 `RawModule` 的子类（`T<:RawModule`) 。 `(Circuit("", Seq()), mod)` 目前返回空的电路 `Circuit("", Seq())` 和参数中例化过的模块 `mod` 。

这里简单解释下电路 **IR** `Circuit("", Seq())`，第一个参数是电路的名字，为空字符串；第二个参数是电路里的 **Verilog** 模块列表，目前为空，什么模块也还没有。

## Module

```shell
$ touch chiselFrontend/src/Module.scala
```

`Module.scala` 除了 `object Module` 之外还有放在 `package experimental` 里用来生成 **Verilog** 模块端口的 `object IO` 和 **Verilog** 模块的基类 `BaseModule` 。

``` scala
// play-chisel/chiselFrontend/src/Module.scala
package chisel3

import chisel3.internal._

object Module {
  def apply[T <: BaseModule](bc: => T): T = {
    val module: T = bc
    module
  }
}

package experimental {
  object IO {
    def apply[T<:Data](iodef: T): T = {
      iodef
    }
  }
}

abstract class BaseModule extends HasId
```

`object Module` 目前只有一个 `apply` 方法，接受的参数类型是 `BaseModule` 的子类，`=> T` 表示引用类型，传参的时候不会执行，等到执行了 `val module: T = bc` 才执行。目前什么也没有做，只是满足类型检查，返回参数自身。

`object IO` 也是什么也不做，仅仅是返回参数自身。

`BaseModule` 也只是继承了 `HasId` 而已（**Chisel** 相关的元素都会继承 `HasId`）。

## RawModule

```shell
$ touch chiselFrontend/src/RawModule.scala
```

生成的 **Verilog** 模块分为两种，一种是定义了端口和实现的模块，比如这个例子里的二选一多路选择器；另一种是只定义端口，没有实现，用于例化子模块，比如 **BlackBox**。

**RawModule** 属于定义了端口和实现的 **Verilog** 模块的基类。不会像 **Module**（**RawModule** 的子类）帮用户定义好了时钟 `clock` 和同步复位 `reset`。

```scala
// play-chisel/chiselFrontend/src/RawModule.scala
package chisel3

abstract class RawModule extends BaseModule
```

`RawModule.scala` 只有 `abstract class RawModule` ，`RawModule` 也只是继承了 `BaseModule` ，什么也不做。

## IR

```shell
$ mkdir -p chiselFrontend/src/internal/firrtl
$ touch chiselFrontend/src/internal/firrtl/IR.scala
```

**Chisel** 会把用户写的模块信息先收集好，比如端口信息，电路的基本元素（**Wire**， **Reg**），以及它们的连接方式等等，存到一组定义好的数据结构里统一起来，在用户写的 **Chisel** 和生成的 **FIRRTL** 中间多了一层，成为 **IR**（Intermediate Representation），中间表示或中间语言表示。

```scala
// play-chisel/src/internal/firrtl/IR.scala
package chisel3.internal.firrtl

sealed abstract class Width
sealed case class UnknownWidth() extends Width
sealed case class KnownWidth(value: Int) extends Width

object Width {
  def apply(x: Int): Width = KnownWidth(x)
  def apply(): Width = UnknownWidth()
}

abstract class Component
case class Circuit(name: String, components: Seq[Component])
```

`IR.scala` 主要存放的是中间语言表示（**IR**）相关的数据类型。

`Width` 表示的是 **Verilog** 数据宽度，未知宽度 `UnknownWidth` 和已知宽度 `KnownWidth`。

`Component` 表示的是 `Verilog` 里的模块。

`Circuit` 电路包括了自己的名字 `name: String`和它里面包含的所有 **Verilog** 模块 `components: Seq[Component]`。

## Data.scala
```shell
$ touch chiselFrontend/src/Data.scala
```

`Data` 是定义 **Verilog** 数据的基类。

```scala
// play-chisel/chiselFrontend/src/Data.scala
package chisel3

import chisel3.internal._
import chisel3.internal.firrtl._

object Input {
  def apply[T<:Data](source: T): T = {
    source
  }
}
object Output {
  def apply[T<:Data](source: T): T = {
    source
  }
}
abstract class Data extends HasId {
  private[chisel3] def connect(that: Data): Unit = {}
  private[chisel3] def width: Width
  final def := (that: Data): Unit = this.connect(that)
}
```

`Data.scala` 里有 `Input` 和 `Output` 用来定义端口是输入还是输出。这里也只是返回参数，没有做其他的事情。

`abstract class Data` 继承了 `HasId` ，它是有宽度 `width`的，`def :=` 方法是方便用户调用 `def connect` 方法的 **API**，用来把 `:=` 两边的数据连接到一起，相当于 **Verilog** 里的赋值语句。

## Element.scala

```shell
$ touch chiselFrontend/src/Element.scala
```

**Element** 是一个“叶子” **Verilog** 数据类型：它不会包含其他``Data`` 类派生的 **Verilog** 数据类型，用来表示原语（ *primitive data types* ）的数据类型，像整型（ *integers* ）和比特类型（ *bits* ）。

```scala
// play-chisel/chiselFrontend/src/Element.scala
package chisel3
abstract class Element extends Data
```

`Element` 这里只是继承了 `Data`，没有做其他的事情。

## Num.scala

```shell
$ touch chiselFrontend/src/Num.scala
```

`trait Num` 里定义了一系列用于 **Verilog** 数值类型的操作，目前这里为空。

```scala
// play-chisel/chiselFrontend/src/Num.scala

package chisel3
trait Num[T <: Data] {
  self: Num[T] =>
}
```

`T<:Data` 和 `self: Num[T] =>` 在继承的时候限定了里面定义的方法只能是用于某一个 `Data`子类的数据类型，比如 `class UInt extends Num[UInt]`，那么这些定义的方法只能用于 `UInt`。

## Bits.scala

```shell
$ touch chiselFrontend/src/Bits.scala
```

`Bits` 表示该数据类型是有一个或多个比特的，同时定义了基本的位运算方法。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
package chisel3

import chisel3.internal.firrtl._

sealed abstract class Bits(private[chisel3] val width: Width) extends Element

sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  override def toString: String = {
    s"UInt$width"
  }

  final def & (that: UInt): UInt = that
  final def | (that: UInt): UInt = that
  def unary_~ (): UInt = UInt(width = width)
}
```

`Bits.scala` 定义了对应到 **Verilog** 里的数据类型，目前只有无符号整型 `UInt` 一种，并定义了两个 `UInt` 数据类型之间可以进行与 `&`、 或 `|` 、非 `~` 三种位运算（ *Bitwise* ）。

## UIntFactory.scala

```shell
$ touch chiselFrontend/src/UIntFactory.scala
```

`UIntFactory` 方便用户创建 `UInt` 类型。咧威猜测为什么不直接定义 `object UInt` 而是先定义了 `UIntFactory` 再 `object UInt extends UIntFactory`，可能是因为 `UInt` 和 `Bits` 等价。把它俩做的事情抽象成独立的 `trait` ，然后继承了 `UIntFactory` 就好了：`object UInt extends UIntFactory` 和 `object Bits extends UIntFactory`。不过这里 `Bits` 先不细讲。

```scala
// play-chisel/chiselFrontend/src/UIntFactory.scala
package chisel3

import chisel3.internal.firrtl.Width

trait UIntFactory {
  def apply(): UInt = apply(Width())
  def apply(width: Width): UInt = new UInt(width)
}
```

不带宽度参数的 `apply()` 是创建让 **Chisel** 帮用户自动推导宽度的 `UInt`（自动推导宽度这里先不做），`apply(width: Width)`是根据用户指定的宽度创建 `UInt`。

## package.scala

```shell
$ touch chiselFrontend/src/package.scala
```

`package object chisel3` 定义的是在 `chisel3` 这个包的全局变量和函数。

```scala
// play-chisel/chiselFrontend/src/package.scala
package object chisel3 {
  import internal.firrtl.{Width}

  import scala.language.implicitConversions

  implicit class fromIntToWidth(int: Int) {
    def W: Width = Width(int)
  }

  object UInt extends UIntFactory
}
```

伴生对象（**Companions**）是要定义同一个文件里的，也就是说， `Bits.scala` 里的 `sealed class UInt` 的伴生对象 `object UInt` 只能定义在同一个文件 `Bits.scala` 里，否则会报错 `error: Companions 'class UInt' and 'object UInt' must be defined in same file` 。为了规避这个问题，只需要 `import scala.language.implicitConversions` 即可。

`implicit class` 的构造函数只支持一个参数，这里是整型 `Int` 。 当编译器遇到 `5.W` ，就会去找参数是 `Int` 类型的隐式 `class` ，这里是 `implicit class fromIntToWidth`。找到之后，再去这里个类里找名字叫 `W` 的方法，找到了就会调用执行，因此 `5.W` 相当于是 `Width(5)`。

对于 `implicit class` 这里再举个简单直观的例子，打开 **Scala**，按照下面步骤自己尝试一遍。这里给整型 `Int` 定义一个方法`S`，将一个整型转换成字符串： `10.S` 输出 `"10"`。

一开始输入 `10.S` 会报错，因为我们还没有定义 `S` 方法。

```shell
scala> 10.S
          ^
       error: value S is not a member of Int

scala>
```

接下来定义 `implicit class fromIntToString` 类，里面再定义一个 `def S: String` 方法，没有参数，方法体里面用的是隐式类 `implicit class fromIntToString` 的参数 `int: Int`，调用 `Int` 类型预先定义好的 `toString` 方法，把整型转换成字符串型。


```shell
scala> implicit class fromIntToString(int: Int) {
     |   def S: String = int.toString
     | }
defined class fromIntToString

scala>
```

当我们再次输入`10.S` ，就能输出 `"10"`了。

```shell
scala> 10.S
res1: String = 10

scala>
```

## Emitter.scala

```shell
$ mkdir -p src/internal/firrtl
$ touch src/internal/firrtl/Emitter.scala
```

`Emitter` 里没有涉及到宏，按照 **Chisel** **IR** 划分的话，属于解析 **IR** 的后端，因此源码不是放在 `chiselFrontend` 子项目里，而是 `chisel3` 项目里。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala

package chisel3.internal.firrtl
import chisel3._

object Emitter {
  def emit(circuit: Circuit): String = new Emitter(circuit).toString
}

private class Emitter(circuit: Circuit) {
  override def toString: String = "Hello, chisel3"
}
```

`Emitter.scala` 是把中间语言表示 **IR** 的电路转换成 **FIRRTL**, 目前什么也没有做，只是返回 `"Hello, chisel3"`。
当执行下面命令的时候，会生成 `Mux2.fir` 文件，里面只有一句 `"Hello, chisel3"`。

```shell
$ mill chisel3.run
```
{{% admonition tip 源码02 %}}
{{% /admonition %}}

[play-chisel/tree/chap01-02](https://github.com/colin4124/play-chisel/tree/chap01-02)

# 自底向上继续完善 Chisel

把 `Main.scala` 的 `class Mux2` 补充完整：

```scala
// play-chisel/src/Main.scala
class Mux2 extends RawModule {
  val sel = IO(Input(UInt(1.W)))
  val in0 = IO(Input(UInt(1.W)))
  val in1 = IO(Input(UInt(1.W)))
  val out = IO(Output(UInt(1.W)))
  io.out := (io.sel & io.in1) | (~io.sel & io.in0)
}
```

接下来会根据在 `class Mux2` 里的出现顺序，依次完善 `IO` 、`Input/Output`、与运算`&`、或运算`|`、非运算`~`和连接操作`:=`。

## IO 绑定

模块的每个端口都需要调用 `IO()` 记录下来。 `object IO` 定义在 `Module.scala` 文件里的 `package experimental`。

``` scala
// play-chisel/chiselFrontend/src/Module.scala
package experimental {
  object IO {
    def apply[T<:Data](iodef: T): T = {
      val module = Module.currentModule.get
      require(!module.isClosed, "Can't add more ports after module close")
      requireIsChiselType(iodef, "io type")

      val iodefClone = iodef.cloneTypeFull
      module.bindIoInPlace(iodefClone)
      iodefClone
    }
  }
}

abstract class BaseModule extends HasId {
  ...
  protected def IO[T <: Data](iodef: T): T = chisel3.experimental.IO.apply(iodef)
}
```

`IO()` 参数 `iodef` 的类型是 `Data` 的子类。

先获取当前的模块，检查是否为关闭状态。当模块例化完成之后，也就是当 `new Mux2` 生成对象，调用它自身的 `generateComponent` 方法时，模块才会处于关闭状态，即 `isClosed` 为 `true`。

`requireIsChiselType` 要求这个数据类型是 `chisel type` ，也就是说它还不是硬件类型（可综合的）。对应到 **Verilog**，`16'h2333` 是 `chisel type` ，而 `reg [15:0] foo` 是 `hardware type` 。

`IO()` 不会改变原有的参数，而是调用 `cloneTypeFull` 方法复制出一份新的，再调用模块的 `bindIoInPlace` 方法，把端口绑定到当前模块上。

最后返回参数的复制品。


### 当前模块

`object Builder` 会记录一个构建过程中的当前模块，表示当前正在构建的模块是哪一个。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
object Builder {
  var currentModule: Option[BaseModule] = None
  ...
}
```

`object Module` 的 `def currentModule` 方法拿到的是当前 `object Builder` 的 `currentModule` 变量的值。

```scala
// play-chisel/chiselFrontend/src/Module.scala
object Module {
  ...
  def currentModule: Option[BaseModule] = Builder.currentModule
}
```

每次例化一个新的模块时， `object Builder` 的 `currentModule` 变量都会指向它。

```scala
// play-chisel/chiselFrontend/src/Module.scala
abstract class BaseModule extends HasId {
  Builder.currentModule = Some(this)
  ...
}
```

### 模块的关闭状态

模块的关闭状态是用来区分例化用户定义的模块和生成模块 **IR** 这两个阶段。例化用户定义的模块（比如这里的 `Mux2`），会收集模块定义相关的信息，比如定义了哪些端口，哪些硬件（**reg** 、 **wire**)，它们之间如何连线等等，这个阶段发生在模块关闭之前；这之后，等到调用模块的 `generateComponent` 方法生成 **Verilog** 的模块 **IR** 时，处于模块关闭状态。

```scala
import chisel3.internal.firrtl._

abstract class BaseModule extends HasId {
  ...
  protected var _closed = false
  private[chisel3] def isClosed = _closed
  ...
  private[chisel3] def generateComponent(): Component
}
```

模块初始化的时候，`isClosed` 是 `false`，只有等到调用 `generateComponent` 生成中间语言表示 **IR** 的模块时，`isClosed` 才设置为 `true`。

```scala
import chisel3.internal.firrtl._

abstract class RawModule extends BaseModule {
  def generateComponent(): Component = {
    require(!_closed, "Can't generate module more than once")
    _closed = true
    new Component {}
  }
}
```
### ChiselType & HardwareType

这两种类型的判断依据是数据的可综合属性 `isSynthesizable`，可综合的为硬件类型，不可综合的为 **Chisel** 类型。

```scala
// play-chisel/chiselFrontend/src/internal/Binding.scala
package chisel3.internal
import chisel3._

object requireIsHardware {
  def apply(node: Data, msg: String = ""): Unit = {
    if (!node.isSynthesizable) {
      val prefix = if (msg.nonEmpty) s"$msg " else ""
      throw ExpectedHardwareException(s"$prefix'$node' must be hardware, " +
        "not a bare Chisel type. Perhaps you forgot to wrap it in Wire(_) or IO(_)?")
    }
  }
}

object requireIsChiselType {
  def apply(node: Data, msg: String = ""): Unit = if (node.isSynthesizable) {
    val prefix = if (msg.nonEmpty) s"$msg " else ""
    throw ExpectedChiselTypeException(s"$prefix'$node' must be a Chisel type, not hardware")
  }
}
```

`requireIsHardware` 和 `requireIsChiselType` 用来检查是否符合各自的类型。

```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  private var _binding: Option[Binding] = None
  protected[chisel3] def binding: Option[Binding] = _binding
  protected def binding_=(target: Binding) {
    if (_binding.isDefined) {
      throw RebindingException(s"Attempted reassignment of binding to $this")
    }
    _binding = Some(target)
  }

  private[chisel3] def bind(target: Binding, parentDirection: SpecifiedDirection = SpecifiedDirection.Unspecified)

  private[chisel3] final def isSynthesizable: Boolean = _binding.map {
    case _: TopBinding => true
  }.getOrElse(false)
  ...
}
```

`Data` 数据是否可综合是根据它的 `_binding` 属性，即它被绑定成了什么类型。目前只有 `TopBinding` 绑定类型才算是可综合的硬件类型。

这里的 `def binding` 和 `def binding_=` 类似于 `var binding` 。区别在于写的操作上，当赋值给 `binding` 时，会调用 `binding_=` 方法，检查是否重复绑定了。

这里只是声明了 `def bind` 绑定方法，具体的定义留个它的子类。

```scala
// play-chisel/chiselFrontend/src/Element.scala
package chisel3

import chisel3.internal._

abstract class Element extends Data {
  private[chisel3] override def bind(target: Binding, parentDirection: SpecifiedDirection) {
    binding = target
    val resolvedDirection = SpecifiedDirection.fromParent(parentDirection, specifiedDirection)
    direction = ActualDirection.fromSpecified(resolvedDirection)
  }
}
```

目前 `Data` 的子类只有 `Element`。`Element` 类型的数据 `bind` 绑定方法，需要指定绑定的类型 `target: Binding`，还设置了数据的方向。

{{% admonition tip "注意" %}}
{{% /admonition %}}

```scala
abstract class Data extends HasId {
  ...
  private[chisel3] def bind(target: Binding, parentDirection: SpecifiedDirection = SpecifiedDirection.Unspecified)
}
abstract class Element extends Data {
  ...
  private[chisel3] override def bind(target: Binding, parentDirection: SpecifiedDirection) {
    ...
  }
}
```

`def bind` 在 `abstract class Data` 里的定义，参数 `parentDirection` 是有默认值的，默认父类的指定方向是未定义 `SpecifiedDirection.Unspecified`。 但在 `abstract class Element` 的定义里，没有显性地指定的默认值，那么这里是会继承在 `abstract class Data` 中给定的默认值 `SpecifiedDirection.Unspecified`。

说到方向，先来定义下数据的方向属性：

1. 用户定义的方向 **SpecifiedDirection** ；
2. 被绑定后根据用户指定的（`specifiedDirection`）解析过的实际方向 **ActualDirection**。

它们都是定义在 `abstract class Data` 的自身属性。

```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  private var _specifiedDirection: SpecifiedDirection = SpecifiedDirection.Unspecified

  def specifiedDirection: SpecifiedDirection = _specifiedDirection
  def specifiedDirection_=(direction: SpecifiedDirection) = {
    if (_specifiedDirection != SpecifiedDirection.Unspecified) {
      throw RebindingException(s"Attempted reassignment of user-specified direction to $this")
    }
    _specifiedDirection = direction
  }
  
  private var _direction: Option[ActualDirection] = None

  private[chisel3] def direction: ActualDirection = _direction.get
  private[chisel3] def direction_=(actualDirection: ActualDirection) {
    if (_direction.isDefined) {
      throw RebindingException(s"Attempted reassignment of resolved direction to $this")
    }
    _direction = Some(actualDirection)
  }
  ...
}
```

它们都用了相同方法名加 `_=` 的方式：`def specifiedDirection` 用作读， `def specifiedDirection_=(direction: SpecifiedDirection)` 用作写的方式，来对写入进行检查。

举个例子，打开 **Scala** 的命令行模式（**REPL**）：

```shell
scala> object Num {
    var _foo = 0
    
    def foo = _foo
    def foo_=(n: Int) = {
      if (n < 0)
        println("n should not less than zero!")
      else
        _foo = n
    }
  } 
defined object Num
scala> Num.foo 
res1: Int = 0
scala> Num.foo = 2 
scala> Num.foo 
res3: Int = 2
scala> Num.foo = -1 
n should not less than zero!
scala > Num.foo 
res5: Int = 2
```

方向不能多次绑定，否则会报错。下面是定义在 `package.scala` 的报错类型：

```scala
// play-chisel/chiselFrontend/src/package.scala
package object chisel3 {
  ...
  type ChiselException = internal.ChiselException
  class BindingException(message: String) extends ChiselException(message)
  case class RebindingException(message: String) extends BindingException(message)
  case class ExpectedChiselTypeException(message: String) extends BindingException(message)
  case class ExpectedHardwareException(message: String) extends BindingException(message)
}
```

`ChiselException` 是定义在 `Error.scala` 中。

```scala
// play-chisel/chiselFrontend/src/internal/Error.scala
package chisel3.internal

class ChiselException(message: String, cause: Throwable = null) extends Exception(message, cause)
```

### 绑定的类型 Binding

目前的绑定类型只有 `TopBinding`，也就是可综合的（硬件类型的）绑定。 `location` 记录的是所在的模块。

```scala
// play-chisel/chiselFrontend/src/internal/Binding.scala
sealed trait Binding {
  def location: Option[BaseModule]
}
sealed trait TopBinding extends Binding
```
### 复制方法 cloneTypeFull

除了会根据原来的类型实例化新的对象之外，还会复制它的指定方向属性 `specifiedDirection` 。
```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  def cloneType: this.type

  private[chisel3] def cloneTypeFull: this.type = {
    // get a fresh object, without bindings
    val clone = this.cloneType.asInstanceOf[this.type]
    // Only the top-level direction needs to be fixed up, cloneType should do the rest
    clone.specifiedDirection = specifiedDirection
    clone
  }
}
```

带有宽度的 `Bits` 数据类型，它的复制方法是连宽度一块复制。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
import chisel3.internal.firrtl._

sealed abstract class Bits(private[chisel3] val width: Width) extends Element {
  private[chisel3] def cloneTypeWidth(width: Width): this.type

  def cloneType: this.type = cloneTypeWidth(width)
}
```
`def cloneTypeWidth` 在 `Bits` 的子类里定义，比如 `UInt`：

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  private[chisel3] override def cloneTypeWidth(w: Width): this.type =
    new UInt(w).asInstanceOf[this.type]
  ...
}
```

### 绑定IO端口

绑定 IO 端口的方法 `bindIoInPlace` 先把端口数据 `iodef: Data` 注册为端口类型 `PortBinding`，然后添加到该模块的端口列表里 `_ports` 。**注意**：这里的 `iodef.bind()` 没有给定第二个 `parentDirection` 参数，用的是默认值 `SpecifiedDirection.Unspecified` 。

```scala
// play-chisel/chiselFrontend/src/Module.scala
abstract class BaseModule extends HasId {
  protected def _bindIoInPlace(iodef: Data): Unit = {
    iodef.bind(PortBinding(this))
    _ports += iodef
  }

  private[chisel3] def bindIoInPlace(iodef: Data): Unit = _bindIoInPlace(iodef)
  ...
}
```

端口的绑定类型记录下被绑定到哪个模块上 `enclosure: BaseModule`。

```scala
// Binding.scala
sealed trait ConstrainedBinding extends TopBinding {
  def enclosure: BaseModule
  def location: Option[BaseModule] = Some(enclosure)
}

case class PortBinding(enclosure: BaseModule) extends ConstrainedBinding
```

`_ports` 记录了该模块的端口列表，获取端口列表通过 `getModulePorts` 方法，并且只能在关闭状态，也就是 `generateComponent` 方法里面才能调用。

```scala
// play-chisel/chiselFrontend/src/Module.scala
import scala.collection.mutable.ArrayBuffer

abstract class BaseModule extends HasId {
  ...
  private val _ports = new ArrayBuffer[Data]()

  protected[chisel3] def getModulePorts = {
    require(_closed, "Can't get ports before module close")
    _ports.toSeq
  }
  ...
}
```

## Input/Output

`object Input` 和 `object Output` 是让用户改变数据的方向，即改变数据的 `specifiedDirection` 属性。

```scala
// play-chisel/chiselFrontend/src/Data.scala
object Input {
  def apply[T<:Data](source: T): T = {
    SpecifiedDirection.specifiedDirection(source)(SpecifiedDirection.Input)
  }
}
object Output {
  def apply[T<:Data](source: T): T = {
    SpecifiedDirection.specifiedDirection(source)(SpecifiedDirection.Output)
  }
}
```

`Input` 是调用 `specifiedDirection` 方法改变数据的方向为输入，`Output` 是调用 `specifiedDirection` 方法改变数据的方向为输出。

`SpecifiedDirection` 类型是由用户指定的方向类型，分别为未定义 `Unspecified`、输出 `Output`、输入 `Input` 和反转 `Flip` 。

```scala
// play-chisel/chiselFrontend/src/Data.scala
sealed abstract class SpecifiedDirection
object SpecifiedDirection {
  case object Unspecified extends SpecifiedDirection
  case object Output      extends SpecifiedDirection
  case object Input       extends SpecifiedDirection
  case object Flip        extends SpecifiedDirection

  def flip(dir: SpecifiedDirection): SpecifiedDirection = dir match {
    case Unspecified => Flip
    case Flip        => Unspecified
    case Output      => Input
    case Input       => Output
  }

  def fromParent(parentDirection: SpecifiedDirection, thisDirection: SpecifiedDirection): SpecifiedDirection =
    (parentDirection, thisDirection) match {
      case (SpecifiedDirection.Output, _) => SpecifiedDirection.Output
      case (SpecifiedDirection.Input, _) => SpecifiedDirection.Input
      case (SpecifiedDirection.Unspecified, thisDirection) => thisDirection
      case (SpecifiedDirection.Flip, thisDirection) => SpecifiedDirection.flip(thisDirection)
    }

  private[chisel3] def specifiedDirection[T<:Data](source: T)(dir: SpecifiedDirection): T = {
    val out = source.cloneType.asInstanceOf[T]
    out.specifiedDirection = dir
    out
  }
}
```

反转方法 `def flip` 是把未定义变成反转，反转变成未定义，输出变成输入，输入变成输出。

根据父类的和当前的指定方向来决定最终的指定方向 `def fromParent`，如果父类的指定方向是输入/输出，不用考虑当前的指定方向，得到的是对应的输入/输出；如果父类未定义，则按照当前的指定方向；如果父类是反转类型，那么要相应地反转当前的指定方向。

指定方向的方法 `def specifiedDirection` 是把 `Data` 的子类复制一份，然后把复制品的方向改变为指定的方向类型。

实际方向类型有空 `Empty` 、未定义 `Unspecified` 、输出 `Output` 和输入 `Input`。

```scala
// play-chisel/chiselFrontend/src/Data.scala
sealed abstract class ActualDirection
object ActualDirection {
  case object Empty       extends ActualDirection
  case object Unspecified extends ActualDirection
  case object Output      extends ActualDirection
  case object Input       extends ActualDirection

  def fromSpecified(direction: SpecifiedDirection): ActualDirection = direction match {
    case SpecifiedDirection.Unspecified | SpecifiedDirection.Flip => ActualDirection.Unspecified
    case SpecifiedDirection.Output => ActualDirection.Output
    case SpecifiedDirection.Input => ActualDirection.Input
  }
}
```

从指定方向变成真正的方向类型方法 `def fromSpecified` 会把未定义、反转的指定类型变成未定义的实际方向类型，把输出的指定类型变成输出的实际类型，把输入的指定类型变成输入的实际类型。

## Command 命令

**Chisel** 会把生成 **Verilog** 模块里定义的运算语句，比如这里的位运算，以及赋值语句称为一条命令（**Command**）。

### 位运算

与、或位运算都是二元运算（*binop: binary operation*)， 非位运算是一元运算（*unop: unary operation*) 。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
import chisel3.internal.Builder.pushOp
import chisel3.internal.firrtl.PrimOp._

sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  final def & (that: UInt): UInt =
    binop(UInt(this.width max that.width), BitAndOp, that)
  final def | (that: UInt): UInt =
    binop(UInt(this.width max that.width), BitOrOp, that)

  def unary_~ (): UInt =
    unop(UInt(width = width), BitNotOp)
}
```

二元运算结果的宽度取自两个操作数的宽度最大值。

计算出两个宽度 `Width` 最大值的 `max` 方法：

```scala
// play-chisel/src/internal/firrtl/IR.scala
sealed abstract class Width {
  type W = Int
  def max(that: Width): Width = this.op(that, _ max _)

  protected def op(that: Width, f: (W, W) => W): Width
}
sealed case class UnknownWidth() extends Width {
  def op(that: Width, f: (W, W) => W): Width = this
  override def toString: String = ""
}
sealed case class KnownWidth(value: Int) extends Width {
  def op(that: Width, f: (W, W) => W): Width = that match {
    case KnownWidth(x) => KnownWidth(f(value, x))
    case _ => that
  }
  override def toString: String = s"<${value.toString}>"
}
```

二元运算与一元运算的定义：

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed abstract class Bits(private[chisel3] val width: Width) extends Element {
  ...
  private[chisel3] def unop[T <: Data](dest: T, op: PrimOp): T = {
    requireIsHardware(this, "bits operated on")
    pushOp(DefPrim(dest, op, this.ref))
  }

  private[chisel3] def binop[T <: Data](dest: T, op: PrimOp, other: Bits): T = {
    requireIsHardware(this, "bits operated on")
    requireIsHardware(other, "bits operated on")
    pushOp(DefPrim(dest, op, this.ref, other.ref))
  }
}
```

- 一元运算是生成一个运算结果、运算类型和操作数（自身）的三元组 `DefPrim(dest, op, this.ref)` 命令；
- 二元运算是生成一个运算结果、运算类型、操作数（自身)和操作数（另一个）的四元组 `DefPrim(dest, op, this.ref, other.ref)` 命令。

一元和二元运算是把运算结果，运算类型，以及参数列表放在了 `DefPrim` 数据类型里。

```scala
// play-chisel/src/internal/firrtl/IR.scala
import chisel3._
import chisel3.internal._

abstract class Command
abstract class Definition extends Command  {
  def id: HasId
  def name: String = id.getRef.name
}
case class DefPrim[T <: Data](id: T, op: PrimOp, args: Arg*) extends Definition
```

`PrimOp` 基本运算类型，目前只有 `BitAndOp` 、 `BitOrOp` `BitNotOp` 

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class PrimOp(name: String) {
  override def toString: String = name
}

object PrimOp {
  val BitAndOp = PrimOp("and")
  val BitOrOp  = PrimOp("or")
  val BitNotOp = PrimOp("not")
}
```

`this.ref` 和 `other.ref`里的引用属性 `ref` 是根据 `topBindingOpt` 方法来决定是否为节点 `Node` 类型。

```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  ...
  private[chisel3] def topBindingOpt: Option[TopBinding] = _binding.flatMap {
    case bindingVal: TopBinding => Some(bindingVal)
  }

  private[chisel3] def ref: Arg = {
    requireIsHardware(this)
    topBindingOpt match {
      case Some(binding: TopBinding) => Node(this)
      case opt => throwException(s"internal error: unknown binding $opt in generating LHS ref")
    }
  }
}
```

目前 `def topBindingOpt` 返回的都是 `TopBinding` 类型，因为当前只定义了这么一种，所以 `.ref` 调用 `def ref` 方法的都会返回 `Node` 节点类型。 

而 `abstract class Definition` 里 `def name: String = id.getRef.name` 的 `getRef` 方法用到的 `_ref` 跟 `ref` 方法不一样。

- `ref` 主要是作为命令的操作数，比如 `DefPrim(dest, op, this.ref)`；
- `_ref` 配合 `getRef`、`setRef` 方法来读取或改变自身的值，用于绑定自身的名字，比如 `id.getRef.name`。

因为这两个名字太相似，容易混淆。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] trait HasId {
  var _ref: Option[Arg] = None
  private[chisel3] def setRef(imm: Arg): Unit = {
    _ref = Some(imm)
  }
  private[chisel3] def getRef: Arg = _ref.get
  private[chisel3] def getOptionRef: Option[Arg] = _ref
}
```

`Arg` 参数类型目前只有：`Node` 节点 和 `Ref`引用。

```scala
// play-chisel/src/internal/firrtl/IR.scala
abstract class Arg {
  def fullName(ctx: Component): String = name
  def name: String
}

case class Node(id: HasId) extends Arg {
  override def fullName(ctx: Component): String = id.getOptionRef match {
    case Some(arg) => arg.fullName(ctx)
    case None => id.suggestedName.getOrElse("??")
  }
  def name: String = id.getOptionRef match {
    case Some(arg) => arg.name
    case None => id.suggestedName.getOrElse("??")
  }
}

case class Ref(name: String) extends Arg
```

用户建议的名字 `suggestedName`，只能被设置一次。之后的设置都会被忽略。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] trait HasId {
  private var suggested_name: Option[String] = None

  def suggestName(name: =>String): this.type = {
    if(suggested_name.isEmpty) suggested_name = Some(name)
    this
  }
  private[chisel3] def suggestedName: Option[String] = suggested_name
}
```

添加命令 `pushOp` 的方法先把运算的结果绑定为 `OpBinding` 类型，并添加到当前模块存放命令列表 `_commands` 数据结构里。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
object Builder {
  ...
  def forcedUserModule: RawModule = currentModule match {
    case Some(module: RawModule) => module
    case _ => throwException(
      "Error: Not in a UserModule. Likely cause: Missed Module() wrap, bare chisel API call, or attempting to construct hardware inside a BlackBox."
    )
  }

  def pushCommand[T <: Command](c: T): T = {
    forcedUserModule.addCommand(c)
    c
  }

  def pushOp[T <: Data](cmd: DefPrim[T]): T = {
    // Bind each element of the returned Data to being a Op
    cmd.id.bind(OpBinding(forcedUserModule))
    pushCommand(cmd).id
  }
}
```

抛出异常定义在 `Error.scala` 里，专门用于 **ChiselException** 相关的异常。

```scala
// play-chisel/chiselFrontend/src/internal/Error.scala
private[chisel3] object throwException {
  def apply(s: String, t: Throwable = null): Nothing =
    throw new ChiselException(s, t)
}
```

`OpBinding` 运算结果的绑定类型是 `TopBinding` 的子类。

```scala
// play-chisel/chiselFrontend/src/internal/Binding.scala
sealed trait ReadOnlyBinding extends TopBinding
case class OpBinding(enclosure: RawModule) extends ConstrainedBinding with ReadOnlyBinding
```

`RawModule` 模块的命令列表 `_commands`， 添加命令的方法 `addCommand` 和获取命令的方法 `getCommands`。

```scala
// play-chisel/chiselFrontend/src/RawModule.scala
import scala.collection.mutable.ArrayBuffer

import chisel3.internal._
import chisel3.internal.firrtl._

abstract class RawModule extends BaseModule {
  val _commands = ArrayBuffer[Command]()
  def addCommand(c: Command) {
    require(!_closed, "Can't write to module after module close")
    _commands += c
  }
  def getCommands = {
    require(_closed, "Can't get commands before module close")
    _commands.toSeq
  }
  ...
}
```

添加命令只能在例化模块的过程中调用，获取命令只能等到模块例化完毕后，在 `generateComponent` 方法里调用。

### Connect 命令

`Connect` 连接命令对应的是 **Verilog** 的赋值语句。

```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  private[chisel3] def connect(that: Data): Unit = {
    requireIsHardware(this, "data to be connected")
    requireIsHardware(that, "data to be connected")
    this.topBinding match {
      case _: ReadOnlyBinding => throwException(s"Cannot reassign to read-only $this")
      case _ =>  // fine
    }
    try {
      MonoConnect.connect(this, that, Builder.referenceUserModule)
    } catch {
      case MonoConnectException(message) =>
        throwException(
          s"Connection between sink ($this) and source ($that) failed @$message"
        )
    }
  }
}
```

`def connect` 方法要求连接的两边都是可综合的硬件类型。在目前这个二选一的多路选择器里，涉及到连接的只有把位运算结果赋值给输出端口，位运算结果是 `OpBinding`，端口都是 `PortBinding`，都是硬件类型，符合条件。

`connect` 方法的左侧（也就是调用该方法的对象）不能是 `ReadOnlyBinding` 的类型，比如这里的位运算结果，只能放在右侧。

```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  private[chisel3] def topBinding: TopBinding = topBindingOpt.get
}
```

`def topBinding` 只是获取了 `topBindingOpt` 的 `Option` 值。

`MonoConnect.connected` 方法的参数除了左/右值，还需要 `Builder` 的当前模块 `Builder.referenceUserModule`。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
object Builder {
  def referenceUserModule: RawModule = {
    currentModule match {
      case Some(module: RawModule) => module
      case _ => throwException(
        "Error: Not in a RawModule. Likely cause: Missed Module() wrap, bare chisel API call, or attempting to construct hardware inside a BlackBox."
      )
    }
  }
}
```

`referenceUserModule` 确保当前的模块类型是 `RawModule` 或是其子类。

```shell
$ touch chiselFrontend/src/internal/MonoConnect.scala
```

```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
package chisel3.internal

private[chisel3] object MonoConnect {
  def UnwritableSinkException =
    MonoConnectException(": Sink is unwriteable by current module.")
  def UnknownRelationException =
    MonoConnectException(": Sink or source unavailable to current module.")
  def MismatchedException(sink: String, source: String) =
    MonoConnectException(s": Sink ($sink) and Source ($source) have different types.")
}
```

`Sink` 直译是水槽， `Source` 是水源。按照水源流向水槽来理解，`Sink` 相当于赋值语句的左值，`Source` 相当于赋值语句的右值。

这里定义了三种错误类型：左值不可被赋值 `UnwritableSinkException`、当前模块的左/右值不可用 `UnknownRelationException` 和左/右值的类型不同 `MismatchedException` 。

```scala
// play-chisel/chiselFrontend/src/package.scala
package object chisel3 {
  ...
  case class MonoConnectException(message: String) extends ChiselException(message)
}
```

`MonoConnectException` 继承了 `ChiselException`，作为 **Chisel** 异常的一种。

```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
import chisel3._

private[chisel3] object MonoConnect {
  ...
  def connect(
    sink: Data,
    source: Data,
    context_mod: RawModule): Unit =
    (sink, source) match {
      case (sink_e: UInt, source_e: UInt) =>
        elemConnect(sink_e, source_e, context_mod)
      case (sink, source) => throw MismatchedException(sink.toString, source.toString)
    }
}
```

`def connect` 方法这里只检查了左/右值的类型是否相同，然后调用 `elemConnect` 方法。

```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
private[chisel3] object MonoConnect {
  ...
  def elemConnect(sink: Element, source: Element, context_mod: RawModule): Unit = {
    import BindingDirection.{Internal, Input, Output}
    val sink_mod: BaseModule   = sink.topBinding.location.getOrElse(throw UnwritableSinkException)
    val source_mod: BaseModule = source.topBinding.location.getOrElse(context_mod)

    val sink_direction = BindingDirection.from(sink.topBinding, sink.direction)
    val source_direction = BindingDirection.from(source.topBinding, source.direction)
  }
}
```

如果找不到 `sink` 所在的模块，就无法赋值给左值，抛出 `UnwritableSinkException` 异常。找不到 `source` 所在的模块则认为是在当前模块下，比如字面量 `literal`，这里先不细究。

`BindingDirection.from` 方法是把数据最终解析后的实际方向转换成绑定方向，规则如下：

```scala
// play-chisel/chiselFrontend/src/internal/Binding.scala
private[chisel3] sealed abstract class BindingDirection
private[chisel3] object BindingDirection {
  case object Internal extends BindingDirection
  case object Output extends BindingDirection
  case object Input extends BindingDirection

  def from(binding: TopBinding, direction: ActualDirection): BindingDirection = {
    binding match {
      case PortBinding(_) => direction match {
        case ActualDirection.Output => Output
        case ActualDirection.Input => Input
        case dir => throw new RuntimeException(s"Unexpected port element direction '$dir'")
      }
      case _ => Internal
    }
  }
}
```

绑定方向只针对于 `Element` 元素，且只用在绑定规则里（也就是这里的连接规则）。

绑定方向有三种：

1. 内部使用 `Internal`：比如 `wire` 或是其他内部类型；
2. 输出类型 `Output`：模块的输出端口；
3. 输入类型 `Input`：模块的输入端口。

`def from` 方法是根据一个 `Element` 类型数据的绑定类型和最终确定的实际方向得到的绑定方向。如果是端口绑定类型，实际方向的输入/输出就对应绑定方向的输入/输出。其他的绑定类型都是内部使用。

目前的二选一多路选择器例子只有一个模块，因此左/右值所在的模块就是当前模块。

```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
private[chisel3] object MonoConnect {
  ...
  def elemConnect(sink: Element, source: Element, context_mod: RawModule): Unit = {
    ...
    // CASE: Context is same module that both left node and right node are in
    if( (context_mod == sink_mod) && (context_mod == source_mod) ) {
      ((sink_direction, source_direction): @unchecked) match {
        //    SINK          SOURCE
        //    CURRENT MOD   CURRENT MOD
        case (Output,       _) => issueConnect(sink, source)
        case (Internal,     _) => issueConnect(sink, source)
        case (Input,        _) => throw UnwritableSinkException
      }
    }
    else throw UnknownRelationException
  }
}
```

输出端口和内部使用的数据都是可以当作左值被赋值，只有输入端口不能被赋值。

当所有连接相关的检查通过之后，调用 `issueConnect` 方法生成连接命令，添加到模块的命令列表里。

```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
import chisel3.internal.Builder.pushCommand
import chisel3.internal.firrtl.Connect

private[chisel3] object MonoConnect {
  ...
  private def issueConnect(sink: Element, source: Element): Unit = {
    source.topBinding match {
      case _ => pushCommand(Connect(sink.lref, source.ref))
    }
  }
}
```

`Connect` 命令存放了左值 `loc: Node` 必须是 `Node` 节点参数类型，右值 `exp: Arg` 是 `Arg` 基本参数类型。

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class Connect(loc: Node, exp: Arg) extends Command
```

`lref` 跟 `ref` 有所不同，`lref` 要求不能是 `ReadOnlyBinding` 只读绑定类型（只读的当然不能被赋值啦）， `ref` 要求不能是字面量绑定类型 `LitBinding` （这里不细究）。

```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  private[chisel3] def lref: Node = {
    requireIsHardware(this)
    topBindingOpt match {
      case Some(binding: ReadOnlyBinding) => throwException(s"internal error: attempted to generate LHS ref to ReadOnlyBinding $binding")
      case Some(binding: TopBinding) => Node(this)
      case opt => throwException(s"internal error: unknown binding $opt in generating LHS ref")
    }
  }
}
```

## 命名空间

命名空间 **Namespace** 内部使用可变的数据结构 `HashMap[String, Long]`。命名用的是字符串类型，作为哈希的键；名字出现的次数是长整型，作为哈希的值。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] class Namespace(keywords: Set[String]) {
  private val names = collection.mutable.HashMap[String, Long]()
  for (keyword <- keywords)
    names(keyword) = 1
  ...
}
```

关键词（`keyword`）是命名空间的保留字，不能再用来命名。因此命名空间初始化的时候需要把关键词加上， 赋值为 `1` 表示关键词出现了一次。 `Set[String]` 集合类型保证了关键词不会重复出现。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] class Namespace(keywords: Set[String]) {
  ...
  def contains(elem: String): Boolean = names.contains(elem)
}
```

内部的数据结构 `private val names` 是私有成员，外部不可访问，因此用一个同名方法 `def contains` 包装一层。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] class Namespace(keywords: Set[String]) {
  ...
  def name(elem: String): String = {
    if (this contains elem) {
      name(rename(elem))
    } else {
      names(elem) = 1
      elem
    }
  }
}
```

传递一个你想命名的名字给 `def name` 方法，1）如果命名空间里不存在，则添加到命名空间并标记为 `1` （出现了 1 次）；2）如果命名空间存在，那么就在该名字后面加上出现的第几次作为后缀。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] class Namespace(keywords: Set[String]) {
  ...
  private def rename(n: String): String = {
    val index = names(n)
    val tryName = s"${n}_${index}"
    names(n) = index + 1
    if (this contains tryName) rename(n) else tryName
  }
}
```

`def rename` 方法是私有的，只有当 `def name` 方法里发现命名空间存在该名字的时候才被调用。先拿到该名字在命名空间出现的次数，作为后缀，然后更新出现的次数。如果加了后缀还是有冲突，那么再继续加后缀。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] object Namespace {
  /** Constructs an empty Namespace */
  def empty: Namespace = new Namespace(Set.empty[String])
}
```

单例对象 `Namespace` 只是用 `def empty` 方法来方便创建一个没有关键字，空的命名空间。跟自己写一个`new Namespace(Set.empty[String])`等价。

## 生成 Verilog 模块

`generateComponent` 方法根据例化用户定义的模块收集到的信息生成 **Verilog** 模块。

```scala
// play-chisel/chiselFrontend/src/RawModule.scala
abstract class RawModule extends BaseModule {
  ...
  def generateComponent(): Component = {
    val names = nameIds(classOf[RawModule])
    ...
  }
}
```

 `val names` 存放的是`nameIds` 先创建的哈希表`HashMap`，存放用户定义的**Chisel** 模块（这里的例子是 `class Mux2`）里的每个 **Chisel** 元素（ `HasId` 的子类型）及其对应的名字。
 

```scala
// play-chisel/chiselFrontend/src/Module.scala
import scala.collection.mutable.{ArrayBuffer, HashMap}

abstract class BaseModule extends HasId {
  ...
  protected def nameIds(rootClass: Class[_]): HashMap[HasId, String] = {
    val names = new HashMap[HasId, String]()
    def name(node: HasId, name: String) {
      if (!names.contains(node)) {
        names.put(node, name)
      }
    }
    for (m <- getPublicFields(rootClass)) {
      Builder.nameRecursively(m.getName, m.invoke(this), name)
    }
    names
  }
}
```

`def nameIds` 方法是定义在 `BaseModule` 里的。

`val names` 是方法内部的数据结构，`def name` 方法往里添加 **Chisel** 元素和对应名字的键值对，确保不会重复。

 `for (m <- getPublicFields(rootClass)) {...}` 循环是迭代用户继承 `rootClass` （这里是 `RawModule`）自定义模块（这里是 `Mux2`）里的每一个公共数据值 `val`，传给 `Builder.nameRecursively` 方法筛选出哪些 `val` 是定义了 **Chisel** 元素，然后给它们命名。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] trait HasId {
  ...
  private[chisel3] def getPublicFields(rootClass: Class[_]): Seq[java.lang.reflect.Method] = {
    // Suggest names to nodes using runtime reflection
    def getValNames(c: Class[_]): Set[String] = {
      if (c == rootClass) {
        Set()
      } else {
        getValNames(c.getSuperclass) ++ c.getDeclaredFields.map(_.getName)
      }
    }
    val valNames = getValNames(this.getClass)
    def isPublicVal(m: java.lang.reflect.Method) =
      m.getParameterTypes.isEmpty && valNames.contains(m.getName) && !m.getDeclaringClass.isAssignableFrom(rootClass)

    this.getClass.getMethods.sortWith(_.getName < _.getName).filter(isPublicVal(_))
  }
}
```

`getValNames(this.getClass)` 是从当前类遍历到根类 `rootClass` 之前，不包含根类（这里是 `RawModule` ）的所有声明过的变量名字，放在 `valNames` 里。

`def isPublicVal` 筛选出公共变量的方法是根据三个条件：

1. `getParameterTypes.isEmpty` 没有参数的，排除掉带参数的方法；
2. `valNames.contains` 名字是限定在刚才获取的 `valNames` 范围里的；
3. `!m.getDeclaringClass.isAssignableFrom` 是不可以被赋值的，排除 `var`。

`this.getClass.getMethods` 是获得该类以及它继承的所有父类定义过的方法（`val` 和 `var`包括在内），然后通过 `isPublicVal` 方法筛选出从根类（这里是 `RawModule`）之后的子类定义过的公共 `val` 声明的数据。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
object Builder {
  ...
  def nameRecursively(prefix: String, nameMe: Any, namer: (HasId, String) => Unit): Unit = nameMe match {
    case (id: HasId) => namer(id, prefix)
    case Some(elt) => nameRecursively(prefix, elt, namer)
    case m: Map[_, _] =>
      m foreach { case (k, v) =>
        nameRecursively(s"${k}", v, namer)
      }
    case (iter: Iterable[_]) if iter.hasDefiniteSize =>
      for ((elt, i) <- iter.zipWithIndex) {
        nameRecursively(s"${prefix}_${i}", elt, namer)
      }

    case _ => // Do nothing
  }
}
```
`def nameRecursively` 结合调用它传递的参数来看，分别是 `val` 声明的名字， `val` 执行后的值和往`names`里添加 **Chisel** 及其对应的名字键值对的 `name` 方法。

`case (id: HasId)` 如果是 **Chisel** 类型的数据，比如 `val foo = UInt(3.W)`，`UInt` 是 `HasId` 的子类满足条件，然后执行 `namer(id, prefix)` 把 **Chisel** 元素 `UInt(3.W)`（`id`）和对应的名字 `foo`（`prefix`）键值对存到之前声明的 `names` 数据结构里。

`case Some(elt)` 是对应 `val foo = Some(UInt(3.W))` 这种情况。

`case m: Map[_,_]` 是咧威新增的，支持 `val fooMap = Map("foo" -> UInt(3.W))` 这种情况。

`case  (iter: Iterable[_]) if iter.hasDefiniteSize` 支持的是 `val fooList = List(UInt(3.W))` 这种情况。

其他情况则忽略。

```scala
// play-chisel/chiselFrontend/src/RawModule.scala
import scala.collection.mutable.{ArrayBuffer, HashMap}

abstract class RawModule extends BaseModule {
  ...
  private[chisel3] def namePorts(names: HashMap[HasId, String]): Unit = {
    for (port <- getModulePorts) {
      port.suggestedName.orElse(names.get(port)) match {
        case Some(name) =>
          if (_namespace.contains(name)) {
            throwException(s"""Unable to name port $port to "$name" in $this,""" +
              " name is already taken by another port!")
          }
          port.setRef(ModuleIO(this, _namespace.name(name)))
        case None => throwException(s"Unable to name port $port in $this, " +
          "try making it a public field of the Module")
      }
    }
  }

  def generateComponent(): Component = {
    ...
    namePorts(names)
  }
}
```

`port.suggestedName.orElse(names.get(port))` 如果用户定义了名字（`suggestedName`）则使用自定义的，否则才去用代码里写的变量名。

`namePorts` 方法给模块的端口绑定名字 `port.setRef(ModuleIO(this, _namespace.name(name)))`。调用命名空间的 `name` 方法是确保不会起重复的名字。

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class ModuleIO(mod: BaseModule, name: String) extends Arg
```

`ModuleIO` 存放的信息是端口所在的模块和它的名字。

```scala
// play-chisel/chiselFrontend/src/Module.scala
abstract class BaseModule extends HasId {
  ...
  private[chisel3] val _namespace = Namespace.empty
}
```

这里的 `_namespace` 模块自己的命名空间。

```scala
// play-chisel/chiselFrontend/src/RawModule.scala
abstract class RawModule extends BaseModule {
  ...
  def generateComponent(): Component = {
    ...

    for ((node, name) <- names) {
      node.suggestName(name)
    }
  }
}
```

`suggestName` 只绑定一次，多次调用会忽略。 这里是确保所有 **Chisel** 元素都有自定义的名字，如果用户没有定义则用代码里声明的名字，定义了不会被覆盖。

```scala
// play-chisel/chiselFrontend/src/RawModule.scala
abstract class RawModule extends BaseModule {
  ...
  def generateComponent(): Component = {
    ...
    // All suggestions are in, force names to every node.
    for (id <- getIds) {
      id match {
        case id: BaseModule => id.forceName(default=id.desiredName, _namespace)
        case id: Data  =>
          if (id.isSynthesizable) {
            id.topBinding match {
              case OpBinding(_) | PortBinding(_) =>
                id.forceName(default="_T", _namespace)
              case _ =>  // don't name literals
            }
          } // else, don't name unbound types
      }
    }
  }
}
```

`getIds` 是获取当前模块里所有的 **Chisel** 元素。

`case id: BaseModule` 如果是 **Verilog** 模块类型的，调用自身的 `forceName` 方法起 `desiredName` 定义的名字，确保在 `_namespace`命名空间里不重名。

`case id: Data` 是给可综合的硬件类型起名字，这里只有位运算的结果 `OpBinding` 和端口 `PortBinding` 两种。

```scala
// play-chisel/chiselFrontend/src/Module.scala
abstract class BaseModule extends HasId {
  ...
  private val _ids = ArrayBuffer[HasId]()
  private[chisel3] def addId(d: HasId) {
    require(!_closed, "Can't write to module after module close")
    _ids += d
  }
  protected def getIds = {
    require(_closed, "Can't get ids before module close")
    _ids.toSeq
  }
}
```

**Verilog** 模块里定义的 **Chisel** 元素是收集在 `_ids` 列表里的。只有在模块没有关闭之前可以添加 `addId`，在模块关闭后（也就是调用 `generateComponent` 方法时）获取 `getIds`。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
private[chisel3] trait HasId {
  private[chisel3] val _parent: Option[BaseModule] = Builder.currentModule
  _parent.foreach(_.addId(this))
  ...
}
```

每个 **Chisel** 元素在例化的时候都会自动添加到当前的模块。

**注意**： **Mux2** 模块也是 **HasId**，也会自动添加到当前模块。但由于它是最顶层的 **HasId**，此时的 `Builder.currentModule` 还是初始值 `None`。 `_parent.foreach` 就派上用场了。 `foreach` 这里不是迭代的意思，而是当遇到 `None` 的时候不执行。因此 **Mux2** 顶层模块不会被添加。

```scala
private[chisel3] trait HasId {
  ...
  private[chisel3] def forceName(default: =>String, namespace: Namespace): Unit =
    if(_ref.isEmpty) {
      val candidate_name = suggested_name.getOrElse(default)
      val available_name = namespace.name(candidate_name)
      setRef(Ref(available_name))
    }
}
```

`forceName` 会先看 `suggested_name` 有没有定义，没有定义则用默认值 `default`，同时调用命名空间的 `name` 方法确保不重名。

`setRef` 方法只是为了起名字。

**注意**：回想下前面的：

```scala
  for (id <- getIds) {
      id match {
        ...
        case id: Data  =>
              ...
              case OpBinding(_) | PortBinding(_) =>
                id.forceName(default="_T", _namespace)

```

以现在的二选一多路选择器为例 `io.out := (io.sel & io.in1) | (~io.sel & io.in0)` 这里的位运算`(io.sel & io.in1)`会产生一个 **Chisel** `Data` 存放结果，但是它又不是用 `val` 独立声明的，所以 `suggestedName` 是没有定义的，会采用默认值 `default="_T"`命名。当出现多个这种中间变量时，根据命名空间重名加出现次数作为后缀的解决办法，就会生成 `_T_1`、 `_T_2`这种名字。

```scala
// play-chisel/chiselFrontend/src/Module.scala
abstract class BaseModule extends HasId {
  ...
  /** Desired name of this module. Override this to give this module a custom, perhaps parametric,
    * name.
    */
  def desiredName: String = this.getClass.getName.split('.').last

  final lazy val name = desiredName
}
```

`desiredName` 默认是声明的模块类的名字，比如 `class Mux2` 的名字是 `Mux2`。如果想改变模块的名字为 `Mux2Foo`，重载 `desiredName` 方法：

```scala
class Mux2 extends RawModule {
  override desiredName = "Mux2Foo"
  ...
}
```

```scala
// play-chisel/chiselFrontend/src/RawModule.scala
abstract class RawModule extends BaseModule {
  ...
  def generateComponent(): Component = {
    ...
    val firrtlPorts = getModulePorts map { port: Data =>
      val direction = port.specifiedDirection
      Port(port, direction)
    }

    DefModule(this, name, firrtlPorts, getCommands)
  }
}

```

`getModulePorts` 获取当前模块的端口列表，转成端口 **IR** `Port(port, direction)`。

**Verilog** 模块 **IR** 的信息包括该模块自身 `this`、名字 `name`、端口 **IR** 列表和命令列表（命令对应 **Verilog** 的语句）。

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class Port(id: Data, dir: SpecifiedDirection)

abstract class Component extends Arg {
  def id: BaseModule
  def name: String
  def ports: Seq[Port]
}

case class DefModule(id: RawModule, name: String, ports: Seq[Port], commands: Seq[Command]) extends Component
```

端口 **IR** 包括端口数据和方向。

`Component` 是所有 **Verilog** 模块类型的基类，包括模块自身，名字和端口。 

`DefModule` 比基类多了命令列表。还有其他的模块类型，比如 **BlackBox** 也是继承的 `Component` 基类，这里不细究。

## 生成电路 IR

`object Module` 是负责生成 **Verilog** 模块 **IR**，并添加到 **Builder** 的全局 **Verilog** 模块列表里。

`Builder.build` 是负责把 **Verilog** 模块列表生成整个电路 **IR**。

```scala
// play-chisel/chiselFrontend/src/Module.scala
object Module {
  def apply[T <: BaseModule](bc: => T): T = {
    val module: T = bc
    val component = module.generateComponent()
    Builder.components += component
    module
  }
  ...
}
```

`Module.apply` 调用参数传进来的模块 `generateComponent` 方法生成 **Verilog** 模块 **IR**，添加到 `Builder.components` 的模块列表里。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
import scala.collection.mutable.ArrayBuffer

object Builder {
  val components: ArrayBuffer[Component] = ArrayBuffer[Component]()
  def globalNamespace: Namespace = Namespace.empty

  ...
  def build[T <: RawModule](f: => T): (Circuit, T) = {
    println("Elaborating design...")
    val mod = f
    mod.forceName(mod.name, globalNamespace)
    println("Done elaborating.")

    (Circuit(components.last.name, components), mod)
  }
}
```

`Builder.build` 拿到的是经过了 **Module()** 后最顶层的模块。调用`forceName` 起名字，确保在 `Builder` 的全局命名空间里不重名。

电路 **IR** 的名字是最顶层（最后一个添加的，所以是列表的最后一个）模块的名字，它存放的信息是所有模块的 **IR**。

# 解析 IR 生成 FIRRTL

`Emitter` 负责把电路 **IR** 生成 **FIRRTL** 字符串的形式。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  private val res = new StringBuilder()
  override def toString: String = res.toString
}
```

`val res` 可变的字符串构造器 `StringBuilder`。`def toString` 方法是生成最终的字符串。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  ...
  private var indentLevel = 0
  private def newline = "\n" + ("  " * indentLevel)
  private def indent(): Unit = indentLevel += 1
  private def unindent() { require(indentLevel > 0); indentLevel -= 1 }
  private def withIndent(f: => Unit) { indent(); f; unindent() }
}
```

这里先定义一些后续用到的变量和方法：

- `indentLevel` 缩进层次记录当前的缩进是多少个空格，初始值为 0；
- `def newline` 方法除了换行还会加上缩进层次的空格；
- `indent` 和 `unindent` 分别是缩进层次加 1 和减 1；
- `withIndent` 是对应的一套缩进层次，方便构建一个作用域。

都是方便创建缩进层次。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  ...
  res ++= s"circuit ${circuit.name} : "
  withIndent { circuit.components.foreach(c => res ++= emit(c)) }
  res ++= newline
}
```

先生成电路的名字，然后创建一级缩进的作用域，生成每一个模块，最后空一行结束。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  ...
  private def emit(m: Component): String = {
    val sb = new StringBuilder
    sb.append(moduleDecl(m))
    sb.append(moduleDefn(m))
    sb.result
  }
}
```

生成模块的 `emit(m: Component)`方法先生成模块的声明，然后才生成模块的定义。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  ...
  private def moduleDecl(m: Component): String = m.id match {
    case _: RawModule => newline + s"module ${m.name} : "
  }
}
```

`def moduleDecl(m: Component)` 方法生成 **FIRRTL** 模块的声明，`module 模块名字`。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  ...
  private def moduleDefn(m: Component): String = {
    val body = new StringBuilder
    withIndent {
      for (p <- m.ports) {
        val portDef = m match {
          case mod: DefModule => emitPort(p)
        }
        body ++= newline + portDef
      }
      body ++= newline

      m match {
          case mod: DefModule => {
          val procMod = mod
          for (cmd <- procMod.commands) { body ++= newline + emit(cmd, procMod)}
        }
      }
      body ++= newline
    }
    body.toString()
  }
}
```

`def moduleDefn` 生成模块定义的方法，先生成一级缩进，作为模块内部作用域的缩进；再生成端口列表；最后生成命令列表。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  ...
  private def emitPort(e: Port, topDir: SpecifiedDirection=SpecifiedDirection.Unspecified): String = {
    val resolvedDir = SpecifiedDirection.fromParent(topDir, e.dir)
    val dirString = resolvedDir match {
      case SpecifiedDirection.Unspecified | SpecifiedDirection.Output => "output"
      case SpecifiedDirection.Flip | SpecifiedDirection.Input => "input"
    }
    val clearDir = resolvedDir match {
      case SpecifiedDirection.Input | SpecifiedDirection.Output => true
      case SpecifiedDirection.Unspecified | SpecifiedDirection.Flip => false
    }
    s"$dirString ${e.id.getRef.name} : ${emitType(e.id, clearDir)}"
  }
}
```

`def emitPort` 生成端口的方法先确定端口的方向，然后通过 `id.getRef.name` 拿到端口的名字，最后调用 `emitType` 得到端口的类型。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  ...
  private def emitType(d: Data, clearDir: Boolean = false): String = d match {
    case d: UInt => s"UInt${d.width}"
  }
}
```

`def emitType` 生成类型的方法，目前只有 `UInt` 类型。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  ...
  private def emit(e: Command, ctx: Component): String = { // scalastyle:ignore cyclomatic.complexity
    val firrtlLine = e match {
      case e: DefPrim[_] => s"node ${e.name} = ${e.op.name}(${e.args.map(_.fullName(ctx)).mkString(", ")})"
      case e: Connect => s"${e.loc.fullName(ctx)} <= ${e.exp.fullName(ctx)}"
    }
    firrtlLine
  }
}
```

`def emit(e: Command, ctx: Component)` 方法生成 **Verilog** 语句。目前只有两种类型：

- `DefPrim` 运算语句：作为 **FIRRTL** 的一个节点 `node 名字 = 运算类型的名字 参数列表`
- `Connect` 赋值语句： `左值名字 <= 右值表达式名字`

{{% admonition tip 源码03 %}}
{{% /admonition %}}

[play-chisel/tree/chap01-03](https://github.com/colin4124/play-chisel/tree/chap01-03)

# 生成 Verilog

```shell
$ mill chisel3.run
```

[下载](https://github.com/colin4124/chisel3-releases/releases/download/firrtl-1.2.0/firrtl-1.2.0.jar) **FIRRTL** 到项目的根目录：

```shell
$ wget https://github.com/colin4124/chisel3-releases/releases/download/firrtl-1.2.0/firrtl-1.2.0.jar
$ java -cp firrtl-1.2.0.jar firrtl.stage.FirrtlMain -i Mux2.fir -o Mux.v
```
