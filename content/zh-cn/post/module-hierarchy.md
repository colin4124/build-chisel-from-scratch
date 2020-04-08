---
title: "第三章：模块的层次结构"
date: 2020-02-22T00:12:35+08:00
draft: true
---

本章定义一个四选一的多路选择器 `Mux4` 模块，包含了三个之前定义的 `Mux2` 模块作为子模块。这是最简单的二级模块层次结构。

# 目标

在 `Main.scala` 新增 `Mux4`

```scala
...
class Mux4 extends RawModule {
  val in0 = IO(Input(UInt(1.W)))
  val in1 = IO(Input(UInt(1.W)))
  val in2 = IO(Input(UInt(1.W)))
  val in3 = IO(Input(UInt(1.W)))
  val sel = IO(Input(UInt(2.W)))
  val out = IO(Output(UInt(1.W)))

  val m0 = Module(new Mux2)
  m0.sel := sel(0)
  m0.in0 := in0
  m0.in1 := in1
  val m1 = Module(new Mux2)
  m1.sel := sel(0)
  m1.in0 := in2
  m1.in1 := in3
  val m2 = Module(new Mux2)
  m2.sel := sel(1)
  m2.in0 := m0.out
  m2.in1 := m1.out

  out := m3.out
}

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Mux4))

  val emitted = Emitter.emit(circuit)

  val file = new File("Mux4.fir")
  val w = new FileWriter(file)
  w.write(emitted)
  w.close()
}
```

```firrtl
circuit Mux4 : 
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
    
  module Mux2_1 : 
    input sel : UInt<1>
    input in0 : UInt<1>
    input in1 : UInt<1>
    output out : UInt<1>
    
    node _T = and(sel, in1)
    node _T_1 = not(sel)
    node _T_2 = and(_T_1, in0)
    node _T_3 = or(_T, _T_2)
    out <= _T_3
    
  module Mux2_2 : 
    input sel : UInt<1>
    input in0 : UInt<1>
    input in1 : UInt<1>
    output out : UInt<1>
    
    node _T = and(sel, in1)
    node _T_1 = not(sel)
    node _T_2 = and(_T_1, in0)
    node _T_3 = or(_T, _T_2)
    out <= _T_3
    
  module Mux4 : 
    input in0 : UInt<1>
    input in1 : UInt<1>
    input in2 : UInt<1>
    input in3 : UInt<1>
    input sel : UInt<2>
    output out : UInt<1>
    
    inst m0 of Mux2
    node _T = bits(sel, 0, 0)
    m0.sel <= _T
    m0.in0 <= in0
    m0.in1 <= in1
    inst m1 of Mux2_1
    node _T_1 = bits(sel, 0, 0)
    m1.sel <= _T_1
    m1.in0 <= in2
    m1.in1 <= in3
    inst m2 of Mux2_2
    node _T_2 = bits(sel, 1, 1)
    m2.sel <= _T_2
    m2.in0 <= m0.out
    m2.in1 <= m1.out
    out <= m2.out
```

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
module Mux4(
  input        in0,
  input        in1,
  input        in2,
  input        in3,
  input  [1:0] sel,
  output       out
);
  wire  m0_sel;
  wire  m0_in0;
  wire  m0_in1;
  wire  m0_out;
  wire  m1_sel;
  wire  m1_in0;
  wire  m1_in1;
  wire  m1_out;
  wire  m3_sel;
  wire  m3_in0;
  wire  m3_in1;
  wire  m3_out;
  Mux2 m0 (
    .sel(m0_sel),
    .in0(m0_in0),
    .in1(m0_in1),
    .out(m0_out)
  );
  Mux2 m1 (
    .sel(m1_sel),
    .in0(m1_in0),
    .in1(m1_in1),
    .out(m1_out)
  );
  Mux2 m3 (
    .sel(m3_sel),
    .in0(m3_in0),
    .in1(m3_in1),
    .out(m3_out)
  );
  assign out = m3_out;
  assign m0_sel = sel[0];
  assign m0_in0 = in0;
  assign m0_in1 = in1;
  assign m1_sel = sel[0];
  assign m1_in0 = in2;
  assign m1_in1 = in3;
  assign m3_sel = sel[1];
  assign m3_in0 = m0_out;
  assign m3_in1 = m1_out;
endmodule
```

# 实现

## 构造模块前保存它的父模块

`Module()` 在构建一个 **Verilog** 模块的时候，先保存之前 `Builder` 的 `Builder.currentModule`。因为当例化用户定义的模块时，`Builder.currentModule` 会被覆盖成当前正在例化的用户自定义模块（比如这里的 `Mux2` 和 `Mux4`都是用户定义的模块）。

```scala
// play-chisel/chiselFrontend/src/Module.scala
import chisel3.internal.Builder._

object Module {
  def apply[T <: BaseModule](bc: => T): T = {
    val parent = Builder.currentModule
    val module: T = bc
    Builder.currentModule = parent // Back to parent!
    val component = module.generateComponent()
    Builder.components += component

    if(!Builder.currentModule.isEmpty) {
      pushCommand(DefInstance(module, component.ports))
    }

    module
  }
}
```

在例化用户定义的模块 `val module: T = bc` 的前后，分别是保存之前的 `Builder.currentModule` 到 `parent` 和恢复之前的 `Builder.currentModule` 。

当 `Builder` 的当前模块不为空的时候（`!Builder.currentModule.isEmpty`），会生成 **Verilog** 实例的 **IR** `DefInstance`。换句话说，只用构建顶层模块的时候，`Builder.currentModule` 是空的，除此之外调用 `Module()` 构建模块的情形都是顶层模块里面用到了才会例化，因此要多生成一条实例化的语句。

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class DefInstance(id: BaseModule, ports: Seq[Port]) extends Definition
```

**Verilog** 实例的 **IR** 包含了 **Verilog** 模块和端口列表。

## 提取一个比特位操作

`Mux4` 的选择信号 `sel` 宽度是两位的，可以用 `sel(0)` 和 `sel(1)` 来获取某个比特。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed abstract class Bits(private[chisel3] val width: Width) extends Element {
  ...
  final def apply(x: BigInt): Bool =  {
    if (x < 0) {
      throwException(s"Negative bit indices are illegal (got $x)")
    }
    requireIsHardware(this, "bits to be indexed")
    pushOp(DefPrim(Bool(), BitsExtractOp, this.ref, ILit(x), ILit(x)))
  }

  final def apply(x: Int): Bool = apply(BigInt(x))
}
```

`sel()` 是 `sel.apply()` 的简写，它的类型是 `UInt`，位提取的操作不仅仅限于 `UInt`，凡是基于 `UInt` 的父类 `Bits`的数据类型都支持。

`def apply(x: BigInt)` 接收的类型是 `BigInt`，如果是 `Int` 是则调用它的多态方法（名字相同但参数类型不同）`def apply(x: Int)` 来转成 `BigInt` ：`apply(BigInt(x))`。

位提取操作的索引不能小于 `0`，也要求继承了 `Bits` 基类的数据类型已经被绑定成硬件类型。

最后往当前模块添加一条位操作的命令（语句）。位操作提取的结果类型是 `Bool()`，位操作的名称是 `BitsExtractOp`，参数列表是被提前的对象，也就是自身 `this.ref`，提取的范围（最高有效位和最低有效位），因为只有一个比特，所以最高、最低有效位相同。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val BitsExtractOp = PrimOp("bits")
}

case class ILit(n: BigInt) extends Arg {
  def name: String = n.toString
}
```

比特提取操作对应到 **FIRRTL** 的名称是 `bits`。

`ILit` 整型字面量（Integer Literals）的参数类型包含的信息就是索引值。

## Bool 类型

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed trait Reset extends Element

sealed class Bool() extends UInt(1.W) with Reset {
  private[chisel3] override def cloneTypeWidth(w: Width): this.type = {
    require(!w.known || w.get == 1)
    new Bool().asInstanceOf[this.type]
  }
}
```

`Bool` 类型是基于 `Reset` 和 `UInt` 类型，并且宽度为 1。其他的 `Reset` 类型目前不细究。

克隆复本的方法要求宽度参数要么是未知的，或者宽度是 1。

```shell
$ touch chiselFrontend/src/BoolFactory.scala
```

```scala
// play-chisel/chiselFrontend/src/BoolFactory.scala
package chisel3

trait BoolFactory {
  def apply(): Bool = new Bool()
}
```

`BoolFactory` 跟 `UIntFactory` 作用类似。

```scala
// play-chisel/chiselFrontend/src/package.scala
package object chisel3 {
  ...
  object Bool extends BoolFactory
}
```

`Bool()` 方便创建一个 `Bool` 值。

```scala
// play-chisel/src/internal/firrtl/IR.scala
sealed abstract class Width {
  def known: Boolean
  def get: W
  ...
}
sealed case class UnknownWidth() extends Width {
  def known: Boolean = false
  def get: Int = None.get
  ...
}
sealed case class KnownWidth(value: Int) extends Width {
  require(value >= 0)
  def known: Boolean = true
  def get: Int = value
  ...
}
```

`def known` 的值区分已知宽度和未知宽度。已知宽度的能拿到 `get` 宽度是多少，未知的没有宽度 `None`。

## 更新连接的方法

因为之前的例子是单个模块的， **Verilog** 元素之间的赋值只需要考虑在同一个模块的情况，如果不是在同一个模块，那么会有三种可能：

1. 左值属于当前模块，右值属于实例化的子模块；
2. 左值属于子模块，右值属于当前模块；
3. 左、右值均属于子模块。

下面分别就这三种可能进行判断。

```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
private[chisel3] object MonoConnect {
  def elemConnect(sink: Element, source: Element, context_mod: RawModule): Unit = {
    ...
    // CASE: Context is same module as sink node and right node is in a child module
    else if( (sink_mod == context_mod) &&
             (source_mod._parent.map(_ == context_mod).getOrElse(false)) ) {
      // Thus, right node better be a port node and thus have a direction
      ((sink_direction, source_direction): @unchecked) match {
        //    SINK        SOURCE
        //    CURRENT MOD CHILD MOD
        case (Internal,   Output)   => issueConnect(sink, source)
        case (Internal,   Input)    => issueConnect(sink, source)
        case (Output,     Output)   => issueConnect(sink, source)
        case (Output,     Input)    => issueConnect(sink, source)
        case (Input,      _)        => throw UnwritableSinkException
      }
    }
  }
}
```

当前模块可以被赋值的类型有内部使用的值和输出端口，它们可以被子模块的输出端口赋值，也可以被子模块的输入端口赋值（实际上是赋值给子模块的输入端口的值做个转发）。


```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
private[chisel3] object MonoConnect {
  def elemConnect(sink: Element, source: Element, context_mod: RawModule): Unit = {
    ...
    // CASE: Context is same module as source node and sink node is in child module
    else if( (source_mod == context_mod) &&
             (sink_mod._parent.map(_ == context_mod).getOrElse(false)) ) {
      // Thus, left node better be a port node and thus have a direction
      ((sink_direction, source_direction): @unchecked) match {
        //    SINK          SOURCE
        //    CHILD MOD     CURRENT MOD
        case (Input,        _) => issueConnect(sink, source)
        case (Output,       _) => throw UnwritableSinkException
        case (Internal,     _) => throw UnwritableSinkException
      }
    }
  }
}
```

子模块可以被赋值的类型只有输入端口，可以被当前模块的任意类型赋值。
    
```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
private[chisel3] object MonoConnect {
  def elemConnect(sink: Element, source: Element, context_mod: RawModule): Unit = {
    ...
    
    // CASE: Context is the parent module of both the module containing sink node
    //                                        and the module containing source node
    //   Note: This includes case when sink and source in same module but in parent
    else if( (sink_mod._parent.map(_ == context_mod).getOrElse(false)) &&
             (source_mod._parent.map(_ == context_mod).getOrElse(false))
    ) {
      // Thus both nodes must be ports and have a direction
      ((sink_direction, source_direction): @unchecked) match {
        //    SINK      SOURCE
        //    CHILD MOD CHILD MOD
        case (Input,    _) => issueConnect(sink, source)
        case (Output,   _) => throw UnwritableSinkException
        case (Internal, _) => throw UnwritableSinkException
      }
    }
  }
}
```

## 更新 IR

子模块可以被赋值的类型只有输入模块，可以被另一个子模块的任意类型赋值。

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class ModuleIO(mod: BaseModule, name: String) extends Arg {
  override def fullName(ctx: Component): String =
    if (mod eq ctx.id) name else s"${mod.getRef.name}.$name"
}
```

这里更新代码是，比较端口所属的模块 `mod` 和调用这个端口全名时的当前模块 `ctx.id`。如果是在所属模块里调用的，那么就是端口本身的名字；如果是在所属模块的父模块调用，那么要加上父模块的引用名字。例如：`val m0 = Module(new Mux2)` ，`m0` 是模块 `Mux2`的引用名字，`Mux2` 是模块名字。

## 更新生成 FIRRTL 的方法

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  private def emit(e: Command, ctx: Component): String = {
    val firrtlLine = e match {
      ...
      case e: DefInstance => s"inst ${e.name} of ${e.id.name}"
    }
  }
  ...
}
```

实例化子模块的 **FIRRTL** 语法是关键字 `inst` + 模块实例的名字 `of` + 模块的名字。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
object Builder {
  val globalNamespace: Namespace = Namespace.empty
}
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码01 %}}
{{% /admonition %}}

[play-chisel/tree/chap03-01](https://github.com/colin4124/play-chisel/tree/chap03-01)


# 总结

我们来走一遍 **Chisel** 生成 **FIRRTL** 的流程。

```scala
class Mux2 extends RawModule {
  val sel = IO(Input(UInt(1.W)))
  ... // 省略了其他类似的端口声明
  out := (sel & in1) | (~sel & in0)
}

class Mux4 extends RawModule {
  val in0 = IO(Input(UInt(1.W)))
  ... // 省略了其他类似的端口声明
  val m0 = Module(new Mux2)
  m0.sel := sel(0)
  ... // 省略了其他类似的 m0 子模块端口连接
  val m1 = Module(new Mux2)
  m1.sel := sel(0)
  ... // 省略了其他类似的 m1 子模块端口连接
  val m3 = Module(new Mux2)
  m3.sel := sel(1)
  ... // 省略了其他类似的 m1 子模块端口连接
  out := m3.out
}
  ...
  val (circuit, _) = Builder.build(Module(new Mux4))
```

## Builder.build 方法

首先执行的是 `Builder.build` 的方法。

```scala
object Builder {
  ... // 省略其他的属性和方法的定义
  def build[T <: RawModule](f: => T): (Circuit, T) = {
    val mod = f
    ...
  }
}
```

初始化完成 `Builder` 的属性（执行完`val xxx` 这些定义）之后才执行它的参数 `Module(new Mux4)`。

## 顶层模块 Mux4 的 Module() 上半部分

上半部分完成的是 `val module: T = bc`，即执行 `new Mux4` 完的部分。

```scala
// 这是 Module(new Mux4) 调用的 apply 方法
object Module {
  def apply[T <: BaseModule](bc: => T): T = {
    val parent = Builder.currentModule
    val module: T = bc
    ...
  }
}
```

因为是顶层模块，此时的 `Builder.currentModule` 还是 `None`。（`val parent = None`）

`val module: T = bc` 开始执行 `new Mux4`，收集完 **IO** `val  sel = IO(Input(UInt(1.W)))` 等等的端口列表信息。

```scala
abstract class BaseModule extends HasId {
  Builder.currentModule = Some(this)
  ...
}
```

每次例化一个模块，都会把当前模块设置为正在例化的模块，此时 `Builder.currentModule` 就变成了 `Mux4`。

然后遇到了例化 `Mux2` 类型的子模块 `m0`。

## 子模块 m0 的 Module()

`val m0 = Module(new Mux2)` 会再次调用 `Module.apply()` 方法。

```scala
// 这是 val m0 = Module(new Mux2) 调用的 apply 方法
object Module {
  def apply[T <: BaseModule](bc: => T): T = {
    val parent = Builder.currentModule
    val module: T = bc
    Builder.currentModule = parent // Back to parent!
    ...
    val component = module.generateComponent()
    ...
    if(!Builder.currentModule.isEmpty) {
      pushCommand(DefInstance(module, component.ports))
    }
    ...
  }
}
```

会先保存当前的模块 `Mux4` 作为它的父模块，放到变量 `val parent`。`Mux2` 模块例化过程中会改变 `Builder.currentModule` 为自身，例化完完之后，恢复原来 `Builder.currentModule` 为父模块 `Mux4`。

这时 `Builder.currentModule` 是 `Mux4` 不为空，`pushCommand` 是则把 `DefInstance(module, component.ports)` 生成例化子模块的语句添加到当前模块 `Builder.currentModule` 也就是 `Mux4` 的 `_commands` 列表里。 `DefInstance` 里的 `module` 是 `Mux4`，`component.ports` 也是 `Mux4` 的端口列表。

## m0 子模块端口的连接

`Mux4` 里的 `m0` 子模块例化好了之后，接下来是子模块端口的连接。

```scala
m0.sel := sel(0)
m0.in0 := in0
m0.in1 := in1
```

`sel` 是子模块 `m0` 的输入模块，因此可以被当前模块的输入端口 `sel(0)` 赋值。

`m1` 和 `m2` 以此类推。

最后是顶层模块 `Mux4` 的输出端口被子模块 `m3` 的输出端口 `out` 赋值 `out := m3.out`。

## 顶层模块 Mux4 的 Module() 下半部分

```scala
// 这是 Module(new Mux4) 调用的 apply 方法
object Module {
  def apply[T <: BaseModule](bc: => T): T = {
    val parent = Builder.currentModule
    val module: T = bc
    Builder.currentModule = parent // Back to parent!
    val component = module.generateComponent()
    Builder.components += component

    if(!Builder.currentModule.isEmpty) {
      pushCommand(DefInstance(module, component.ports))
      module.initializeInParent
    }

    module
  }
}
```

当 `val module: T = bc` 执行完之后，`Builder.currentModule` 恢复到原来的 `None`。

`module.generateComponent` 生成模块 **IR** 的过程中，会给模块里的每个 `HasId` 绑定名字，比如 `val m0 = Module(new Mux2)` ，`Module()` 返回的是例化好的 `new Mux2` 模块，调用 `setRef` 方法为它的 `_ref` 绑定了名字 `m0`（通过 `getRef`读取）。所以当 `Emitter` 生成 `DefInstance` 对应的 **FIRRTL** 语句时：

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  private def emit(e: Command, ctx: Component): String = {
    val firrtlLine = e match {
      ...
      case e: DefInstance => s"inst ${e.name} of ${e.id.name}"
    }
  }
  ...
}
```

`case e: DefInstance => s"inst ${e.name} of ${e.id.name}"` 这里的 `e` 是命令 **IR** `DefInstance`。来看下 `DefInstance` 的原型：

```scala
abstract class Definition extends Command  {
  def id: HasId
  def name: String = id.getRef.name
}
case class DefInstance(id: BaseModule, ports: Seq[Port]) extends Definition
```

`e.name` 相当于 `e.id.getRef.name`， `e.id` 是模块 `Mux2`， `getRef.name` 就是例化 `Mux2` 时绑定的 `_ref` `m0` 。

`e.id.name` 是 `Mux2` 模块的名字。相关的代码如下：

```scala
// play-chisel/chiselFrontend/src/Module.scala
abstract class BaseModule extends HasId {
  def desiredName: String = this.getClass.getName.split('.').last

  /** Legalized name of this module. */
  final lazy val name = Builder.globalNamespace.name(desiredName)
  ...
}
```

模块默认的名字类的名字 `Mux2`，然后会调用全局命名空间 `Builder.globalNamespace`的 `name` 方法，通过把相同名字出现第几次作为后缀避免名字重名，比如 `m0` 是 `Mux2`，`m1` 是 `Mux2_1`。不过 **FIRRTL** 生成的时候会去掉重复的模块定义，只有 `Module Mux2`。

`m0.sel := sel(0)` 相当于 `Connect(m0.sel.lref, sel(0).ref)`，再进一步 `Connect(Node(m0.sel), Node(sel(0)))`。 `Emitter` 对应生成 **FIRRTL** 的代码：

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  private def emit(e: Command, ctx: Component): String = {
    val firrtlLine = e match {
      ...
      case e: Connect => s"${e.loc.fullName(ctx)} <= ${e.exp.fullName(ctx)}"
    }
  }
  ...
}
```

这里的 `e` 是 `Connect` 命令，看下它的原型：

```scala
case class Connect(loc: Node, exp: Arg) extends Command
```

`e.loc` 是左值 `Node(m0.sel)`，`e.loc.fullName` 相当于 `m0.sel.fullName`。而 `sel` 是 `ModuleIO` 类型，它`fullName` 会判断出 `m0.sel` 是在 `m0` 的父模块，生成的名字会加上 `m0`模块的引用名字 `m0`。

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class ModuleIO(mod: BaseModule, name: String) extends Arg {
  override def fullName(ctx: Component): String =
    if (mod eq ctx.id) name else s"${mod.getRef.name}.$name"
}
```

# 格式化 Chisel IR 输出

```shell
$ touch play-chisel/firrtl/src/PrintIR.scala
```

## 电路 IR

电路 **IR** 包括名字和模块 `Component` 列表。

```scala
case class Circuit(name: String, components: Seq[Component])
```

`print_ir` 先输出电路的名字，然后把每个模块传给 `print_module` 输出模块方法。

```scala
// play-chisel/firrtl/src/PrintIR.scala
package chisel3.debug

import chisel3.internal.firrtl._

object PrintIR {
  def print_ir(c: Circuit) = {
    println(s"Circuit: ${c.name}")
    c.components foreach { m =>
      print_module(m)
    }
  }
}
```

## 模块 IR

模块 **IR** 目前只用 `DefModule` 一种类型，这里输出模块 **IR** 只关心名字，端口列表和命令列表。

```scala
abstract class Component extends Arg {
  def id: BaseModule
  def name: String
  def ports: Seq[Port]
}

case class DefModule(id: RawModule, name: String, ports: Seq[Port], commands: Seq[Command]) extends Component
```

输出模块方法 `print_module` 先判断是否为 `DefModule` 类型，如果是其他不支持的类型会报错。然后分别输出名字、端口列表和命令列表。

`def tab` 方法根据缩进层级参数，得到多个 TAB（2个空格）字符串。

`m_str` 方法得到加上了缩进的模块名字字符串。

```scala
// play-chisel/firrtl/src/PrintIR.scala
package chisel3.debug

...
import chisel3.internal.throwException

object PrintIR {
  ...
  def print_module(m: Component) = {
    m match {
      case DefModule(_, name, ports, cmds) =>
        // 输出模块名字
        val name_str = m_str(name, 1)
        println(name_str)
        // 输出端口列表
        ports foreach { p =>
          println(p_str(p, 2))
        }
        // 输出命令列表
        println(cmds_str(cmds, 2))
      case _ => throwException(s"PrintIR: Unknown $m")
    }
  }

  def tab(level: Int) = (" " * 2) * level

  def m_str(name: String, l: Int) = s"${tab(l)}Module: ${name}"
}
```

## 端口 IR

端口 **IR** 包括端口的数据信息和方向。需要输出端口的名字，方向和类型。

```scala
case class Port(id: Data, dir: SpecifiedDirection)
```

`dir_str` 方法把方向类型变成对应的字符串。

`type_str` 方法根据数据的类型转成对应的字符串。

`w_str` 方法把宽度类型转成对应的字符串。

```scala
// play-chisel/chiselFrontend/src/PrintIR.scala
package chisel3.debug

...
import chisel3._

object PrintIR {
  ...
  def p_str(p: Port, l: Int) = s"${tab(l)}Port: ${p.id.getRef.name} ${dir_str(p.dir)} ${type_str(p.id)}"
  def dir_str(d: SpecifiedDirection) = {
    import SpecifiedDirection._
    d match {
      case Unspecified => "Unspecifed"
      case Output      => "Output"
      case Input       => "Input"
      case Flip        => "Flip"
   }
  }
  def type_str(d: Data) = d match {
    case u: UInt => s"UInt<${w_str(u.width)}>"
    case _       => "UnknownType"
  }
  def w_str(w: Width) = w match {
    case UnknownWidth() => "UnknownWidth"
    case KnownWidth(i)  => s"$i"
  }
}
```

输出信息：

```scala
Circuit: Mux4
  Module: Mux2
    Port: sel Input UInt<1>
    Port: in0 Input UInt<1>
    Port: in1 Input UInt<1>
    Port: out Output UInt<1>
  Module: Mux2_1
    Port: sel Input UInt<1>
    Port: in0 Input UInt<1>
    Port: in1 Input UInt<1>
    Port: out Output UInt<1>
  Module: Mux2_2
    Port: sel Input UInt<1>
    Port: in0 Input UInt<1>
    Port: in1 Input UInt<1>
    Port: out Output UInt<1>
  Module: Mux4
    Port: in0 Input UInt<1>
    Port: in1 Input UInt<1>
    Port: in2 Input UInt<1>
    Port: in3 Input UInt<1>
    Port: sel Input UInt<2>
    Port: out Output UInt<1>
```

## 命令 IR

命令 **IR** 目前的类型有定义类型的语句：

- 运算（Primitive Operation）语句 `DefPrim`
- 例化模块语句 `DefInstance`
- 赋值/连接语句 `Connect`

```scala
abstract class Command
abstract class Definition extends Command  {
  def id: HasId
  def name: String = id.getRef.name
}
case class DefPrim[T <: Data](id: T, op: PrimOp, args: Arg*) extends Definition
case class DefInstance(id: BaseModule, ports: Seq[Port]) extends Definition
case class Connect(loc: Node, exp: Arg) extends Command
```

`def cmds_str` 里用递归的方式把列表的每个命令传给 `def cmd_str` 方法转成对应的字符串。

`def rec` 递归方法有存放转换好的字符串 `acc` 队列和尚未转换的命令列表 `cmds`。当命令列表为空时，则转换结束。

```scala
// play-chisel/chiselFrontend/src/PrintIR.scala
package chisel3.debug

import scala.annotation.tailrec
import scala.collection.immutable.Queue

object PrintIR {
  ...
  def cmds_str(cmds: Seq[Command], l: Int) = {
    @tailrec
    def rec(acc: Queue[String])(cmds: Seq[Command]): Seq[String] = {
      if (cmds.isEmpty) {
        acc
      } else cmd_str(cmds.head, l) match {
        case Some(stmt) =>
          rec(acc :+ stmt)(cmds.tail)
        case None => rec(acc)(cmds)
      }
    }
    rec(Queue.empty)(cmds) mkString "\n"
  }
}
```

`def cmd_str` 方法根据不同的命令 IR 输出不同的字符串。未知的类型会报错。

```scala
// play-chisel/chiselFrontend/src/PrintIR.scala
object PrintIR {
  ...
  def cmd_str(cmd: Command, l: Int): Option[String] = cmd match {
    case e: DefPrim[_] =>
      ...
    case Connect(loc, exp) =>
      ...
    case e @ DefInstance(id, _) =>
      ...
    case _ =>
      throwException(s"Unknown Command: $cmd")
  }
}
```

### 运算语句

输出运算语句字符串包括名字、操作类型、参数列表和常量列表。

```scala
case class DefPrim[T <: Data](id: T, op: PrimOp, args: Arg*) extends Definition
```

常量列表 `consts` 是运算语句里参数的整型立即数 `ILit` 类型。

参数列表是 `args` 是运算语句里参数的非 `ILit` 类型。

`arg_str` 方法把参数类型转换成对应的字符串。

```scala
def cmd_str(cmd: Command, l: Int): Option[String] = cmd match {
  case e: DefPrim[_] =>
    val consts = e.args.collect { case ILit(i) => s"$i" } mkString ", "
    val args = e.args.flatMap {
      case _: ILit => None
      case other => Some(arg_str(other))
    } mkString ", "
    val op   = e.op.name
    Some(s"${tab(l)}DefPrim: ${e.name} $op $args $consts")
  ...
}
```

### 连接语句

输出连接语句字符串包括连接右手边和左手边的参数。

```scala
case class Connect(loc: Node, exp: Arg) extends Command
```

`arg_str` 方法把参数类型转换成对应的字符串。

```scala
def cmd_str(cmd: Command, l: Int): Option[String] = cmd match {
  case Connect(loc, exp) =>
    val lhs = arg_str(loc)
    val rhs = arg_str(exp)
    Some(s"${tab(l)}Connect: $lhs <- $rhs")
  ...
}
```

### 例化模块语句

输出例化模块语句包括实例的名字和被例化的模块名。

```scala
case class DefInstance(id: BaseModule, ports: Seq[Port]) extends Definition
```

```scala
def cmd_str(cmd: Command, l: Int): Option[String] = cmd match {
  case e @ DefInstance(id, _) =>
    Some(s"${tab(l)}DefInstance: ${e.name} of ${id.name}")
  ...
}
```

## 参数 IR

参数 **IR** 的类型有:

- 节点参数 Node
- 整型立即数参数 ILit
- 引用参数 Ref
- 模块 IO 参数 ModuleIO

```scala
abstract class Arg

case class Node(id: HasId) extends Arg
case class ILit(n: BigInt) extends Arg
case class Ref(name: String) extends Arg
case class ModuleIO(mod: BaseModule, name: String) extends Arg
```

`arg_str` 方法把以上四种类型的参数转成对应的字符串。

```scala
// play-chisel/chiselFrontend/src/PrintIR.scala
object PrintIR {
  ...
  def arg_str(arg: Arg): String = arg match {
    case Node(id) =>
      arg_str(id.getRef)
    case Ref(name) =>
      s"Reference: $name"
    case ModuleIO(mod, name) =>
      s"ModuleIO: ${mod.getRef.name} $name"
    case lit: ILit =>
      throwException(s"Internal Error! Unexpected ILit: $lit")
  }
}
```

```scala
// play-chisel/src/Main.scala
package playchisel

import chisel3.debug.PrintIR.print_ir

...

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Mux4))

  val emitted = Emitter.emit(circuit)

  val file = new File("Mux4.fir")
  val w = new FileWriter(file)
  w.write(emitted)
  w.close()

  print_ir(circuit)
  // val firrtl = Converter.convert(circuit)

  // val state = CircuitState(firrtl, ChirrtlForm)
  // val compiler = new VerilogCompiler
  // compiler.compile(state)
}
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码02 %}}
{{% /admonition %}}

[play-chisel/tree/chap03-02](https://github.com/colin4124/play-chisel/tree/chap03-02)

# 添加新的 FIRRTL IR

```scala
// play-chisel/firrtl/src/ir/IR.scala
case class SubField(expr: Expression, name: String, tpe: Type) extends Expression with HasName {
  def mapExpr(f: Expression => Expression): Expression = this.copy(expr = f(expr))
  def mapType(f: Type => Type): Expression = this.copy(tpe = f(tpe))
  def mapWidth(f: Width => Width): Expression = this
}

case class DefInstance(name: String, module: String) extends Statement with IsDeclaration {
  def mapStmt(f: Statement => Statement): Statement = this
  def mapExpr(f: Expression => Expression): Statement = this
  def mapType(f: Type => Type): Statement = this
  def mapString(f: String => String): Statement = DefInstance(f(name), module)
  def foreachStmt(f: Statement => Unit): Unit = Unit
  def foreachExpr(f: Expression => Unit): Unit = Unit
  def foreachType(f: Type => Unit): Unit = Unit
  def foreachString(f: String => Unit): Unit = f(name)
}
```

在转成模块端口 IO 的时候，需要考虑是否为子模块的情况，因此需要对比当前的上下文。

如果是子模块，那么它的端口对应的 FIRRTL IR 是子字段的类型 **SubField**。

转换参数的时候也不需要考虑 `ILit`，它在进入这个方法之前以及被处理掉了。如果出现则报错。

```scala
// play-chisel/chiselFrontend/src/internal/firrtl/Converter.scala
import chisel3.internal.throwException

object Converter {
  ...
  def convert(arg: Arg, ctx: Component): fir.Expression = arg match {
    case Node(id) =>
      convert(id.getRef, ctx)
    ...
    case ModuleIO(mod, name) =>
      if (mod eq ctx.id) fir.Reference(name, fir.UnknownType)
      else fir.SubField(fir.Reference(mod.getRef.name, fir.UnknownType), name, fir.UnknownType)
    case lit: ILit =>
      throwException(s"Internal Error! Unexpected ILit: $lit")
  }
}
```

因为这里多了上下文的参数 `ctx: Component`，需要改下其他的方法，把这个参数传进来。

`convertSimpleCommand` 方法需要处理 `DefInstance` 子模块的 IR， `DefPrim` 运算 IR 需要处理 `ILit` 的常量参数。

```scala
// play-chisel/chiselFrontend/src/internal/firrtl/Converter.scala
object Converter {
  ...
  def convert(component: Component): fir.DefModule = component match {
    ...
      fir.Module(name, ports.map(p => convert(p)), convert(cmds.toList, ctx))
  }
  ...
  def convert(cmds: Seq[Command], ctx: Component): fir.Statement = {
    @tailrec
    def rec(acc: Queue[fir.Statement])(cmds: Seq[Command]): Seq[fir.Statement] = {
      if (cmds.isEmpty) {
        ...
      } else convertSimpleCommand(cmds.head, ctx) match {
        ...
      }
    }
    ...
  }
  def convertSimpleCommand(cmd: Command, ctx: Component): Option[fir.Statement] = cmd match {
    case e: DefPrim[_] =>
      val consts = e.args.collect { case ILit(i) => i }
      val args = e.args.flatMap {
        case _: ILit => None
        case other => Some(convert(other, ctx))
      }
      val expr = fir.DoPrim(convert(e.op), args, consts, fir.UnknownType)
      Some(fir.DefNode(e.name, expr))
    case Connect(loc, exp) =>
      Some(fir.Connect(convert(loc, ctx), convert(exp, ctx)))
    case e @ DefInstance(id, _) =>
      Some(fir.DefInstance(e.name, id.name))
    case _ => None
  }
```

添加 Bit 操作。

```scala
// play-chisel/firrtl/src/PrimOps.scala
package firrtl

object PrimOps {
  /** Bit Extraction */
  case object Bits extends PrimOp { override def toString = "bits" }
  
  private lazy val builtinPrimOps: Seq[PrimOp] =
    Seq(Not, And, Or, Bits)
  ...
}
```

调整输出 FIRRTL IR

```scala
// play-chisel/firrtl/src/PrintIR.scala
package firrtl.debug

object PrintIR {
  ...
  def stmt_str(s: fir.Statement, l: Int): String = {
    s match {
      case fir.DefNode(n, v) => s"${tab(l)}DefNode: $n\n${tab(l+1)}${e_str(v, l+1)}"
      case fir.Block(s) => s"${tab(l)}Block\n" + (s map { x => stmt_str(x, l+1) } mkString "\n")
      case fir.Connect(left, right) =>
        s"${tab(l)}Connect\n${tab(l+1)}${e_str(left, l+1)} ${e_str(right, l+1)}"
      case fir.DefInstance(n, m) => s"${tab(l)}DefInstance: inst $n of $m"
    }
  }
  def e_str(e: fir.Expression, l: Int): String = {
    e match {
      ...
      case fir.SubField(e, n, t) => s"SubField(${e_str(e, l)}.$n, ${type_str(t)})"
    }
  }
}
```

```scala
// play-chisel/src/Main.scala
import firrtl.debug.PrintIR.print_fir
object Main extends App {
  ...
  val firrtl = Converter.convert(circuit)
  print_fir(firrtl)
  ...
}
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码03 %}}
{{% /admonition %}}

[play-chisel/tree/chap03-03](https://github.com/colin4124/play-chisel/tree/chap03-03)

# FIRRTL

## 推导类型

增加 `SubField` 和 `DefInstance`

```scala
// play-chisel/firrtl/src/passes/InferTypes.scala
package firrtl.passes

import debug.PrintIR.print_fir

object CInferTypes extends Pass {
  ...
  def run(c: Circuit): Circuit = {
    println("Run CInferTypes ......")
    val mtypes = (c.modules map (m => m.name -> module_type(m))).toMap
    
    def infer_types_e(types: TypeMap)(e: Expression) : Expression =
      e map infer_types_e(types) match {
        ...
        case (e: SubField) => e copy (tpe = field_type(e.expr.tpe, e.name))
      }

    def infer_types_s(types: TypeMap)(s: Statement): Statement = s match {
      ...
      case sx: DefInstance =>
        types(sx.name) = mtypes(sx.module)
        sx
    }

    ...
    
    val res = c copy (modules = c.modules map infer_types)
    println("Done CInferTypes")
    print_fir(res)
    res
  }
}
```

`module_type` 和 `field_type` 方法。

```scala
// play-chisel/firrtl/src/Utils.scala
package firrtl

object Utils {
  ...
  def module_type(m: DefModule): BundleType = BundleType(m.ports map {
    case Port(name, dir, tpe) => Field(name, to_flip(dir), tpe)
  })

  def field_type(v: Type, s: String) : Type = v match {
    case vx: BundleType => vx.fields find (_.name == s) match {
      case Some(f) => f.tpe
      case None => UnknownType
    }
    case vx => UnknownType
  }

  def to_flip(d: Direction): Orientation = d match {
    case Input => Flip
    case Output => Default
  }
}
```

更新 `set_primop_type` 方法，支持 `Bits` 操作。

```scala
// play-chisel/firrtl/src/PrimOps.scala
package firrtl

object PrimOps {
  def PLUS (w1:Width, w2:Width) : Width = (w1, w2) match {
    case (IntWidth(i), IntWidth(j)) => IntWidth(i + j)
    case _ => PlusWidth(w1, w2)
  }

  def MINUS (w1:Width, w2:Width) : Width = (w1, w2) match {
    case (IntWidth(i), IntWidth(j)) => IntWidth(i - j)
    case _ => MinusWidth(w1, w2)
  }

  def set_primop_type (e:DoPrim) : DoPrim = {
  ...
    def c1 = IntWidth(e.consts.head)
    def c2 = IntWidth(e.consts(1))
    
    e copy (tpe = e.op match {
      ...
      case Bits => t1 match {
        case (_: UIntType) => UIntType(PLUS(MINUS(c1, c2), IntWidth(1)))
        case _ => UnknownType
      }
    })
```

```scala
// play-chisel/firrtl/src/WIR.scala
package firrtl

case class PlusWidth(arg1: Width, arg2: Width) extends Width with HasMapWidth {
  def mapWidth(f: Width => Width): Width = PlusWidth(f(arg1), f(arg2))
}
case class MinusWidth(arg1: Width, arg2: Width) extends Width with HasMapWidth {
  def mapWidth(f: Width => Width): Width = MinusWidth(f(arg1), f(arg2))
}
```

`module_type` 引入了 `BundleType`

```scala
// play-chisel/firrtl/src/ir/IR.scala
package firrtl
package ir

/** Orientation of [[Field]] */
abstract class Orientation extends FirrtlNode
case object Default extends Orientation
case object Flip extends Orientation

/** Field of [[BundleType]] */
case class Field(name: String, flip: Orientation, tpe: Type) extends FirrtlNode with HasName

abstract class AggregateType extends Type {
  def mapWidth(f: Width => Width): Type = this
}

case class BundleType(fields: Seq[Field]) extends AggregateType {
  def mapType(f: Type => Type): Type =
    BundleType(fields map (x => x.copy(tpe = f(x.tpe))))
}
```

改下输出 **IR** 的方法：

```scala
// play-chisel/firrtl/src/PrintIR.scala
package firrtl.debug

object PrintIR {
  ...
    def bundle_str(b: fir.BundleType) = {
    b.fields map field_str mkString ", "
  }
  def field_str(f: fir.Field) = {
    s"${f.name}: ${ori_str(f.flip)} ${type_str(f.tpe)}"
  }
  def ori_str(o: fir.Orientation) = o match {
    case fir.Default => "Default"
    case fir.Flip    => "Flip"
  }
  def type_str(d: fir.Type): String = d match {
    ...
    case bundle: fir.BundleType  => s"Bundle( ${bundle_str(bundle)} )"
    ...
  }
  ...
  def stmt_str(s: fir.Statement, l: Int): String = {
    s match {
      ...
      case fir.Connect(left, right) =>
        s"${tab(l)}Connect\n${tab(l+1)}${e_str(left, l+1)}\n${tab(l+1)}${e_str(right, l+1)}"
    }
  }
  def print_fir(ast: fir.Circuit) = {
    println(s"Circuit:")
    println(s"  modules: [${ast.modules map { _.name } mkString ", "}]")
    println(s"  main: ${ast.main}\n")
    ast.modules foreach { m =>
    ...
  }
}

```

```scala
// play-chisel/src/Main.scala
package playchisel

...

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Mux4))

  ...
  
  val firrtl = Converter.convert(circuit)
  print_fir(firrtl)

  val state = CircuitState(firrtl, ChirrtlForm)
  val compiler = new VerilogCompiler
  println("FIRRTL Compiling")
  compiler.compile(state)
}
```

把输出信息删掉。

```scala
// play-chisel/firrtl/src/Compiler.scala
abstract class SeqTransform extends Transform with SeqTransformBased {
  def execute(state: CircuitState): CircuitState = {
    val ret = runTransforms(state)
    CircuitState(ret.circuit, outputForm)
  }
}
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码04 %}}
{{% /admonition %}}

[play-chisel/tree/chap03-04](https://github.com/colin4124/play-chisel/tree/chap03-04)

## ToWorkingIR

```scala
// play-chisel/firrtl/src/passes/Passes.scala
package firrtl.passes

import debug.PrintIR.print_fir

object ToWorkingIR extends Pass {
  def toExp(e: Expression): Expression = e map toExp match {
    ...
    case ex: SubField => WSubField(ex.expr, ex.name, ex.tpe, UnknownFlow)
  }

  def toStmt(s: Statement): Statement = s map toExp match {
    case sx: DefInstance => WDefInstance(sx.name, sx.module, UnknownType)
    ...
  }

  def run (c:Circuit): Circuit = {
    println("Run ToWorkingIR  ......")
    val res = c copy (modules = c.modules map (_ map toStmt))
    println("Done ToWorkingIR.")
    print_fir(res)
    res
  }
}
```

增加工作 IR 类型：

```scala
// play-chisel/firrtl/src/WIR.scala
package firrtl

private[firrtl] trait GenderFromFlow { this: Expression =>
  val flow: Flow
}

case class WSubField(expr: Expression, name: String, tpe: Type, flow: Flow) extends Expression with GenderFromFlow {
  def mapExpr(f: Expression => Expression): Expression = this.copy(expr = f(expr))
  def mapType(f: Type => Type): Expression = this.copy(tpe = f(tpe))
  def mapWidth(f: Width => Width): Expression = this
}

case class WDefInstance(name: String, module: String, tpe: Type) extends Statement with IsDeclaration {
  def mapExpr(f: Expression => Expression): Statement = this
  def mapStmt(f: Statement => Statement): Statement = this
  def mapType(f: Type => Type): Statement = this.copy(tpe = f(tpe))
  def mapString(f: String => String): Statement = this.copy(name = f(name))
  def foreachStmt(f: Statement => Unit): Unit = Unit
  def foreachExpr(f: Expression => Unit): Unit = Unit
  def foreachType(f: Type => Unit): Unit = f(tpe)
  def foreachString(f: String => Unit): Unit = f(name)
}
```

```scala
// play-chisel/firrtl/src/PrintIR.scala
package firrtl.debug

import firrtl._
import firrtl.{ir => fir}

object PrintIR {
  ...
  def stmt_str(s: fir.Statement, l: Int): String = {
    s match {
      ...
      case WDefInstance(n, m, t) => s"${tab(l)}WDefInstance($n, $m, ${type_str(t)})"
    }
  }
  
  def e_str(e: fir.Expression, l: Int): String = {
    e match {
      ...
      case WSubField(e, n, t, f) => s"WSubField(${e_str(e, l)}.${n}: ${type_str(t)} ${f_str(f)})"
    }
  }
}
```

这里引入了未知类型（UnknownType）、未知种类（UnknownKind）和未知流向（UnknownFlow）。这些由后续的 `InferTypes`、`ResloveKinds`和`ResolveFlows` 解决。

```shell
$ mill chisel3.run
```

{{% admonition tip 源码05 %}}
{{% /admonition %}}

[play-chisel/tree/chap03-05](https://github.com/colin4124/play-chisel/tree/chap03-05)

## 解析种类和流向

```scala
// play-chisel/firrtl/src/passes/Resolves.scala
package firrtl.passes

import debug.PrintIR.print_fir

object ResolveKinds extends Pass {
  def find_stmt(kinds: KindMap)(s: Statement):Statement = {
    s match {
      ...
      case sx: WDefInstance => kinds(sx.name) = InstanceKind
    }
    ...
  }
  ...
  def run(c: Circuit): Circuit = {
    println("Run ResolveKinds ......")
    val res = c copy (modules = c.modules map resolve_kinds)
    println("Done ResolveKinds.")
    print_fir(res)
    res
  }
}

object ResolveFlows extends Pass {
  def resolve_e(g: Flow)(e: Expression): Expression = e match {
    ...
    case WSubField(exp, name, tpe, _) => WSubField(
      Utils.field_flip(exp.tpe, name) match {
        case Default => resolve_e(g)(exp)
        case Flip => resolve_e(Utils.swap(g))(exp)
      }, name, tpe, g)
    ...
  }
  ...
  def run(c: Circuit): Circuit = {
    println("Run ResolveFlows ......")
    val res = c copy (modules = c.modules map resolve_flow)
    println("Done ResolveFlows.")
    print_fir(res)
    res
  }
}
```

```scala
// play-chisel/firrtl/src/Utils.scala
object Utils {
  ...
  def swap(g: Flow) : Flow = g match {
    case UnknownFlow => UnknownFlow
    case SourceFlow => SinkFlow
    case SinkFlow => SourceFlow
  }
  def field_flip(v: Type, s: String): Orientation = v match {
    case vx: BundleType => vx.fields find (_.name == s) match {
      case Some(ft) => ft.flip
      case None => Default
    }
    case vx => Default
  }
}
```

```scala
// play-chisel/firrtl/src/WIR.scala

case object InstanceKind extends Kind
```

```scala
// play-chisel/firrtl/src/PrintIR.scala
object PrintIR {
  ...
  def k_str(k: Kind): String = k match {
    case InstanceKind => "InstanceKind"
    ...
  }
}
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码06 %}}
{{% /admonition %}}

[play-chisel/tree/chap03-06](https://github.com/colin4124/play-chisel/tree/chap03-06)

## 推导类型

```scala
// play-chisel/firrtl/src/passes/InferTypes.scala
package firrtl.passes

import debug.PrintIR.print_fir

object InferTypes extends Pass {
  type TypeMap = collection.mutable.LinkedHashMap[String, Type]

  def run(c: Circuit): Circuit = {
    val namespace = Namespace()
    val mtypes = (c.modules map (m => m.name -> module_type(m))).toMap

    def remove_unknowns_w(w: Width): Width = w match {
      case UnknownWidth => VarWidth(namespace.newName("w"))
      case wx => wx
    }

    def remove_unknowns(t: Type): Type =
      t map remove_unknowns map remove_unknowns_w

    def infer_types_e(types: TypeMap)(e: Expression): Expression =
      e map infer_types_e(types) match {
        case e: WRef => e copy (tpe = types(e.name))
        case e: WSubField => e copy (tpe = field_type(e.expr.tpe, e.name))
        case e: DoPrim => PrimOps.set_primop_type(e)
      }

    def infer_types_s(types: TypeMap)(s: Statement): Statement = s match {
      case sx: WDefInstance =>
        val t = mtypes(sx.module)
        types(sx.name) = t
        sx copy (tpe = t)
      case sx: DefNode =>
        val sxx = (sx map infer_types_e(types)).asInstanceOf[DefNode]
        val t = remove_unknowns(sxx.value.tpe)
        types(sx.name) = t
        sxx map infer_types_e(types)
      case sx => sx map infer_types_s(types) map infer_types_e(types)
    }

    def infer_types_p(types: TypeMap)(p: Port): Port = {
      val t = remove_unknowns(p.tpe)
      types(p.name) = t
      p copy (tpe = t)
    }

    def infer_types(m: DefModule): DefModule = {
      val types = new TypeMap
      m map infer_types_p(types) map infer_types_s(types)
    }

    println("Run InferTypes ......")
    val res = c copy (modules = c.modules map infer_types)
    println("Done InferTypes.")
    print_fir(res)
    res
  }
}
```

```scala
package firrtl

import scala.collection.mutable
import firrtl.ir._

class Namespace private {
  private val tempNamePrefix: String = "_GEN"
  // Begin with a tempNamePrefix in namespace so we always have a number suffix
  private val namespace = mutable.HashSet[String](tempNamePrefix)
  // Memoize where we were on a given prefix
  private val indices = mutable.HashMap[String, Int]()

  def tryName(value: String): Boolean = {
    val unused = !contains(value)
    if (unused) namespace += value
    unused
  }

  def contains(value: String): Boolean = namespace.contains(value)

  def newName(value: String): String = {
    // First try, just check
    if (tryName(value)) value
    else {
      var idx = indices.getOrElse(value, 0)
      var str = value
      do {
        str = s"${value}_$idx"
        idx += 1
      }
      while (!(tryName(str)))
      indices(value) = idx
      str
    }
  }
}

object Namespace {
  def apply(names: Seq[String] = Nil): Namespace = {
    val namespace = new Namespace
    namespace.namespace ++= names
    namespace
  }
}
```

```scala
// play-chisel/firrtl/src/WIR.scala
case class VarWidth(name: String) extends Width with HasMapWidth {
  def mapWidth(f: Width => Width): Width = this
}
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码07 %}}
{{% /admonition %}}

[play-chisel/tree/chap03-07](https://github.com/colin4124/play-chisel/tree/chap03-07)


## 删除重复的模块

```shell
$ touch firrtl/src/transforms/Dedup.scala
```

```scala
// play-chisel/firrtl/src/transforms/Dedup.scala
package firrtl
package transforms

import firrtl.ir._

import debug.PrintIR.print_fir

class DedupModules extends Transform {
  def inputForm: CircuitForm = HighForm
  def outputForm: CircuitForm = HighForm

  def execute(state: CircuitState): CircuitState = {
    println("Run Dedup...")
    val newC = run(state.circuit)
    println("Done Dedup.")
    print_fir(newC)
    state.copy(circuit = newC)
  }

  def run(c: Circuit): Circuit = {
    // Maps module name to corresponding dedup module
    val dedupMap = DedupModules.deduplicate(c)

    // Use old module list to preserve ordering
    val dedupedModules = c.modules.map(m => dedupMap(m.name)).distinct

    c.copy(modules = dedupedModules)
  }
}
```

### deduplicate 方法

```scala
// play-chisel/firrtl/src/transforms/Dedup.scala
import scala.collection.mutable

import firrtl.analyses.InstanceGraph
import firrtl.Utils.throwInternalError

object DedupModules {
  def deduplicate(circuit: Circuit): Map[String, DefModule] = {
    val (moduleMap, moduleLinearization) = {
      val iGraph = new InstanceGraph(circuit)
      (iGraph.moduleMap, iGraph.moduleOrder.reverse)
    }
    moduleLinearization foreach { x => println(x.name) }

    val main = circuit.main
    val (tag2all, tagMap) = buildRTLTags(moduleLinearization)

    // Set tag2name to be the best dedup module name
    val moduleIndex = circuit.modules.zipWithIndex.map{case (m, i) => m.name -> i}.toMap
    def order(l: String, r: String): String = {
     if (l == main) l
     else if (r == main) r
     else if (moduleIndex(l) < moduleIndex(r)) l else r
    }

    // Maps a module's tag to its deduplicated module
    val tag2name = mutable.HashMap.empty[String, String]
    tag2all.foreach { case (tag, all) => tag2name(tag) = all.reduce(order)}


    // Create map from original to dedup name
    val name2name = moduleMap.keysIterator.map{ originalModule =>
      tagMap.get(originalModule) match {
        case Some(tag) => originalModule -> tag2name(tag)
        case None => originalModule -> originalModule
        case other => throwInternalError(other.toString)
      }
    }.toMap

    val dedupedName2module = tag2name.map({ case (_, name) => name -> DedupModules.dedupInstances(name, moduleMap, name2name) })
    println("dedupedName2module")
    dedupedName2module foreach { case (n, m) => println(s"$n -> ${m.name}") }

    // Build map from original name to corresponding deduped module
    val name2module = tag2all.flatMap({ case (tag, names) => names.map(n => n -> dedupedName2module(tag2name(tag))) })

    name2module.toMap
  }
}
```

### InstanceGraph

```scala
// play-chisel/firrtl/src/analyses/InstanceGraph.scala
package firrtl.analyses

import scala.collection.mutable

import firrtl._
import firrtl.ir._
import firrtl.graph._
import firrtl.Utils._
import firrtl.traversals.Foreachers._

class InstanceGraph(c: Circuit) {
  val moduleMap = c.modules.map({m => (m.name,m) }).toMap

  private val instantiated = new mutable.LinkedHashSet[String]
  private val childInstances =
    new mutable.LinkedHashMap[String, mutable.LinkedHashSet[WDefInstance]]
  for (m <- c.modules) {
    childInstances(m.name) = new mutable.LinkedHashSet[WDefInstance]
    m.foreach(InstanceGraph.collectInstances(childInstances(m.name)))
    instantiated ++= childInstances(m.name).map(i => i.module)
  }

  private val instanceGraph = new MutableDiGraph[WDefInstance]
  private val instanceQueue = new mutable.Queue[WDefInstance]

  for (subTop <- c.modules.view.map(_.name).filterNot(instantiated)) {
    val topInstance = WDefInstance(subTop,subTop)
    instanceQueue.enqueue(topInstance)
    while (instanceQueue.nonEmpty) {
      val current = instanceQueue.dequeue
      instanceGraph.addVertex(current)
      for (child <- childInstances(current.module)) {
        if (!instanceGraph.contains(child)) {
          instanceQueue.enqueue(child)
          instanceGraph.addVertex(child)
        }
        instanceGraph.addEdge(current,child)
      }
    }
  }

  lazy val graph = DiGraph(instanceGraph)

  def moduleOrder: Seq[DefModule] = {
    graph.transformNodes(_.module).linearize.map(moduleMap(_))
  }
}

object InstanceGraph {
  def collectInstances(insts: mutable.Set[WDefInstance])
                      (s: Statement): Unit = s match {
    case i: WDefInstance => insts += i
    case i: DefInstance => throwInternalError("Expecting WDefInstance, found a DefInstance!")
    case _ => s.foreach(collectInstances(insts))
  }
}
```

```scala
// play-chisel/firrtl/src/WIR.scala

object WDefInstance {
  def apply(name: String, module: String): WDefInstance = new WDefInstance(name, module, UnknownType)
}
```

### DiGraph

```scala
// play-chisel/firrtl/src/graph/DiGraph.scala
package firrtl.graph

import scala.collection.{Set, Map}
import scala.collection.mutable
import scala.collection.mutable.{LinkedHashSet, LinkedHashMap}

class CyclicException(val node: Any) extends Exception(s"No valid linearization for cyclic graph, found at $node")

object DiGraph {
  def apply[T](mdg: MutableDiGraph[T]): DiGraph[T] = mdg
}

class DiGraph[T] private[graph] (private[graph] val edges: LinkedHashMap[T, LinkedHashSet[T]]) {
  def contains(v: T): Boolean = edges.contains(v)

  def getVertices: Set[T] = new LinkedHashSet ++ edges.map({ case (k, _) => k })

  def seededLinearize(seed: Option[Seq[T]] = None): Seq[T] = {
    val order = new mutable.ArrayBuffer[T]
    val unmarked = new mutable.LinkedHashSet[T]
    val tempMarked = new mutable.LinkedHashSet[T]

    case class LinearizeFrame[A](v: A, expanded: Boolean)
    val callStack = mutable.Stack[LinearizeFrame[T]]()

    unmarked ++= seed.getOrElse(getVertices)
    while (unmarked.nonEmpty) {
      callStack.push(LinearizeFrame(unmarked.head, false))
      while (callStack.nonEmpty) {
        val LinearizeFrame(n, expanded) = callStack.pop()
        if (!expanded) {
          if (tempMarked.contains(n)) {
            throw new CyclicException(n)
          }
          if (unmarked.contains(n)) {
            tempMarked += n
            unmarked -= n
            callStack.push(LinearizeFrame(n, true))
            // We want to visit the first edge first (so push it last)
            for (m <- edges.getOrElse(n, Set.empty).toSeq.reverse) {
              callStack.push(LinearizeFrame(m, false))
            }
          }
        } else {
          tempMarked -= n
          order.append(n)
        }
      }
    }

    order.reverse.toSeq
  }

  def linearize: Seq[T] = seededLinearize(None)

  def transformNodes[Q](f: (T) => Q): DiGraph[Q] = {
    val eprime = edges.map({ case (k, _) => (f(k), new LinkedHashSet[Q]) })
    edges.foreach({ case (k, v) => eprime(f(k)) ++= v.map(f(_)) })
    new DiGraph(eprime)
  }
}

class MutableDiGraph[T] extends DiGraph[T](new LinkedHashMap[T, LinkedHashSet[T]]) {
  def addVertex(v: T): T = {
    edges.getOrElseUpdate(v, new LinkedHashSet[T])
    v
  }
  def addEdge(u: T, v: T): Unit = {
    require(contains(u))
    require(contains(v))
    edges(u) += v
  }
}
```

### Foreachers

```scala
// play-chisel/firrtl/src/traversals/Foreachers.scala
object Foreachers {
  ...
  /** Module Foreachers */
  private trait ModuleForMagnet {
    def foreach(module: DefModule): Unit
  }
  private object ModuleForMagnet {
    implicit def forStmt(f: Statement => Unit): ModuleForMagnet = new ModuleForMagnet {
      def foreach(module: DefModule): Unit = module foreachStmt f
    }
    implicit def forPorts(f: Port => Unit): ModuleForMagnet = new ModuleForMagnet {
      def foreach(module: DefModule): Unit = module foreachPort f
    }
    implicit def forString(f: String => Unit): ModuleForMagnet = new ModuleForMagnet {
      def foreach(module: DefModule): Unit = module foreachString f
    }
  }
  implicit class ModuleForeach(val _module: DefModule) extends AnyVal {
    def foreach[T](f: T => Unit)(implicit magnet: (T => Unit) => ModuleForMagnet): Unit = magnet(f).foreach(_module)
  }
}
```

```scala
// play-chisel/firrtl/src/ir/IR.scala
abstract class DefModule extends FirrtlNode with IsDeclaration {
  ...
  def foreachStmt(f: Statement => Unit): Unit
  def foreachPort(f: Port => Unit): Unit
  def foreachString(f: String => Unit): Unit
}

case class Module(name: String, ports: Seq[Port], body: Statement) extends DefModule {
  ...
  def foreachStmt(f: Statement => Unit): Unit = f(body)
  def foreachPort(f: Port => Unit): Unit = ports.foreach(f)
  def foreachString(f: String => Unit): Unit = f(name)
}
```

### buildRTLTags

```scala
// play-chisel/firrtl/src/transforms/Dedup.scala
object DedupModules {
  ...
    def buildRTLTags(moduleLinearization: Seq[DefModule]): (collection.Map[String, collection.Set[String]], mutable.HashMap[String, String]) = {
    // Maps a module name to its agnostic name
    val tagMap = mutable.HashMap.empty[String, String]

    // Maps a tag to all matching module names
    val tag2all = mutable.HashMap.empty[String, mutable.HashSet[String]]

    def fastSerializedHash(s: Statement): Int ={
      def serialize(builder: StringBuilder, nindent: Int)(s: Statement): Unit = s match {
        case Block(stmts) => stmts.map {
          val x = serialize(builder, nindent)(_)
          builder ++= "\n"
          x
        }
        case other: Statement =>
          builder ++= ("  " * nindent)
          builder ++= other.serialize
      }
      val builder = new mutable.StringBuilder()
      serialize(builder, 0)(s)
      builder.hashCode()
    }

    moduleLinearization.foreach { originalModule =>
      // Build name-agnostic module
      val agnosticModule = originalModule

      // Build tag
      val builder = new mutable.ArrayBuffer[Any]()
      agnosticModule.ports.foreach { builder ++= _.serialize }
      agnosticModule match {
        case Module(_, _, b) => builder ++= fastSerializedHash(b).toString()//.serialize
      }
      val tag = builder.hashCode().toString

      tagMap += originalModule.name -> tag

      // Set tag's module to be the first matching module
      val all = tag2all.getOrElseUpdate(tag, mutable.HashSet.empty[String])
      all += originalModule.name
    }
    (tag2all, tagMap)
  }
}
```

### 序列化

```scala
// play-chisel/firrtl/src/ir/IR.scala
import Utils.indent

abstract class FirrtlNode {
  def serialize: String
}
abstract class PrimOp extends FirrtlNode {
  def serialize: String = this.toString
}
case class Reference(name: String, tpe: Type) extends Expression with HasName {
  def serialize: String = name
  ...
}
case class SubField(expr: Expression, name: String, tpe: Type) extends Expression with HasName {
  def serialize: String = s"${expr.serialize}.$name"
  ...
}
case class DoPrim(op: PrimOp, args: Seq[Expression], consts: Seq[BigInt], tpe: Type) extends Expression {
  def serialize: String = op.serialize + "(" +
    (args.map(_.serialize) ++ consts.map(_.toString)).mkString(", ") + ")"
  ...
}
case class DefInstance(name: String, module: String) extends Statement with IsDeclaration {
  def serialize: String = s"inst $name of $module"
  ...
}
case class DefNode(name: String, value: Expression) extends Statement with IsDeclaration {
  def serialize: String = s"node $name = ${value.serialize}"
  ...
}
case class Block(stmts: Seq[Statement]) extends Statement {
  def serialize: String = stmts map (_.serialize) mkString "\n"
  ...
}
case class Connect(loc: Expression, expr: Expression) extends Statement {
  def serialize: String =  s"${loc.serialize} <= ${expr.serialize}"
  ...
}
class IntWidth(val width: BigInt) extends Width {
  def serialize: String = s"<$width>"
}
case object UnknownWidth extends Width {
  def serialize: String = ""
}
case object Default extends Orientation {
  def serialize: String = ""
}
case object Flip extends Orientation {
  def serialize: String = "flip "
}

case class Field(name: String, flip: Orientation, tpe: Type) extends FirrtlNode with HasName {
  def serialize: String = flip.serialize + name + " : " + tpe.serialize
}
case class UIntType(width: Width) extends GroundType {
  def serialize: String = "UInt" + width.serialize
  ...
}
case class BundleType(fields: Seq[Field]) extends AggregateType {
  def serialize: String = "{ " + (fields map (_.serialize) mkString ", ") + "}"
  ...
}
case object UnknownType extends Type {
  def serialize: String = "?"
  ...
}
case object Input extends Direction {
  def serialize: String = "input"
}
case object Output extends Direction {
  def serialize: String = "output"
}
case class Port(...) extends FirrtlNode with IsDeclaration {
  def serialize: String = s"${direction.serialize} $name : ${tpe.serialize}"
}

abstract class DefModule extends FirrtlNode with IsDeclaration {
  protected def serializeHeader(tpe: String): String =
    s"$tpe $name :${indent(ports.map("\n" + _.serialize).mkString)}\n"
  ...
}
case class Module(name: String, ports: Seq[Port], body: Statement) extends DefModule {
  def serialize: String = serializeHeader("module") + indent("\n" + body.serialize)
  ...
}
case class Circuit(modules: Seq[DefModule], main: String) extends FirrtlNode {
  def serialize: String =
    s"circuit $main :" +
      (modules map ("\n" + _.serialize) map indent mkString "\n") + "\n"
  ...
}
```

```scala
// play-chisel/firrtl/src/WIR.scala
case class WRef(name: String, tpe: Type, kind: Kind, flow: Flow) extends Expression {
  def serialize: String = name
  ...
}
case class WSubField(expr: Expression, name: String, tpe: Type, flow: Flow) extends Expression with GenderFromFlow {
  def serialize: String = s"${expr.serialize}.$name"
  ...
}
case class WDefInstance(name: String, module: String, tpe: Type) extends Statement with IsDeclaration {
  def serialize: String = s"inst $name of $module"
  ...
}
case class VarWidth(name: String) extends Width with HasMapWidth {
  def serialize: String = name
  ...
}
case class PlusWidth(arg1: Width, arg2: Width) extends Width with HasMapWidth {
  def serialize: String = "(" + arg1.serialize + " + " + arg2.serialize + ")"
  ...
}
case class MinusWidth(arg1: Width, arg2: Width) extends Width with HasMapWidth {
  def serialize: String = "(" + arg1.serialize + " - " + arg2.serialize + ")"
  ...
}
case class MaxWidth(args: Seq[Width]) extends Width with HasMapWidth {
  def serialize: String = args map (_.serialize) mkString ("max(", ", ", ")")
  ...
}
```

```scala
// play-chisel/firrtl/src/Utils.scala
object Utils {
  /** Indent the results of [[ir.FirrtlNode.serialize]] */
  def indent(str: String) = str replaceAllLiterally ("\n", "\n  ")
  ...
}
```

### dedupInstances

```scala
// play-chisel/firrtl/src/transforms/Dedup.scala
object DedupModules {
  def dedupInstances(originalModule: String,
                     moduleMap: Map[String, DefModule],
                     name2name: Map[String, String]): DefModule = {
    val module = moduleMap(originalModule)

    // Get all instances to know what to rename in the module
    val instances = mutable.Set[WDefInstance]()
    InstanceGraph.collectInstances(instances)(module.asInstanceOf[Module].body)
    val instanceModuleMap = instances.map(i => i.name -> i.module).toMap

    def getNewModule(old: String): DefModule = {
      moduleMap(name2name(old))
    }
    // Define rename functions
    def renameOfModule(instance: String, ofModule: String): String = {
      name2name(ofModule)
    }
    val typeMap = mutable.HashMap[String, Type]()
    def retype(name: String)(tpe: Type): Type = {
      if (typeMap.contains(name)) typeMap(name) else {
        if (instanceModuleMap.contains(name)) {
          val newType = Utils.module_type(getNewModule(instanceModuleMap(name)))
          typeMap(name) = newType
          newType
        } else {
          tpe
        }
      }
    }

    // Change module internals
    changeInternals({n => n}, retype, renameOfModule)(module)
  }
  ...
}
```

### changeInternals

```scala
// play-chisel/firrtl/src/transforms/Dedup.scala
import firrtl.Mappers._

object DedupModules {
  def changeInternals(rename: String=>String,
                      retype: String=>Type=>Type,
                      renameOfModule: (String, String)=>String,
                      renameExps: Boolean = true
                     )(module: DefModule): DefModule = {
    def onPort(p: Port): Port = Port(rename(p.name), p.direction, retype(p.name)(p.tpe))
    def onExp(e: Expression): Expression = e match {
      case WRef(n, t, k, g) => WRef(rename(n), retype(n)(t), k, g)
      case WSubField(expr, n, tpe, kind) =>
        val fieldIndex = expr.tpe.asInstanceOf[BundleType].fields.indexWhere(f => f.name == n)
        val newExpr = onExp(expr)
        val newField = newExpr.tpe.asInstanceOf[BundleType].fields(fieldIndex)
        val finalExpr = WSubField(newExpr, newField.name, newField.tpe, kind)
        finalExpr
      case other => other map onExp
    }
    def onStmt(s: Statement): Statement = s match {
      case DefNode(name, value) =>
        if(renameExps) DefNode(rename(name), onExp(value))
        else DefNode(rename(name), value)
      case WDefInstance(n, m, t) =>
        val newmod = renameOfModule(n, m)
        WDefInstance(rename(n), newmod, retype(n)(t))
      case DefInstance(n, m) => DefInstance(rename(n), renameOfModule(n, m))
      case h: IsDeclaration =>
        val temp = h map rename map retype(h.name)
        if(renameExps) temp map onExp else temp
      case other =>
        val temp = other map onStmt
        if(renameExps) temp map onExp else temp
    }
    module map onPort map onStmt
  }
  ...
}
```

```scala
// play-chisel/firrtl/src/Compiler.scala
object CompilerUtils {
  def getLoweringTransforms(inputForm: CircuitForm, outputForm: CircuitForm): Seq[Transform] = {
    if (outputForm >= inputForm) {
      ...
    } else {
      inputForm match {
        ...
        case HighForm =>
          Seq(new IRToWorkingIR,
              new ResolveAndCheck,
              new transforms.DedupModules,
              new HighFirrtlToMiddleFirrtl) ++
            getLoweringTransforms(MidForm, outputForm)
        ...
      }
    }
  }
}
```

```shell
$ mill chisel3.run
```

{{% admonition tip 源码08 %}}
{{% /admonition %}}

[play-chisel/tree/chap03-08](https://github.com/colin4124/play-chisel/tree/chap03-08)

## ConstantPropagation

