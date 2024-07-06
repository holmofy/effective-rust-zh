## 条款1：用类型系统去表示你的数据结构

> “谁叫他们是程序员而不是打字员”—— [@thingskatedid](https://twitter.com/thingskatedid/status/1400213496785108997)

本条款快速介绍了 Rust 的类型系统，从编译器提供的基本类型开始，然后介绍将值组合成数据结构的各种方式。

Rust的枚举类型随后扮演了主角。虽然基本版本与其他语言提供的枚举相同，但将枚举变量与数据字段组合的能力可以增强灵活性和表现力。

### 基本类型

Rust类型系统的基础知识对于来自其他静态类型编程语言（例如 C++、Go 或 Java）的人来说非常熟悉。有一批具有特定大小的整数类型，包括有符号 ([i8](https://doc.rust-lang.org/std/primitive.i8.html), [i16](https://doc.rust-lang.org/std/primitive.i16.html), [i32](https://doc.rust-lang.org/std/primitive.i32.html), [i64](https://doc.rust-lang.org/std/primitive.i64.html), [i128](https://doc.rust-lang.org/std/primitive.i128.html)) 和无符号 ([u8](https://doc.rust-lang.org/std/primitive.u8.html), [u16](https://doc.rust-lang.org/std/primitive.u16.html), [u32](https://doc.rust-lang.org/std/primitive.u32.html), [u64](https://doc.rust-lang.org/std/primitive.u64.html), [u128](https://doc.rust-lang.org/std/primitive.u128.html)) 。

还有两种整数类型，其大小与目标系统上的指针大小匹配：有符号的isize和无符号的usize。但是，您不会用 Rust 在指针和整数之间进行太多转换，因此大小等价并不重要。但是，标准集合将其大小作为 usize（来自 .len()）返回，因此集合索引意味着 usize 值非常常见——从容量角度来看，这显然是没有问题的，因为内存集合中的项目不能多于系统上的内存地址。

整数类型确实给了我们第一个提示，即 Rust 是一个比 C++ 更严格的世界。在 Rust 中，尝试将较大的整数类型 (i32) 放入较小的整数类型 (i16) 会产生编译时错误：

```rust
let x: i32 = 42;
let y: i16 = x;
```

```rust
error[E0308]: mismatched types
  --> src/main.rs:18:18
   |
18 |     let y: i16 = x;
   |            ---   ^ expected `i16`, found `i32`
   |            |
   |            expected due to this
   |
help: you can convert an `i32` to an `i16` and panic if the converted value
      doesn't fit
   |
18 |     let y: i16 = x.try_into().unwrap();
   |                   ++++++++++++++++++++
```
这让人放心：当程序员做有风险的操作时，Rust不会坐视不理。虽然我们可以看到，这种特定转换中涉及的值是没问题的，但编译器必须考虑到转换不顺利的可能性：

```rust
let x: i32 = 66_000;
let y: i16 = x; // What would this value be?
```

错误输出也给出了一个早期迹象，表明虽然 Rust 有更严格的规则，但它也有有用的编译器消息，指出如何遵守规则。建议的解决方案提出了一个问题，即如何处理转换必须改变值以适应的情况，我们稍后将更多地讨论错误处理（第 4 条）和使用 panic！（第 18 条）。

Rust 还不允许一些看似“安全”的事情，例如将较小整数类型的值放入较大的整数类型中：

```rust
let x = 42i32; // Integer literal with type suffix
let y: i64 = x;
```

```rust
error[E0308]: mismatched types
  --> src/main.rs:36:18
   |
36 |     let y: i64 = x;
   |            ---   ^ expected `i64`, found `i32`
   |            |
   |            expected due to this
   |
help: you can convert an `i32` to an `i64`
   |
36 |     let y: i64 = x.into();
   |                   +++++++
```

在这里，建议的解决方案不会引发错误处理的担忧，但转换仍然需要明确。我们将在后面更详细地讨论类型转换（第 5 条）。

继续使用不足为奇的原始类型，Rust 有 [bool](https://doc.rust-lang.org/std/primitive.bool.html) 类型、浮点类型（[f32](https://doc.rust-lang.org/std/primitive.f32.html)、[f64](https://doc.rust-lang.org/std/primitive.f64.html)）和 [unit 类型](https://en.wikipedia.org/wiki/Unit_type) ()（类似 C 的 void）。

更有趣的是 char 字符类型，它保存一个 Unicode 值（类似于 Go 的 rune 类型）。虽然它在内部存储为四个字节，但同样没有与 32 位整数的隐式转换。

类型系统中的这种精度迫使您明确说明您想要表达的内容 - u32 值不同于 char，而 char 又不同于 UTF-8 字节序列，而 UTF-8 字节序列又不同于任意字节序列，而具体说明您指的是哪个则取决于您。Joel Spolsky 的[著名博客文章](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)可以帮助您了解您需要哪个。

当然，有一些辅助方法允许您在这些不同类型之间进行转换，但它们的签名会迫使您处理（或明确忽略）失败的可能性。例如，Unicode 代码点始终可以用 32 位表示，因此允许将 'a' 表示为 u32，但另一个方向则比较棘手（因为有些 u32 值不是有效的 Unicode 代码点）：

* char::from_u32：返回一个 Option<char>，强制调用者处理失败情况。
* char::from_u32_unchecked：假设有效性，但如果该假设不成立，则有可能导致未定义的行为。因此，该函数被标记为不安全，从而迫使调用者也使用不安全（第 16 条）。

### 聚合类型

说到聚合类型，Rust有多种组合相关值的方法。其中大多数与其他语言中可用的聚合机制相似：

* 数组：保存单一类型的多个实例，其中实例数在编译时已知。例如，`[u32; 4]` 是一行中的四个4字节整数。
* 元组：保存多个异构类型的实例，其中元素数及其类型在编译时已知，例如 `(WidgetOffset、WidgetSize、WidgetColor)`。如果元组中的类型没有区别（例如 `(i32、i32、&'static str、bool)`），最好为每个元素命名并使用结构。
* 结构体：还保存编译时已知的异构类型的实例，但允许通过名称引用整体类型和各个字段。
Rust 还包括元组结构体，它是结构体和元组的混合体：整体类型有名称，但没有各个字段的名称 - 它们用数字来引用：s.0、s.1，等等：

```rust
/// Struct with two unnamed fields.
struct TextMatch(usize, String);

// Construct by providing the contents in order.
let m = TextMatch(12, "needle".to_owned());

// Access by field number.
assert_eq!(m.0, 12);
```

### 枚举

让我们看看 Rust 类型系统中的瑰宝——枚举。枚举的基本形式很难让人兴奋。与其他语言一样，枚举允许您指定一组互斥的值，可能还附加一个数值：

```rust
enum HttpResultCode {
    Ok = 200,
    NotFound = 404,
    Teapot = 418,
}

let code = HttpResultCode::NotFound;
assert_eq!(code as i32, 404);
```

由于每个枚举定义都会创建一个不同的类型，因此可以使用它来提高采用 bool 参数的函数的可读性和可维护性。而不是：

```rust
print_page(/* both_sides= */ true, /* color= */ false);
```

使用一对枚举的版本：

```rust
pub enum Sides {
    Both,
    Single,
}

pub enum Output {
    BlackAndWhite,
    Color,
}

pub fn print_page(sides: Sides, color: Output) {
    // ...
}
```

在调用时更加类型安全并且更易于阅读：

```rust
print_page(Sides::Both, Output::BlackAndWhite);
```

与 bool 版本不同，如果库用户意外地颠倒了参数的顺序，编译器会立即报错：

```rust
error[E0308]: arguments to this function are incorrect
   --> src/main.rs:104:9
    |
104 | print_page(Output::BlackAndWhite, Sides::Single);
    | ^^^^^^^^^^ ---------------------  ------------- expected `enums::Output`,
    |            |                                    found `enums::Sides`
    |            |
    |            expected `enums::Sides`, found `enums::Output`
    |
note: function defined here
   --> src/main.rs:145:12
    |
145 |     pub fn print_page(sides: Sides, color: Output) {
    |            ^^^^^^^^^^ ------------  -------------
help: swap these arguments
    |
104 | print_page(Sides::Single, Output::BlackAndWhite);
    |             ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

使用 newtype 模式（参见第 6 条）包装 bool 也能实现类型安全性和可维护性；如果语义始终为布尔值，则通常最好使用 newtype 模式，如果将来有可能出现新的替代方案（例如 Sides::BothAlternateOrientation），则最好使用枚举。

Rust 枚举的类型安全性通过 match 表达式继续：

```rust
let msg = match code {
    HttpResultCode::Ok => "Ok",
    HttpResultCode::NotFound => "Not found",
    // forgot to deal with the all-important "I'm a teapot" code
};
```

```rust
error[E0004]: non-exhaustive patterns: `HttpResultCode::Teapot` not covered
  --> src/main.rs:44:21
   |
44 |     let msg = match code {
   |                     ^^^^ pattern `HttpResultCode::Teapot` not covered
   |
note: `HttpResultCode` defined here
  --> src/main.rs:10:5
   |
7  | enum HttpResultCode {
   |      --------------
...
10 |     Teapot = 418,
   |     ^^^^^^ not covered
   = note: the matched value is of type `HttpResultCode`
help: ensure that all possible cases are being handled by adding a match arm
      with a wildcard pattern or an explicit pattern as shown
   |
46 ~         HttpResultCode::NotFound => "Not found",
47 ~         HttpResultCode::Teapot => todo!(),
   |
```

编译器强制程序员考虑枚举所表示的所有可能性，即使结果只是添加一个默认的 arm _ => {}。（请注意，现代 C++ 编译器也可以并且确实会警告枚举缺少 switch 分支。）

### 带字段的枚举

Rust 枚举特性的真正威力在于，每个变量都可以拥有随附的数据，使其成为充当代数数据类型 (ADT) 的聚合类型。主流语言的程序员对此不太熟悉；用 C/C++ 术语来说，它就像枚举与联合的组合——只是类型安全的。

这意味着程序数据结构的不变量可以编码到 Rust 的类型系统中；不符合这些不变量的状态甚至不会编译。设计良好的枚举可以让人类和编译器清楚地了解创建者的意图：

```rust
use std::collections::{HashMap, HashSet};

pub enum SchedulerState {
    Inert,
    Pending(HashSet<Job>),
    Running(HashMap<CpuId, Vec<Job>>),
}
```

仅从类型定义来看，可以合理地猜测作业会排队等待，直到调度程序完全激活，此时它们会被分配到某个 CPU 池中。

这突出了本条目的中心主题，即使用 Rust 的类型系统来表达与软件设计相关的概念。

当这种情况没有发生时，一个明显的迹象是注释解释了某些字段或参数何时有效：

```rust
pub struct DisplayProps {
    pub x: u32,
    pub y: u32,
    pub monochrome: bool,
    // `fg_color` must be (0, 0, 0) if `monochrome` is true.
    pub fg_color: RgbColor,
}
```

这是用枚举保存数据进行替换的最佳候选：

```rust
pub enum Color {
    Monochrome,
    Foreground(RgbColor),
}

pub struct DisplayProps {
    pub x: u32,
    pub y: u32,
    pub color: Color,
}
```

这个小例子说明了一条关键建议：让无效状态无法在类型中表达。仅支持有效值组合的类型意味着整个错误类别都会被编译器拒绝，从而产生更小、更安全的代码。

### 无处不在的枚举类型

回到枚举的强大功能，有两个概念非常常见，以至于 Rust 的标准库包含内置枚举类型来表达它们；这些类型在 Rust 代码中无处不在。

#### Option<T>

第一个概念是 `Option`：要么存在特定类型的值（`Some(T)`），要么不存在（`None`）。始终使用 `Option` 来表示可能不存在的值；切勿回退到使用标记值（-1、`nullptr` 等）来尝试在带内表达相同的概念。

不过，有一个微妙的点需要考虑。如果您处理的是事物集合，则需要确定集合中没有事物是否等同于没有集合。在大多数情况下，不会出现这种区别，您可以继续使用（例如）`Vec<Thing>`：计数为零意味着没有事物。

但是，肯定还有其他罕见的情况需要使用 `Option<Vec<Thing>>` 来区分这两种情况 - 例如，加密系统可能需要区分“单独传输的有效载荷”和“提供的空有效载荷”。 （这与 SQL 中关于列的 NULL 标记的争论有关。）

同样，对于可能不存在的字符串，最佳选择是什么？“”或 None 是否更适合表示值不存在？两种方式都可以，但 `Option<String>` 清楚地传达了该值可能不存在的可能性。

#### Result<T, E>

第二个常见概念来自错误处理：如果函数失败，应该如何报告该失败？从历史上看，使用特殊的标记值（例如，Linux 系统调用的 `-errno` 返回值）或全局变量（POSIX 系统的 `errno`）。最近，支持函数返回多个或元组返回值的语言（例如 Go）可能有一个惯例，即返回 `(result, error)` 对，假设当`error`非“零”时，`result`存在一些合适的“零”值。

在 Rust 中，有一个`enum`专门用于此目的：始终将可能失败的操作的结果编码为 `Result<T, E>`。`T` 类型保存成功的结果（在 Ok 变体中），`E` 类型保存失败时的错误详细信息（在 `Err` 变体中）。

使用标准类型使设计意图清晰。它还允许使用标准转换（第 3 项）和错误处理（第 4 项），这反过来又使得使用 `?` 运算符简化错误处理成为可能。