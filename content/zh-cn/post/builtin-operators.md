---
title: "第四章：内置运算"
date: 2019-12-11T23:01:05+08:00
draft: true
---

Chisel 定义了硬件运算操作：

# 相等性比较

`SInt`、`UInt` 和 `Bool` 类型有效，返回值是 `Bool` 类型。

| 运算                | 说明   |
|---------------------|--------|
| `val equ = x === y` | 相等   |
| `val neq = x =/= y` | 不相等 |

## 目标

```scala
class Equality_Comparison extends RawModule {
  val x = IO(Input(UInt(5.W)))
  val y = IO(Input(UInt(4.W)))
  val equ = IO(Output(Bool()))
  val neq = IO(Output(Bool()))

  equ := x === y
  neq := x =/= y
}

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Equality_Comparison))
  ...
  val file = new File("Equality_Comparison.fir")
  ...
}
```

```firrtl
circuit Equality_Comparison : 
  module Equality_Comparison : 
    input x : UInt<5>
    input y : UInt<4>
    output equ : UInt<1>
    output neq : UInt<1>
    
    node _T = eq(x, y)
    equ <= _T
    node _T_1 = neq(x, y)
    neq <= _T_1
```

```verilog
module Equality_Comparison(
  input  [4:0] x,
  input  [3:0] y,
  output       equ,
  output       neq
);
  wire [4:0] _GEN_0;
  assign _GEN_0 = {{1'd0}, y};
  assign equ = x == _GEN_0;
  assign neq = x != _GEN_0;
endmodule
```

## 实现

比较是否相等的运算只在 `UInt`、 `SInt` 和 `Bool` 各自同类型之间有效，`Bool` 是位宽为 1 的特殊 `UInt`，只需在 `UInt` 和 `SInt` 里定义相当运算 `def ===` 和不相等运算 `def =/=`。

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

sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  final def =/= (that: UInt): Bool = compop(NotEqualOp, that)
  final def === (that: UInt): Bool = compop(EqualOp, that)
}

sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  final def =/= (that: SInt): Bool = compop(NotEqualOp, that)
  final def === (that: SInt): Bool = compop(EqualOp, that)
}
```

`UInt` 和 `SInt` 的方法除了参数的类型不同，剩下都是一样的，都调用 `compop` 比较运算的方法。不相等的运算名 `NotEqualOp` 和相等运算名 `EqualOp`。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val EqualOp = PrimOp("eq")
  val NotEqualOp = PrimOp("neq")
}
```

相等运算和不相等运算对应的 **FIRRTL** 为 `eq` 和 `neq`。

执行 `mill` 命令：

```shell
mill chisel3.run
```

# 算术比较

`SInt`和 `UInt` 类型有效，返回值是 `Bool` 类型。

| 运算               | 说明       |
|--------------------|------------|
| `val gt = a > b`   | 大于       |
| `val gte = a >= b` | 大于或等于 |
| `val lt = a < b`   | 小于       |
| `val lte = a <= b` | 小于或等于 |

## 目标

```scala
class Arithmetic_Comparisons extends RawModule {
  val a = IO(Input(UInt(8.W)))
  val b = IO(Input(UInt(8.W)))
  val gt  = IO(Output(Bool()))
  val gte = IO(Output(Bool()))
  val lt  = IO(Output(Bool()))
  val lte = IO(Output(Bool()))

  gt  := a > b
  gte := a >= b
  lt  := a < b
  lte := a <= b
}

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Arithmetic_Comparisons))
  ...
  val file = new File("Arithmetic_Comparisons.fir")
  ...
}
```

```firrtl
circuit Arithmetic_Comparisons : 
  module Arithmetic_Comparisons : 
    input a : UInt<8>
    input b : UInt<8>
    output gt : UInt<1>
    output gte : UInt<1>
    output lt : UInt<1>
    output lte : UInt<1>
    
    node _T = gt(a, b)
    gt <= _T
    node _T_1 = geq(a, b)
    gte <= _T_1
    node _T_2 = lt(a, b)
    lt <= _T_2
    node _T_3 = leq(a, b)
    lte <= _T_3
```

```verilog
module Arithmetic_Comparisons(
  input  [7:0] a,
  input  [7:0] b,
  output       gt,
  output       gte,
  output       lt,
  output       lte
);
  assign gt = a > b;
  assign gte = a >= b;
  assign lt = a < b;
  assign lte = a <= b;
endmodule
```

## 实现

算术比较是属于 `Num` 类型的运算，因此要在 `trait Num` 里定义，强制它的子类必须实现这些方法。

```scala
// play-chisel/chiselFrontend/src/Num.scala
trait Num[T <: Data] {
  ...
  def <  (that: T): Bool
  def <= (that: T): Bool
  def >  (that: T): Bool
  def >= (that: T): Bool
}
```

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  override def <  (that: UInt): Bool = compop(LessOp, that)
  override def >  (that: UInt): Bool = compop(GreaterOp, that)
  override def <= (that: UInt): Bool = compop(LessEqOp, that)
  override def >= (that: UInt): Bool = compop(GreaterEqOp, that)
}

sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  override def <  (that: SInt): Bool = compop(LessOp, that)
  override def >  (that: SInt): Bool = compop(GreaterOp, that)
  override def <= (that: SInt): Bool = compop(LessEqOp, that)
  override def >= (that: SInt): Bool = compop(GreaterEqOp, that)
}
```

都是调用 `compop` 方法，只是参数类型和具体的运算名不同。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val LessOp = PrimOp("lt")
  val LessEqOp = PrimOp("leq")
  val GreaterOp = PrimOp("gt")
  val GreaterEqOp = PrimOp("geq")
}
```

小于、小于或等于、大于和大于或等于对应的 **FIRRTL** 分别是 `lt`、`leq`、`gt`、`geq`。

执行 `mill` 命令：

```shell
mill chisel3.run
```


# 算术运算

对数值类型 `SInt` 和 `UInt` 有效。

| 运算                                    | 说明                |
|-----------------------------------------|---------------------|
| `val sum = a + b or val sum = a +% b  ` | 加法 （不会位扩展） |
| `val sum = a +& b                     ` | 加法 （位扩展）     |
| `val diff = a - b or val diff = a -% b` | 减法 （不会位扩展） |
| `val diff = a -& b                    ` | 减法 (位扩展)       |
| `val prod = a * b                     ` | 乘法                |
| `val div = a / b                      ` | 除法                |
| `val mod = a % b                      ` | 模运算              |

## 目标

```scala
class Arithmetic_Operations extends RawModule {
  val a = IO(Input(UInt(8.W)))
  val b = IO(Input(UInt(8.W)))
  val sum0  = IO(Output(UInt(9.W)))
  val sum1  = IO(Output(UInt(9.W)))
  val sum2  = IO(Output(UInt(9.W)))
  val diff0 = IO(Output(UInt(9.W)))
  val diff1 = IO(Output(UInt(9.W)))
  val diff2 = IO(Output(UInt(9.W)))
  val prod  = IO(Output(UInt(16.W)))
  val div   = IO(Output(UInt(8.W)))
  val mod   = IO(Output(UInt(8.W)))

  sum0  := a + b
  sum1  := a +% b
  sum2  := a +& b
  diff0 := a - b
  diff1 := a -% b
  diff2 := a -& b
  prod  := a * b
  div   := a / b
  mod   := a % b
}

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Arithmetic_Operations))
  ...
  val file = new File("Arithmetic_Operations.fir")
  ...
}
```

## 实现

算术运算也是属于 `Num` 类型的运算，因此要在 `trait Num` 里定义，强制它的子类必须实现这些方法。

```scala
// play-chisel/chiselFrontend/src/Num.scala
trait Num[T <: Data] {
  def + (that: T): T
  def * (that: T): T
  def / (that: T): T
  def % (that: T): T
  def - (that: T): T
  ...
}
```

```scala
// play-chisel/chiselFrontend/src/Bits.scala

sealed abstract class Bits(private[chisel3] val width: Width) extends Element {
  ...
  final def tail(n: Int): UInt = {
    val w = width match {
      case KnownWidth(x) =>
        require(x >= n, s"Can't tail($n) for width $x < $n")
        Width(x - n)
      case UnknownWidth() => Width()
    }
    binop(UInt(width = w), TailOp, n)
  }
}

sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  override def + (that: UInt): UInt = this +% that
  override def - (that: UInt): UInt = this -% that
  override def * (that: UInt): UInt = binop(UInt(this.width + that.width), TimesOp, that)
  override def / (that: UInt): UInt = binop(UInt(this.width), DivideOp, that)
  override def % (that: UInt): UInt = binop(UInt(this.width), RemOp, that)

  final def * (that: SInt): SInt =  that * this

  final def +& (that: UInt): UInt = binop(UInt((this.width max that.width) + 1), AddOp, that)
  final def +% (that: UInt): UInt = (this +& that).tail(1)
  final def -& (that: UInt): UInt = (this subtractAsSInt that).asUInt
  final def -% (that: UInt): UInt = (this subtractAsSInt that).tail(1)
  
  final def zext(): SInt = pushOp(DefPrim(SInt(width + 1), ConvertOp, ref))

  private def subtractAsSInt(that: UInt): SInt =
    binop(SInt((this.width max that.width) + 1), SubOp, that)
}

sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  override def + (that: SInt): SInt = this +% that
  override def - (that: SInt): SInt = this -% that
  override def * (that: SInt): SInt = binop(SInt(this.width + that.width), TimesOp, that)
  override def / (that: SInt): SInt = binop(SInt(this.width), DivideOp, that)
  override def % (that: SInt): SInt = binop(SInt(this.width), RemOp, that)

  final def * (that: UInt): SInt = {
    val thatToSInt = that.zext()
    val result = binop(SInt(this.width + thatToSInt.width), TimesOp, thatToSInt)
    result.tail(1).asSInt
  }

  final def +& (that: SInt): SInt = binop(SInt((this.width max that.width) + 1), AddOp, that)
  final def +% (that: SInt): SInt = (this +& that).tail(1).asSInt
  final def -& (that: SInt): SInt = binop(SInt((this.width max that.width) + 1), SubOp, that)
  final def -% (that: SInt): SInt = (this -& that).tail(1).asSInt
}
```

`UInt` 和 `SInt` 大部分的算术运算方法都是调用 `compop` 方法，只是参数类型和具体的运算名不同。这里注意下不同类型进行的乘法运算、和减法运算。

`UInt` 类型乘以 `SInt` 调用的是 `SInt` 乘以 `UInt` 的方法，需要调用零扩展方法 `zext`。

`UInt` 的减法会转成 `SInt` 类型再相减。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  val AddOp = PrimOp("add")
  val SubOp = PrimOp("sub")
  val TailOp = PrimOp("tail")
  val TimesOp = PrimOp("mul")
  val DivideOp = PrimOp("div")
  val RemOp = PrimOp("rem")
  val ConvertOp = PrimOp("cvt")
  ...
}
```

这些都是对应的 **FIRRTL** 运算的名字。

## 移位运算

移位运算对 `SInt` 和 `UInt` 类型有效。

| 运算                       | 说明                                                   |
|----------------------------|--------------------------------------------------------|
| `val twoToTheX = 1.S << x` | 逻辑左移                                               |
| `val hiBits = x >> 16.U  ` | 右移 （ `UInt` 类型为逻辑右移、`SInt` 类型为算术右移） |

## 目标

```scala
class Shifts extends RawModule {
  val x         = IO(Input(UInt(3.W)))
  val y         = IO(Input(UInt(32.W)))
  val twoToTheX = IO(Output(SInt(33.W)))
  val hiBits    = IO(Output(UInt(16.W)))
  twoToTheX := 1.S << x
  hiBits    := y >> 16.U
}
object Main extends App {
  val (circuit, _) = Builder.build(Module(new Shifts))
  ...
  val file = new File("Shifts.fir")
  ...
}
```

```firrtl
circuit Shifts : 
  module Shifts : 
    input x : UInt<3>
    input y : UInt<32>
    output twoToTheX : SInt<33>
    output hiBits : UInt<16>
    
    node _T = dshl(asSInt(UInt<2>("h01")), x)
    twoToTheX <= _T
    node _T_1 = dshr(y, UInt<5>("h010"))
    hiBits <= _T_1
```

```verilog
module Shifts(
  input  [2:0]  x,
  input  [31:0] y,
  output [32:0] twoToTheX,
  output [15:0] hiBits
);
  wire [8:0] _T;
  wire [15:0] _GEN_0;
  wire [31:0] _T_1;
  assign _T = $signed(9'sh1) << x;
  assign _GEN_0 = y[31:16];
  assign _T_1 = {{16'd0}, _GEN_0};
  assign twoToTheX = {{24{_T[8]}},_T};
  assign hiBits = _T_1[15:0];
endmodule
```

## 实现

移位运算属于 `Bits` 类型的运算，因此要在 `abstract classs Num` 里定义，强制它的子类必须实现这些方法。

```scala
sealed abstract class Bits(private[chisel3] val width: Width) extends Element {
  ...
  private[chisel3] def binop[T <: Data](dest: T, op: PrimOp, other: BigInt): T = {
    requireIsHardware(this, "bits operated on")
    pushOp(DefPrim(dest, op, this.ref, ILit(other)))
  }

  def << (that: BigInt): Bits
  def << (that: Int)   : Bits
  def << (that: UInt)  : Bits
  def >> (that: BigInt): Bits
  def >> (that: Int)   : Bits
  def >> (that: UInt)  : Bits
  
  protected final def validateShiftAmount(x: Int): Int = {
    if (x < 0)
      throwException(s"Negative shift amounts are illegal (got $x)")
    x
  }
}
sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  override def << (that: Int): UInt =
    binop(UInt(this.width + that), ShiftLeftOp, validateShiftAmount(that))
  override def << (that: BigInt): UInt =
    this << castToInt(that, "Shift amount")
  override def << (that: UInt): UInt =
    binop(UInt(this.width.dynamicShiftLeft(that.width)), DynamicShiftLeftOp, that)
  override def >> (that: Int): UInt =
    binop(UInt(this.width.shiftRight(that)), ShiftRightOp, validateShiftAmount(that))
  override def >> (that: BigInt): UInt =
    this >> castToInt(that, "Shift amount")
  override def >> (that: UInt): UInt =
    binop(UInt(this.width), DynamicShiftRightOp, that)
}
sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  override def << (that: Int): SInt =
    binop(SInt(this.width + that), ShiftLeftOp, validateShiftAmount(that))
  override def << (that: BigInt): SInt =
    this << castToInt(that, "Shift amount")
  override def << (that: UInt): SInt =
    binop(SInt(this.width.dynamicShiftLeft(that.width)), DynamicShiftLeftOp, that)
  override def >> (that: Int): SInt =
    binop(SInt(this.width.shiftRight(that)), ShiftRightOp, validateShiftAmount(that))
  override def >> (that: BigInt): SInt =
    this >> castToInt(that, "Shift amount")
  override def >> (that: UInt): SInt =
    binop(SInt(this.width), DynamicShiftRightOp, that)
}
```

`UInt` 和 `SInt` 移位运算方法都是调用二元运算 `binop` 方法，只是参数类型和具体的运算名不同。

```scala
// play-chisel/src/internal/firrtl/IR.scala
sealed abstract class Width {
  def + (that: Width): Width = this.op(that, _ + _)
  def + (that: Int): Width = this.op(this, (a, b) => a + that)
  def shiftRight(that: Int): Width = this.op(this, (a, b) => 0 max (a - that))
  def dynamicShiftLeft(that: Width): Width =
    this.op(that, (a, b) => a + (1 << b) - 1)
  ...
}
```

动态移位的方法 `dynamicShiftLeft`。

```scala
// play-chisel/chiselFrontend/src/internal/Builder.scala
/** Casts BigInt to Int, issuing an error when the input isn't representable. */
private[chisel3] object castToInt {
  def apply(x: BigInt, msg: String): Int = {
    val res = x.toInt
    require(x == res, s"$msg $x is too large to be represented as Int")
    res
  }
}
```

`BigInt` 转 `Int` 类型，确保 `BigInt` 的大小能在 `Int` 表示的范围里。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  val ShiftLeftOp = PrimOp("shl")
  val ShiftRightOp = PrimOp("shr")
  val DynamicShiftLeftOp = PrimOp("dshl")
  val DynamicShiftRightOp = PrimOp("dshr")
  ...
}
```

对应的 **FIRRTL** 运算的名字。

```shell
$ mill chisel3.run
```
# 位运算

位运算对 `SInt`、`UInt` 和 `Bool` 类型有效。


| 运算                                     | 说明     |
|------------------------------------------|----------|
| `val invertedX = ~x                    ` | 按位取反 |
| `val hiBits = x & "h_ffff_0000".U      ` | 按位与   |
| `val flagsOut = flagsIn \vert  overflow` | 按位或   |
| `val flagsOut = flagsIn ^ toggle       ` | 按位异或 |

## 目标

```scala
class Bitwise_Ops extends RawModule {
  val x         = IO(Input(UInt(32.W)))
  val invertedX = IO(Output(UInt(32.W)))
  val hiBits    = IO(Output(UInt(32.W)))
  val OROut     = IO(Output(UInt(32.W)))
  val XOROut    = IO(Output(UInt(32.W)))

  invertedX := ~x 	                // Bitwise NOT
  hiBits    :=  x & "h_ffff_0000".U // Bitwise AND
  OROut     := invertedX | hiBits	  // Bitwise OR
  XOROut    := invertedX ^ hiBits   // Bitwise XOR
}

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Bitwise_Ops))
  ...
  val file = new File("Bitwise_Ops.fir")
  ...
}
```

```firrtl
circuit Bitwise_Ops : 
  module Bitwise_Ops : 
    input x : UInt<32>
    output invertedX : UInt<32>
    output hiBits : UInt<32>
    output OROut : UInt<32>
    output XOROut : UInt<32>
    
    node _T = not(x)
    invertedX <= _T
    node _T_1 = and(x, UInt<32>("h0ffff0000"))
    hiBits <= _T_1
    node _T_2 = or(invertedX, hiBits)
    OROut <= _T_2
    node _T_3 = xor(invertedX, hiBits)
    XOROut <= _T_3
```

```verilog
module Bitwise_Ops(
  input  [31:0] x,
  output [31:0] invertedX,
  output [31:0] hiBits,
  output [31:0] OROut,
  output [31:0] XOROut
);
  assign invertedX = ~ x;
  assign hiBits = x & 32'hffff0000;
  assign OROut = invertedX | hiBits;
  assign XOROut = invertedX ^ hiBits;
endmodule
```

## 实现

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed abstract class Bits(private[chisel3] val width: Width) extends Element {
  ...
  def unary_~ (): Bits
}

sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  def unary_~ (): UInt =
    unop(UInt(width = width), BitNotOp)
}
sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  def unary_~ (): SInt =
    unop(UInt(width = width), BitNotOp).asSInt
}

sealed class Bool() extends UInt(1.W) with Reset {
  ...
  override def unary_~ (): Bool =
    unop(Bool(), BitNotOp)
}
```

按位取反的方法。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val BitNotOp = PrimOp("not")
}
```

按位取反运算的 **FIRRTL** 名字。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  final def & (that: UInt): UInt =
    binop(UInt(this.width max that.width), BitAndOp, that)
}

sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  final def & (that: SInt)(): SInt =
    binop(UInt(this.width max that.width), BitAndOp, that).asSInt
}

sealed class Bool() extends UInt(1.W) with Reset {
  ...
  final def & (that: Bool)(): Bool =
    binop(Bool(), BitAndOp, that)
}

```

按位与运算的方法。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val BitAndOp = PrimOp("and")
}
```

按位与运算的 **FIRRTL** 的名字。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  final def | (that: UInt): UInt =
    binop(UInt(this.width max that.width), BitOrOp, that)
}

sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  final def | (that: SInt): SInt =
    binop(UInt(this.width max that.width), BitOrOp, that).asSInt
}

sealed class Bool() extends UInt(1.W) with Reset {
  ...
  final def | (that: Bool): Bool =
    binop(Bool(), BitOrOp, that)
}

```

按位或运算的方法。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val BitOrOp  = PrimOp("or")
}
```

按位或运算的 **FIRRTL** 名字。

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  final def ^ (that: UInt): UInt =
    binop(UInt(this.width max that.width), BitXorOp, that)
}

sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  final def ^ (that: SInt): SInt =
    binop(UInt(this.width max that.width), BitXorOp, that).asSInt
}

sealed class Bool() extends UInt(1.W) with Reset {
  ...
  final def ^ (that: Bool): Bool =
    binop(Bool(), BitXorOp, that)
}
```

按位异或运算的方法。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val BitXorOp = PrimOp("xor")
}
```

按位异或运算的 **FIRRTL** 的名字。

```shell
$ mill chisel3.run
```

# 按位 reductions

只对 `UInt` 有效，返回值类型是 `Bool`。

| 运算                  | 说明          |
|-----------------------|---------------|
| `val allSet = x.andR` | AND reduction |
| `val anySet = x.orR ` | OR reduction  |
| `val parity = x.xorR` | XOR reduction |

## 目标

```scala
class Bitwise_Reductions extends RawModule {
  val x      = IO(Input(UInt(32.W)))
  val allSet = IO(Output(Bool()))
  val anySet = IO(Output(Bool()))
  val parity = IO(Output(Bool()))

  allSet := x.andR
  anySet := x.orR
  parity := x.xorR
}
object Main extends App {
  val (circuit, _) = Builder.build(Module(new Bitwise_Reductions))
  ...
  val file = new File("Bitwise_Reductions.fir")
  ...
}
```

```firrtl
circuit Bitwise_Reductions : 
  module Bitwise_Reductions : 
    input x : UInt<32>
    output allSet : UInt<1>
    output anySet : UInt<1>
    output parity : UInt<1>
    
    node _T = eq(x, UInt<32>("h0ffffffff"))
    allSet <= _T
    node _T_1 = neq(x, UInt<1>("h00"))
    anySet <= _T_1
    node _T_2 = xorr(x)
    parity <= _T_2
```

```verilog
module Bitwise_Reductions(
  input  [31:0] x,
  output        allSet,
  output        anySet,
  output        parity
);
  assign allSet = x == 32'hffffffff;
  assign anySet = x != 32'h0;
  assign parity = ^x;
endmodule
```

## 实现

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed abstract class Bits(private[chisel3] val width: Width) extends Element {

  ...
  private[chisel3] def redop(op: PrimOp): Bool = {
    requireIsHardware(this, "bits operated on")
    pushOp(DefPrim(Bool(), op, this.ref))
  }
}

sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  final def orR(): Bool = this =/= 0.U
  final def andR(): Bool = width match {
    case KnownWidth(w)  => this === ((BigInt(1) << w) - 1).U
    case UnknownWidth() => ~this === 0.U
  }
  final def xorR(): Bool = redop(XorReduceOp)
}
```

新增了 `redop` 运算的方法。

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val XorReduceOp = PrimOp("xorr")
}
```

异或 reductions 对应 **FIRRTL** 的名字。

# 逻辑运算

只对 `Bool` 类型有效。

| 运算                                                        | 说明               |
|-------------------------------------------------------------|--------------------|
| `val sleep = !busy                       `                  | 逻辑非 NOT         |
| `val hit = tagMatch && valid             `                  | 逻辑与 AND         |
| `val stall = src1busy` <code>&#124;&#124;</code> `src2busy` | 逻辑或 OR          |
| `val out = Mux(sel, inTrue, inFalse)     `                  | 二选一的多路选择器 |

## 目标

```scala
class Logical_Operations extends RawModule {
  val sel   = IO(Input(Bool()))
  val a     = IO(Input(Bool()))
  val b     = IO(Input(Bool()))
  val sleep = IO(Output(Bool()))
  val hit   = IO(Output(Bool()))
  val stall = IO(Output(Bool()))
  val out   = IO(Output(Bool()))

  sleep := !a
  hit   := a && b
  stall := a || b
  out   := Mux(sel, a, b)
}
object Main extends App {
  val (circuit, _) = Builder.build(Module(new Logical_Operations))
  ...
  val file = new File("Logical_Operations.fir")
  ...
}
```

## 实现

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  ...
  def unary_! : Bool = this === 0.U(1.W)
}

sealed class Bool() extends UInt(1.W) with Reset {
  ...
  def || (that: Bool): Bool = this | that
  def && (that: Bool): Bool = this & that
}
```

### Mux

```shell
$ touch chiselFrontend/src/Mux.scala
```

```scala
// play-chisel/chiselFrontend/src/Mux.scala

package chisel3

import chisel3.internal._
import chisel3.internal.Builder.pushOp
import chisel3.internal.firrtl._
import chisel3.internal.firrtl.PrimOp._

object Mux {
  /** Creates a mux, whose output is one of the inputs depending on the
    * value of the condition.
    *
    * @param cond condition determining the input to choose
    * @param con the value chosen when `cond` is true
    * @param alt the value chosen when `cond` is false
    * @example
    * {{{
    * val muxOut = Mux(data_in === 3.U, 3.U(4.W), 0.U(4.W))
    * }}}
    */
  def apply[T <: Data](cond: Bool, con: T, alt: T): T = {
    requireIsHardware(cond, "mux condition")
    requireIsHardware(con, "mux true value")
    requireIsHardware(alt, "mux false value")
    val d = cloneSupertype(Seq(con, alt), "Mux")
    val conRef = con match {  // this matches chisel semantics (DontCare as object) to firrtl semantics (invalidate)
      case DontCare =>
        val dcWire = Wire(d)
        dcWire := DontCare
        dcWire.ref
      case _ => con.ref
    }
    val altRef = alt match {
      case DontCare =>
        val dcWire = Wire(d)
        dcWire := DontCare
        dcWire.ref
      case _ => alt.ref
    }
    pushOp(DefPrim(d, MultiplexOp, cond.ref, conRef, altRef))
  }
}
```

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  ...
  val MultiplexOp = PrimOp("mux")
}
```

```scala
// play-chisel/chiselFrontend/src/package.scala
package object chisel3 {
  ...
  val DontCare = chisel3.internal.InternalDontCare
}
```

```scala
// play-chisel/chiselFrontend/src/Data.scala
package internal {
    /** RHS (source) for Invalidate API.
    * Causes connection logic to emit a DefInvalid when connected to an output port (or wire).
    */
  private[chisel3] object InternalDontCare extends Element {
    // This object should be initialized before we execute any user code that refers to it,
    //  otherwise this "Chisel" object will end up on the UserModule's id list.
    // We make it private to chisel3 so it has to be accessed through the package object.

    private[chisel3] override val width: Width = UnknownWidth()

    bind(DontCareBinding(), SpecifiedDirection.Output)
    override def cloneType: this.type = DontCare
    override def toString: String = "DontCare()"

    def asUInt(): UInt = {
      throwException("DontCare does not have a UInt representation")
      0.U
    }
    // DontCare's only match themselves.
    private[chisel3] def typeEquivalent(that: Data): Boolean = that == DontCare
  }
}
```

```scala
// play-chisel/chiselFrontend/src/Data.scala
private[chisel3] object cloneSupertype {
  def apply[T <: Data](elts: Seq[T], createdType: String): T = {
    require(!elts.isEmpty, s"can't create $createdType with no inputs")

    val filteredElts = elts.filter(_ != DontCare)
    require(!filteredElts.isEmpty, s"can't create $createdType with only DontCare inputs")

    if (filteredElts.head.isInstanceOf[Bits]) {
      val model: T = filteredElts reduce { (elt1: T, elt2: T) => ((elt1, elt2) match {
        case (elt1: Bool, elt2: Bool) => elt1
        case (elt1: Bool, elt2: UInt) => elt2
        case (elt1: UInt, elt2: Bool) => elt1
        case (elt1: UInt, elt2: UInt) =>
          if (elt1.width == (elt1.width max elt2.width)) elt1 else elt2
        case (elt1: SInt, elt2: SInt) => if (elt1.width == (elt1.width max elt2.width)) elt1 else elt2
        case (elt1, elt2) =>
          throw new AssertionError(
            s"can't create $createdType with heterogeneous types ${elt1.getClass} and ${elt2.getClass}")
      }).asInstanceOf[T] }
      model.cloneTypeFull
    }
    else {
      for (elt <- filteredElts.tail) {
        require(elt.getClass == filteredElts.head.getClass,
          s"can't create $createdType with heterogeneous types ${filteredElts.head.getClass} and ${elt.getClass}")
        require(elt typeEquivalent filteredElts.head,
          s"can't create $createdType with non-equivalent types ${filteredElts.head} and ${elt}")
      }
      filteredElts.head.cloneTypeFull
    }
  }
}
```

```scala
// play-chisel/chiselFrontend/src/internal/Binding.scala
// A DontCare element has a specific Binding, somewhat like a literal.
// It is a source (RHS). It may only be connected/applied to sinks.
case class DontCareBinding() extends UnconstrainedBinding
```

```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  ...
  /** Whether this Data has the same model ("data type") as that Data.
    * Data subtypes should overload this with checks against their own type.
    */
  private[chisel3] def typeEquivalent(that: Data): Boolean
}
```

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed class UInt private[chisel3] (width: Width) extends Bits(width) with Num[UInt] {
  private[chisel3] override def typeEquivalent(that: Data): Boolean =
    that.isInstanceOf[UInt] && this.width == that.width
  ...
}

sealed class SInt private[chisel3] (width: Width) extends Bits(width) with Num[SInt] {
  ...
  private[chisel3] override def typeEquivalent(that: Data): Boolean =
    this.getClass == that.getClass && this.width == that.width  // TODO: should this be true for unspecified widths?
}
```

###  Wire

```scala
// play-chisel/chiselFrontend/src/Data.scala
import chisel3.internal.Builder.pushCommand

trait WireFactory {
  /** Construct a [[Wire]] from a type template
    * @param t The template from which to construct this wire
    */
  def apply[T <: Data](t: T): T = {
    requireIsChiselType(t, "wire type")
    val x = t.cloneTypeFull

    // Bind each element of x to being a Wire
    x.bind(WireBinding(Builder.forcedUserModule))
    pushCommand(DefWire(x))
    x
  }
}

object Wire extends WireFactory
```

```scala
// play-chisel/chiselFrontend/src/internal/Binding.scala
case class WireBinding(enclosure: RawModule) extends ConstrainedBinding
```

```scala
// play-chisel/src/internal/firrtl/IR.scala
case class DefWire(id: Data) extends Definition
```

# 位选择操作

对 `UInt` 、`SInt` 和 `Bool` 类型有效。


| 操作                                        | 说明                                 |
|---------------------------------------------|--------------------------------------|
| `val xLSB = x(0)                          ` | 提取单个比特                         |
| `val xTopNibble = x(15, 12)               ` | 提取范围内的比特                     |
| `val usDebt = Fill(3, "hA".U)             ` | 重复多个用字符串表示的比特序列       |
| `val float = Cat(sign, exponent, mantissa)` | 连接多个比特，第一个参数是放在最左边 |

## 目标

```scala
import chisel3.util._

class Bitfield_Manipulation extends RawModule {
  val sel        = IO(Input(UInt(4.W)))
  val x          = IO(Input(UInt(16.W)))
  val dynamicSel = IO(Output(Bool()))
  val xLSB       = IO(Output(Bool()))
  val xTopNibble = IO(Output(UInt(4.W)))
  val usDebt     = IO(Output(UInt(12.W)))
  val float      = IO(Output(UInt(8.W)))

  dynamicSel := x(sel)
  xLSB       := x(0)
  xTopNibble := x(15, 12)
  usDebt     := Fill(3, "hA".U)
  float      := Cat(xLSB, xTopNibble, usDebt)
}

object Main extends App {
  val (circuit, _) = Builder.build(Module(new Bitfield_Manipulation))
  ...
  val file = new File("Bitfield_Manipulation.fir")
  ...
}
```

```firrtl
circuit Bitfield_Manipulation : 
  module Bitfield_Manipulation : 
    input sel : UInt<4>
    input x : UInt<16>
    output dynamicSel : UInt<1>
    output xLSB : UInt<1>
    output xTopNibble : UInt<4>
    output usDebt : UInt<12>
    output float : UInt<8>
    
    node _T = dshr(x, sel)
    node _T_1 = bits(_T, 0, 0)
    dynamicSel <= _T_1
    node _T_2 = bits(x, 0, 0)
    xLSB <= _T_2
    node _T_3 = bits(x, 15, 12)
    xTopNibble <= _T_3
    node _T_4 = cat(UInt<4>("h0a"), UInt<4>("h0a"))
    node _T_5 = cat(UInt<4>("h0a"), _T_4)
    usDebt <= _T_5
    node _T_6 = cat(xLSB, xTopNibble)
    node _T_7 = cat(_T_6, usDebt)
    float <= _T_7
```

```verilog
module Bitfield_Manipulation(
  input  [3:0]  sel,
  input  [15:0] x,
  output        dynamicSel,
  output        xLSB,
  output [3:0]  xTopNibble,
  output [11:0] usDebt,
  output [7:0]  float
);
  wire [15:0] _T;
  wire [16:0] _T_7;
  assign _T = x >> sel;
  assign _T_7 = {xLSB,xTopNibble,usDebt};
  assign dynamicSel = _T[0];
  assign xLSB = x[0];
  assign xTopNibble = x[15:12];
  assign usDebt = 12'haaa;
  assign float = _T_7[7:0];
endmodule
```

## 实现

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
    final def apply(x: UInt): Bool = {
    val theBits = this >> x
    theBits(0)
  }
  final def apply(x: Int, y: Int): UInt = {
    if (x < y || y < 0) {
      throwException(s"Invalid bit range ($x,$y)")
    }
    val w = x - y + 1
    requireIsHardware(this, "bits to be sliced")
    pushOp(DefPrim(UInt(Width(w)), BitsExtractOp, this.ref, ILit(x), ILit(y)))
  }
  final def apply(x: BigInt, y: BigInt): UInt =
    apply(castToInt(x, "High index"), castToInt(y, "Low index"))

}
```

```shell
$ mkdir -p src/util
$ touch src/util/Cat.scala
```

```scala
// play-chisel/src/util/Cat.scala

package chisel3.util

import chisel3._

object Cat {
  def apply[T <: Bits](a: T, r: T*): UInt = apply(a :: r.toList)
  def apply[T <: Bits](r: Seq[T]): UInt = SeqUtils.asUInt(r.reverse)
}
```

```shell
$ touch chiselFrontend/src/SeqUtils.scala
```

```scala
// play-chisel/chiselFrontend/src/SeqUtils.scala
package chisel3

private[chisel3] object SeqUtils {
  def asUInt[T <: Bits](in: Seq[T]): UInt = {
    if (in.tail.isEmpty) {
      in.head.asUInt
    } else {
      val left = asUInt(in.slice(0, in.length/2))
      val right = asUInt(in.slice(in.length/2, in.length))
      right ## left
    }
  }
}
```

```scala
// play-chisel/chiselFrontend/src/Bits.scala
sealed abstract class Bits(private[chisel3] val width: Width) extends Element {
  ...
  def ## (that: Bits): UInt = {
    val w = this.width + that.width
    pushOp(DefPrim(UInt(w), ConcatOp, this.ref, that.ref))
  }
}
```

```scala
// play-chisel/src/internal/firrtl/IR.scala
object PrimOp {
  val ConcatOp = PrimOp("cat")
  ...
}
```

```scala
// play-chisel/src/util/Bitwise.scala
package chisel3.util

import chisel3._

object Fill {
  def apply(n: Int, x: UInt): UInt = {
    n match {
      case _ if n < 0 => throw new IllegalArgumentException(s"n (=$n) must be nonnegative integer.")
      case 0 => UInt(0.W)
      case 1 => x
      case _ if x.isWidthKnown && x.getWidth == 1 =>
        Mux(x.asBool, ((BigInt(1) << n) - 1).asUInt(n.W), 0.U(n.W))
      case _ =>
        val nBits = log2Ceil(n + 1)
        val p2 = Array.ofDim[UInt](nBits)
        p2(0) = x
        for (i <- 1 until p2.length)
          p2(i) = Cat(p2(i-1), p2(i-1))
        Cat((0 until nBits).filter(i => (n & (1 << i)) != 0).map(p2(_)))
    }
  }
}
```

```scala
// play-chisel/chiselFrontend/src/Data.scala
abstract class Data extends HasId {
  ...
  final def getWidth: Int =
    if (isWidthKnown) width.get else throwException(s"Width of $this is unknown!")
  final def isWidthKnown: Boolean = width.known
}
```

```shell
$ touch play-chisel/src/util/Math.scala
```

```scala
// play-chisel/src/util/Math.scala
package chisel3.util

object log2Ceil {
  def apply(in: BigInt): Int = {
    require(in > 0)
    (in-1).bitLength
  }
  def apply(in: Int): Int = apply(BigInt(in))
}
```

```scala
// play-chisel/chiselFrontend/src/Bits.scala
private[chisel3] sealed trait ToBoolable {
  def asBool: Bool
}

sealed abstract class Bits(private[chisel3] val width: Width) extends Element with ToBoolable {
  ...
  final def asBool(): Bool = {
    width match {
      case KnownWidth(1) => this(0)
      case _ => throwException(s"can't covert ${this.getClass.getSimpleName}$width to Bool")
    }
  }
}
```
