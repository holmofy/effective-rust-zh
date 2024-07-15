## 条款10：熟悉标准特征

Rust 通过一组描述其行为的细粒度标准trait (参见第 2 条)，在类型系统本身中编码其类型系统的关键行为。

这些trait中的许多对于来自 C++ 的程序员来说似乎很熟悉，对应于诸如复制构造函数、析构函数、相等和赋值运算符等概念。

与在 C++ 中一样，为您自己的类型实现许多这些trait通常是一个好主意；如果您的类型执行了某些操作，但没有实现这些对应必要的trait，Rust编译器将为您提供有用的错误消息。

实现这么一大堆trait似乎令人生畏，但大多数常见trait都可以使用派生宏自动应用于用户定义的类型。这些派生宏生成该trait的“样板”实现代码（例如，对结构上的 Eq 进行逐字段比较）；这通常要求所有组成部分也实现该trait。自动生成的实现通常是您想要的，但偶尔会出现一些例外情况，后面将在每个trait部分进行讨论。

使用派生宏确实会导致如下类型定义：

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Hash)]
enum MyBooleanOption {
    Off,
    On,
}
```

针对八种不同的trait自动生成的实现。

这种细粒度的行为规范一开始可能会令人不安，但熟悉这些标准trait中最常见的trait很重要，这样才能立即理解类型定义的可用行为。

## 常见标准特征

本节讨论最常见的标准trait。以下对每个trait用一句话进行粗略摘要：

* [Clone](https://doc.rust-lang.org/std/clone/trait.Clone.html)：此类型的项可以在被要求时通过运行用户定义的代码来复制自身。
* [Copy](https://doc.rust-lang.org/std/marker/trait.Copy.html)：如果编译器逐位复制此项的内存表示（不运行任何用户定义的代码），则结果为有效的trait。
* [Default](https://doc.rust-lang.org/std/default/trait.Default.html)：可以使用合理的默认值创建此类型的新实例。
* [PartialEq](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html)：此类型的项存在部分等价关系 - 任何两个项都可以明确比较，但 `x==x` 可能并不总是正确的。
* [Eq](https://doc.rust-lang.org/std/cmp/trait.Eq.html)：此类型的项存在等价关系 - 任何两个项都可以明确比较，并且 `x==x` 始终是正确的。
* [PartialOrd](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html)：此类型的某些项可以进行比较和排序。
* [Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html)：此类型的所有项都可以进行比较和排序。
* [Hash](https://doc.rust-lang.org/std/hash/trait.Hash.html)：当被要求时，此类型的项目可以生成其内容的稳定哈希值。
* [Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html)：此类型的项目可以显示给程序员。
* [Display](https://doc.rust-lang.org/std/fmt/trait.Display.html)：此类型的项目可以显示给用户。

这些trait都可以从用户定义的类型中派生出来，但 Display 除外（此处包括它，因为它与调试重叠）。但是，有时手动实现（或不实现）是更好的选择。

以下各节将更详细地讨论这些常见trait中的每一个。

## Clone trait

表示可以通过调用 `clone()` 方法来创建项目的新副本。这大致相当于 C++ 的复制构造函数，但更明确：编译器永远不会默默地自行调用此方法（请继续阅读下一节）。

如果类型的所有字段都实现了 `Clone`，则可以为该类型派生 `Clone`。派生实现通过依次克隆其每个成员来克隆聚合类型；同样，这大致相当于 C++ 中的默认复制构造函数。这使得trait可选择加入（通过添加 `#[derive(Clone)]`），与 C++ 中的选择退出行为（`MyType(const MyType&) = delete;`）相反。

这是一个非常常见且有用的操作，因此更有趣的是研究您不应该或不能实现 Clone 的情况，或者默认派生实现不合适的情况。

* 如果项目体现了对某些资源的唯一访问（例如 RAII 类型；项目 11），或者有其他原因需要限制复制（例如，如果项目包含加密密钥材料），则不应实现 Clone。
* 如果类型的某些组件反过来不可克隆，则无法实现 Clone。示例包括以下内容：
  * 可变引用的字段（&mut T），因为借用检查器（项目 15）一次只允许一个可变引用。
  * 属于前一类的标准库类型，例如 MutexGuard（体现唯一访问）或 Mutex（限制复制以确保线程安全）。
* 如果项目中的任何内容无法通过（递归）逐字段复制捕获，或者项目生命周期有额外的簿记，则应手动实现 Clone。例如，考虑一种在运行时跟踪现有项目数量的类型，用于度量目的；手动实现 Clone 可以确保计数器保持准确。

## Copy

Copy trait有一个简单的声明：

```rust
pub trait Copy: Clone { }
```

此trait中没有方法，这意味着它是一个标记特征（如第 2 项所述）：它用于指示类型系统中未直接表达的某些类型的约束。

在 Copy 的情况下，此标记的含义是保存项目的内存的逐位复制会给出正确的新项目。实际上，此特征是一个标记，表示类型是“普通旧数据”（POD）类型。

这也意味着 Clone 特征界限可能有点令人困惑：尽管 Copy 类型必须实现 Clone，但当复制类型的实例时，不会调用 clone() 方法 - 编译器会在不涉及用户定义代码的情况下构建新项目。

与用户定义的标记特征（第 2 项）相比，Copy 对编译器具有特殊意义（std::marker 中的其他几个标记特征也是如此），除了可用于特征界限之外 - 它将编译器从移动语义转变为复制语义。

对于赋值运算符，使用移动语义，右手给予什么，左手就拿走什么：

```rust
#[derive(Debug, Clone)]
struct KeyId(u32);

let k = KeyId(42);
let k2 = k; // value moves out of k into k2
println!("k = {k:?}");
```
```rust
error[E0382]: borrow of moved value: `k`
  --> src/main.rs:60:23
   |
58 |         let k = KeyId(42);
   |             - move occurs because `k` has type `main::KeyId`, which does
   |               not implement the `Copy` trait
59 |         let k2 = k; // value moves out of k into k2
   |                  - value moved here
60 |         println!("k = {k:?}");
   |                       ^^^^^ value borrowed here after move
   |
   = note: this error originates in the macro `$crate::format_args_nl`
help: consider cloning the value if the performance cost is acceptable
   |
59 |         let k2 = k.clone(); // value moves out of k into k2
   |                   ++++++++
```
对于复制语义，原始项目存在于：
```rust
#[derive(Debug, Clone, Copy)]
struct KeyId(u32);

let k = KeyId(42);
let k2 = k; // value bitwise copied from k to k2
println!("k = {k:?}");
```

这使得 Copy 成为需要注意的最重要的trait之一：它从根本上改变了赋值的行为——包括方法调用的参数。

在这方面，它再次与 C++ 的复制构造函数重叠，但值得强调一个关键区别：在 Rust 中，没有办法让编译器默默调用用户定义的代码——它要么是显式的（对 .clone() 的调用），要么不是用户定义的（按位复制）。

因为 Copy 具有 Clone 特征绑定，所以可以 .clone() 任何可复制的项目。然而，这不是一个好主意：按位复制总是比调用特征方法更快。Clippy（第 29 项）会就此发出警告：

```rust
let k3 = k.clone();
```
```rust
warning: using `clone` on type `KeyId` which implements the `Copy` trait
  --> src/main.rs:79:14
   |
79 |     let k3 = k.clone();
   |              ^^^^^^^^^ help: try removing the `clone` call: `k`
   |
```

与 Clone 一样，值得探讨何时应该或不应该实现 Copy：

* 显而易见：如果按位复制不会产生有效项，则不要实现 Copy。如果 Clone 需要手动实现而不是自动派生实现，则很可能是这种情况。
* 如果您的类型很大，实现 Copy 可能不是一个好主意。Copy 的基本承诺是按位复制是有效的；但是，这通常与复制速度快的假设相伴而生。如果不是这种情况，跳过 Copy 可防止意外的慢速复制。
* 如果您类型的某些组件反过来不可复制，则您无法实现 Copy。
* 如果您类型的所有组件都是可复制的，那么通常值得派生 Copy。编译器有一个默认关闭的 lint missing_copy_implementations，它指出了这种情况的机会。
