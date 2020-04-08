---
title: "第三章：数据类型"
date: 2019-12-10T20:47:53+08:00
draft: true
---

# 目标

```scala
class GreaterThan extends RawModule {
  val in0 = IO(Input(SInt(3.W)))
  val in1 = IO(Input(SInt(3.W)))
  val out = IO(Output(Bool()))
  out := in0 > in1
}
```

```firrtl
circuit GreaterThan :
  module GreaterThan :
    input in0 : SInt<3>
    input in1 : SInt<3>
    output out : UInt<1>

    node _T = gt(in0, in1)
    out <= _T
```

```verilog
module GreaterThan(
  input  [2:0] in0,
  input  [2:0] in1,
  output       out
);
  assign out = $signed(in0) > $signed(in1);
endmodule
```

# 实现

修改 `Bits.scala` 文件，新增 `SInt` 类型。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  override def toString: String = {
    s"SInt$width"
  }

  private[chisel3] override def cloneTypeWidth(w: Width): this.type =
    new SInt(w).asInstanceOf[this.type]

  def > (that: SInt): Bool = compop(GreaterOp, that)
}
```

`compop` 是比较操作（`compare operation`），`def >` 方法是比较自身是否大于对方 `GreaterOp`，返回结果是 `Bool` 类型。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed abstract class Bits(private[chisel3] val width: Width) extends Element {
  ...
  private[chisel3] def compop(op: PrimOp, other: Bits): Bool = {
    requireIsHardware(this, "bits operated on")
    requireIsHardware(other, "bits operated on")
    pushOp(DefPrim(Bool(), op, this.ref, other.ref))
  }
}
```

`compop` 向当前模块添加的命令 `DefPrim`，运算结果是 `Bool()` 类型。


```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val GreaterOp = PrimOp("gt")
}

```

**FIRRTL** 大于运算符是 `gt`。

```scala
// play-chisel/chiselFrontend/src/SIntFactory.scala
package chisel3

import chisel3.internal.firrtl.Width

trait SIntFactory {
  def apply(): SInt = apply(Width())
  def apply(width: Width): SInt = new SInt(width)
}
```


```scala
// play-chisel/chiselFrontend/src/package.scala
package object chisel3 {
  ...
  object SInt extends SIntFactory
}
```

跟 `UInt` 一样的方式添加一个 `SInt()` 单例对象例化 `SInt` 数据类型。

```scala
// play-chisel/src/internal/firrtl/Emitter.scala
private class Emitter(circuit: Circuit) {
  private def emitType(d: Data, clearDir: Boolean = false): String = d match {
    ...
    case d: SInt => s"SInt${d.width}"
  }
}
```

生成的 **FIRRTL** 语法是 `SInt<5>`（例如宽度是 5） 或者 `SInt`（没有制定宽度），宽度的字符串表示 `d.width` 自带了 `<>` 尖括号。

# 目标

``` scala
  1.U       // decimal 1-bit lit from Scala Int.
  "ha".U    // hexadecimal 4-bit lit from string.
  "o12".U   // octal 4-bit lit from string.
  "b1010".U // binary 4-bit lit from string.

  5.S    // signed decimal 4-bit lit from Scala Int.
  -8.S   // negative decimal 4-bit lit from Scala Int.
  5.U    // unsigned decimal 3-bit lit from Scala Int.

  8.U(4.W) // 4-bit unsigned decimal, value 8.
  -152.S(32.W) // 32-bit signed decimal, value -152.

  true.B // Bool lits from Scala lits.
  false.B

  "h_dead_beef".U   // 32-bit lit of type UInt

  "ha".asUInt(8.W)     // hexadecimal 8-bit lit of type UInt
  "o12".asUInt(6.W)    // octal 6-bit lit of type UInt
  "b1010".asUInt(12.W) // binary 12-bit lit of type UInt

  5.asSInt(7.W) // signed decimal 7-bit lit of type SInt
  5.asUInt(8.W) // unsigned decimal 8-bit lit of type UInt
```

```scala
class LitValue extends RawModule {
  val outU0 = IO(Output(UInt(5.W)))
  val outU1 = IO(Output(UInt(6.W)))
  val outU2 = IO(Output(UInt(7.W)))
  val outU3 = IO(Output(UInt(8.W)))
  val outU4 = IO(Output(UInt(9.W)))
  val outU5 = IO(Output(UInt(4.W)))
  val outU6 = IO(Output(UInt(32.W)))
  val outU7 = IO(Output(UInt(8.W)))
  val outU8 = IO(Output(UInt(6.W)))
  val outU9 = IO(Output(UInt(12.W)))
  val outU10 = IO(Output(UInt(8.W)))
  val outS0 = IO(Output(SInt(5.W)))
  val outS1 = IO(Output(SInt(4.W)))
  val outS2 = IO(Output(SInt(32.W)))
  val outS3 = IO(Output(SInt(7.W)))
  val outB0 = IO(Output(Bool()))
  val outB1 = IO(Output(Bool()))
  val outB2 = IO(Output(Bool()))
  val outB3 = IO(Output(Bool()))

  outU0 := 1.U       // decimal 1-bit lit from Scala Int.
  outU1 := "ha".U    // hexadecimal 4-bit lit from string.
  outU2 := "o12".U   // octal 4-bit lit from string.
  outU3 := "b1010".U // binary 4-bit lit from string.
  outU4 := 5.U       // unsigned decimal 3-bit lit from Scala Int.
  outU5 := 8.U(4.W)  // 4-bit unsigned decimal, value 8.

  outS0 := 5.S          // signed decimal 4-bit lit from Scala Int.
  outS1 := -8.S         // negative decimal 4-bit lit from Scala Int.
  outS2 := -152.S(32.W) // 32-bit signed decimal, value -152.

  outB0 := true.B
  outB1 := false.B
  outB2 := 1.B
  outB3 := 0.B

  outU6 := "h_dead_beef".U   // 32-bit lit of type UInt

  outU7 := "ha".asUInt(8.W)     // hexadecimal 8-bit lit of type UInt
  outU8 := "o12".asUInt(6.W)    // octal 6-bit lit of type UInt
  outU9 := "b1010".asUInt(12.W) // binary 12-bit lit of type UInt

  outS3 := 5.asSInt(7.W) // signed decimal 7-bit lit of type SInt
  outU10 := 5.asUInt(8.W) // unsigned decimal 8-bit lit of type UInt
}

object Main extends App {
  val (circuit, _) = Builder.build(Module(new LitValue))

  val emitted = Emitter.emit(circuit)

  val file = new File("LitValue.fir")
  val w = new FileWriter(file)
  w.write(emitted)
  w.close()
}
```

``` firrtl
circuit LitValue : 
  module LitValue : 
    output outU0 : UInt<5>
    output outU1 : UInt<6>
    output outU2 : UInt<7>
    output outU3 : UInt<8>
    output outU4 : UInt<9>
    output outU5 : UInt<4>
    output outU6 : UInt<32>
    output outU7 : UInt<8>
    output outU8 : UInt<6>
    output outU9 : UInt<12>
    output outU10 : UInt<8>
    output outS0 : SInt<5>
    output outS1 : SInt<4>
    output outS2 : SInt<32>
    output outS3 : SInt<7>
    output outB0 : UInt<1>
    output outB1 : UInt<1>
    output outB2 : UInt<1>
    output outB3 : UInt<1>
    
    outU0 <= UInt<1>("h01")
    outU1 <= UInt<4>("h0a")
    outU2 <= UInt<4>("h0a")
    outU3 <= UInt<4>("h0a")
    outU4 <= UInt<3>("h05")
    outU5 <= UInt<4>("h08")
    outS0 <= asSInt(UInt<4>("h05"))
    outS1 <= asSInt(UInt<4>("h08"))
    outS2 <= asSInt(UInt<32>("h0ffffff68"))
    outB0 <= UInt<1>("h01")
    outB1 <= UInt<1>("h00")
    outB2 <= UInt<1>("h01")
    outB3 <= UInt<1>("h00")
    outU6 <= UInt<32>("h0deadbeef")
    outU7 <= UInt<8>("h0a")
    outU8 <= UInt<6>("h0a")
    outU9 <= UInt<12>("h0a")
    outS3 <= asSInt(UInt<7>("h05"))
    outU10 <= UInt<8>("h05")
```

```verilog
module LitValue(
  output [4:0]  outU0,
  output [5:0]  outU1,
  output [6:0]  outU2,
  output [7:0]  outU3,
  output [8:0]  outU4,
  output [3:0]  outU5,
  output [31:0] outU6,
  output [7:0]  outU7,
  output [5:0]  outU8,
  output [11:0] outU9,
  output [7:0]  outU10,
  output [4:0]  outS0,
  output [3:0]  outS1,
  output [31:0] outS2,
  output [6:0]  outS3,
  output        outB0,
  output        outB1,
  output        outB2,
  output        outB3
);
  assign outU0 = 5'h1;
  assign outU1 = 6'ha;
  assign outU2 = 7'ha;
  assign outU3 = 8'ha;
  assign outU4 = 9'h5;
  assign outU5 = 4'h8;
  assign outU6 = 32'hdeadbeef;
  assign outU7 = 8'ha;
  assign outU8 = 6'ha;
  assign outU9 = 12'ha;
  assign outU10 = 8'h5;
  assign outS0 = 5'sh5;
  assign outS1 = -4'sh8;
  assign outS2 = -32'sh98;
  assign outS3 = 7'sh5;
  assign outB0 = 1'h1;
  assign outB1 = 1'h0;
  assign outB2 = 1'h1;
  assign outB3 = 1'h0;
endmodule
```

# 实现

## 整型转字面量

第一章介绍过 `3.U` 这种语法糖简化把 **Scala** 字面量生成 **Chisel** 数据类型的方式，这里补充其他的数据类型。

```scala
// play-chisel/chiselFrontend/src/package.scala
package object chisel3 {
  ...
  implicit class fromBooleanToLiteral(boolean: Boolean) {
    def B: Bool = Bool.Lit(boolean)
  }

  implicit class fromBigIntToLiteral(bigint: BigInt) {
    /** Int to Bool conversion, allowing compact syntax like 1.B and 0.B
      */
    def B: Bool = bigint match {
      case bigint if bigint == 0 => Bool.Lit(false)
      case bigint if bigint == 1 => Bool.Lit(true)
      case bigint => throwException(s"Cannot convert $bigint to Bool, must be 0 or 1"); Bool.Lit(false)
    }
    def U: UInt = UInt.Lit(bigint, Width())
    def S: SInt = SInt.Lit(bigint, Width())
    def U(width: Width): UInt = UInt.Lit(bigint, width)
    def S(width: Width): SInt = SInt.Lit(bigint, width)
    def asUInt(): UInt = UInt.Lit(bigint, Width())
    def asSInt(): SInt = SInt.Lit(bigint, Width())
    def asUInt(width: Width): UInt = UInt.Lit(bigint, width)
    def asSInt(width: Width): SInt = SInt.Lit(bigint, width)
  }

  implicit class fromIntToLiteral(int: Int) extends fromBigIntToLiteral(int)
  implicit class fromLongToLiteral(long: Long) extends fromBigIntToLiteral(long)
}
```

`false.B`、`3.S`、`10.asUInt(3.W)` 编译器都会自己找到对应符合参数数据类型的隐式类，再调用里面同名的方法： `def B = Bool.Lit(false)`、`def S = SInt.Lit(3, Width())` 和 `def asUInt(3.W) = UInt.Lit(10, 3.W)`。

`fromIntToLiteral` 和 `fromLongToLiteral` 都是分别直接把 `Int` 和 `Long` 类型传给 `fromBigIntToLiteral`， `Int` 和 `Long` 会自动转成 `BigInt`，不需要调用其他方法。

下面分别是 `UInt.Lit`、`SInt.Lit` 和 `Bool.Lit` 三个方法的实现：

```scala
// play-chisel/chiselFrontend/src/UIntFactory.scala
import chisel3.internal.firrtl.{ULit, Width}

trait UIntFactory {
  ...
  protected[chisel3] def Lit(value: BigInt, width: Width): UInt = {
    val lit = ULit(value, width)
    val result = new UInt(lit.width)
    lit.bindLitArg(result)
  }
}
```

```scala
// play-chisel/chiselFrontend/src/SIntFactory.scala

trait SIntFactory {
  ...
  protected[chisel3] def Lit(value: BigInt, width: Width): SInt = {
    val lit = SLit(value, width)
    val result = new SInt(lit.width)
    lit.bindLitArg(result)
  }
}
```

```scala
// play-chisel/chiselFrontend/src/BoolFactory.scala
import chisel3.internal.firrtl.{ULit, Width}

trait BoolFactory {
  ...
  protected[chisel3] def Lit(x: Boolean): Bool = {
    val lit = ULit(if (x) 1 else 0, Width(1))
    val result = new Bool()
    lit.bindLitArg(result)
  }
}
```

它们的构造方式都是类似的，生成对应携带数值和宽度信息的 **IR**：`ULit`、`SLit`。 `Bool.Lit` 生成的是特殊的 `ULit`，携带的数值要么是 1 要么是 0。

再根据宽度生成对应的 **Chisel** 类型： `UInt` 、`SInt` 和 `Bool`。

最后再把 **IR** 信息绑定到对应的 **Chisel** 类型。

下面是 `ULit` 和 `SLit` **IR** 的实现。

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class ULit(n: BigInt, w: Width) extends LitArg(n, w) {
  def name: String = "UInt" + width + "(\"h0" + num.toString(16) + "\")"
  def minWidth: Int = 1 max n.bitLength

  require(n >= 0, s"UInt literal ${n} is negative")
}
```

`ULit` 要求传递的数值参数要大于等于 0。例如 `ULit(13, 5.W)` 生成对应的 **FIRRTL** 字符串 "UInt<5>(h0D)"。

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class SLit(n: BigInt, w: Width) extends LitArg(n, w) {
  def name: String = {
    val unsigned = if (n < 0) (BigInt(1) << width.get) + n else n
    s"asSInt(${ULit(unsigned, width).name})"
  }
  def minWidth: Int = 1 + n.bitLength
}
```

`SLit` 采用的是二的补码表示负数（ [Two's_complement](https://en.wikipedia.org/wiki/Two%27s_complement) ）。

以 `Slit(-2, 3.W)` 为例子。二的补码有个特性，`-2`的二进制表示方式和 `2` 的二进制表示方式相加等于 `0`。 `-2 + 2 = 0`，把 `2` 移到等号右边 `-2 = 0 - 2`。0 对应的 3位二进制 `000` 需要向高位借位变成 `1000`（无符号表示的 8），减去 2 对应的二进制 `010` 得到 `110`，相当于无符号表示的 6。

所以当 `n < 0` 时，`BigInt(1)` 左移对应宽度 `<< width.get`。刚才的例子宽度为 3，1 左移 3 位就是二进制的 `1000`（十进制的8），`n` 是 `-2`，`+ n` 相当于 `8 + (-2) = 6`。验证下，`010`（十进制的2）加上 `110`（十进制的 6）等于 `1000`（十进制的8），由于这里宽度是3，截位正好 `000`（十进制的 0）。

接下来看下 `LitArg` 的实现。

```scala
// play-chisel/src/internal/firrtl/IR.scala
abstract class LitArg(val num: BigInt, widthArg: Width) extends Arg {
  private[chisel3] def forcedWidth = widthArg.known
  private[chisel3] def width: Width = if (forcedWidth) widthArg else Width(minWidth)
  override def fullName(ctx: Component): String = name
  def bindLitArg[T <: Element](elem: T): T = {
    elem.bind(ElementLitBinding(this))
    elem.setRef(this)
    elem
  }

  protected def minWidth: Int
  if (forcedWidth) {
    require(widthArg.get >= minWidth,
      s"The literal value ${num} was elaborated with a specified width of ${widthArg.get} bits, but at least ${minWidth} bits are required.")
  }
}
```

确保表示这个 LitArg 的节点有一个指向 Ensure the node representing this LitArg has a ref to it and a literal binding.

```scala
// play-chisel/chiselFrontend/src/internal/Binding.scala
import chisel3.internal.firrtl.LitArg

sealed trait UnconstrainedBinding extends TopBinding {
  def location: Option[BaseModule] = None
}

sealed trait LitBinding extends UnconstrainedBinding with ReadOnlyBinding
case class ElementLitBinding(litArg: LitArg) extends LitBinding
```

```scala
// play-chisel/chiselFrontend/src/package.scala
package object chisel3 {
  ...
  implicit class fromStringToLiteral(str: String) {
    def U: UInt = str.asUInt()
    def U(width: Width): UInt = str.asUInt(width)

    def asUInt(): UInt = {
      val bigInt = parse(str)
      UInt.Lit(bigInt, Width(bigInt.bitLength max 1))
    }
    def asUInt(width: Width): UInt = UInt.Lit(parse(str), width)

    protected def parse(n: String): BigInt = {
      val (base, num) = n.splitAt(1)
      val radix = base match {
        case "x" | "h" => 16
        case "d" => 10
        case "o" => 8
        case "b" => 2
        case _ => throwException(s"Invalid base $base"); 2
      }
      BigInt(num.filterNot(_ == '_'), radix)
    }
  }
}
```

```scala
// play-chisel/chiselFrontend/src/internal/MonoConnect.scala
private[chisel3] object MonoConnect {
  ...
  def connect(
    sink: Data,
    source: Data,
    context_mod: RawModule): Unit =
    (sink, source) match {
      ...
      case (sink_e: SInt, source_e: SInt) =>
        elemConnect(sink_e, source_e, context_mod)
     }
}
```

