## 条款10：熟悉标准特征

Rust 通过一组描述其行为的细粒度标准trait (参见第 2 条)，在类型系统本身中编码其类型系统的关键行为。

这些trait中的许多对于来自 C++ 的程序员来说似乎很熟悉，对应于诸如复制构造函数、析构函数、相等和赋值运算符等概念。

与在 C++ 中一样，为您自己的类型实现许多这些trait通常是一个好主意；如果您的类型执行了某些操作，但没有实现这些对应必要的trait，Rust编译器将为您提供有用的错误消息。

实现这么一大堆trait似乎令人生畏，但大多数常见trait都可以使用派生宏自动应用于用户定义的类型。这些派生宏生成该trait的“样板”实现代码（例如，对结构体上的 `Eq` 进行逐字段比较）；这通常要求所有组成部分也实现该trait。自动生成的实现通常是您想要的，但偶尔会出现一些例外情况，后面将在每个trait部分进行讨论。

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

* [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html)：此类型的项可以在被要求时通过运行用户定义的代码来复制自身。
* [`Copy`](https://doc.rust-lang.org/std/marker/trait.Copy.html)：如果编译器逐位复制此项的内存表示（不运行任何用户定义的代码），则结果为有效的trait。
* [`Default`](https://doc.rust-lang.org/std/default/trait.Default.html)：可以使用合理的默认值创建此类型的新实例。
* [`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html)：此类型的项存在部分等价关系 - 任何两个项都可以明确比较，但 `x==x` 可能并不总是正确的。
* [`Eq`](https://doc.rust-lang.org/std/cmp/trait.Eq.html)：此类型的项存在等价关系 - 任何两个项都可以明确比较，并且 `x==x` 始终是正确的。
* [`PartialOrd`](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html)：此类型的某些项可以进行比较和排序。
* [`Ord`](https://doc.rust-lang.org/std/cmp/trait.Ord.html)：此类型的所有项都可以进行比较和排序。
* [`Hash`](https://doc.rust-lang.org/std/hash/trait.Hash.html)：当被要求时，此类型的项目可以生成其内容的稳定哈希值。
* [`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html)：此类型的项目可以显示给程序员。
* [`Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html)：此类型的项目可以显示给用户。

这些trait都可以从用户定义的类型中派生出来，但 `Display` 除外（此处包括它，因为它与调试重叠）。但是，有时手动实现（或不实现）是更好的选择。

以下各节将更详细地讨论这些常见trait中的每一个。

## Clone trait

表示可以通过调用 `clone()` 方法来创建项目的新副本。这大致相当于 C++ 的复制构造函数，但更明确：编译器永远不会默默地自行调用此方法（请继续阅读下一节）。

如果类型的所有字段都实现了 `Clone`，则可以为该类型派生 `Clone`。派生实现通过依次克隆其每个成员来克隆聚合类型；同样，这大致相当于 C++ 中的默认复制构造函数。这使得trait可选择加入（通过添加 `#[derive(Clone)]`），与 C++ 中的选择退出行为（`MyType(const MyType&) = delete;`）相反。

这是一个非常常见且有用的操作，因此更有趣的是研究您不应该或不能实现 `Clone` 的情况，或者默认派生实现不合适的情况。

* 如果项目体现了对某些资源的唯一访问（例如 RAII 类型；项目 11），或者有其他原因需要限制复制（例如，如果项目包含加密密钥材料），则不应实现 `Clone`。
* 如果类型的某些组件反过来不可克隆，则无法实现 `Clone`。示例包括以下内容：
  * 可变引用的字段（`&mut T`），因为借用检查器（项目 15）一次只允许一个可变引用。
  * 属于前一类的标准库类型，例如 `MutexGuard`（体现唯一访问）或 `Mutex`（限制复制以确保线程安全）。
* 如果项目中的任何内容无法通过（递归）逐字段复制捕获，或者项目生命周期有额外的簿记，则应手动实现 `Clone`。例如，考虑一种在运行时跟踪现有项目数量的类型，用于度量目的；手动实现 `Clone` 可以确保计数器保持准确。

## Copy

`Copy` trait有一个简单的声明：

```rust
pub trait Copy: Clone { }
```

此trait中没有方法，这意味着它是一个标记特征（如第 2 项所述）：它用于指示类型系统中未直接表达的某些类型的约束。

在 `Copy` 的情况下，此标记的含义是保存项目的内存的逐位复制会给出正确的新项目。实际上，此特征是一个标记，表示类型是“普通旧数据”（POD）类型。

这也意味着 `Clone` 特征界限可能有点令人困惑：尽管 `Copy` 类型必须实现 `Clone`，但当复制类型的实例时，不会调用 `clone()` 方法 - 编译器会在不涉及用户定义代码的情况下构建新项目。

与用户定义的标记特征（第 2 项）相比，`Copy` 对编译器具有特殊意义（`std::marker` 中的其他几个标记特征也是如此），除了可用于特征界限之外 - 它将编译器从移动语义转变为复制语义。

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

这使得 `Copy` 成为需要注意的最重要的trait之一：它从根本上改变了赋值的行为——包括方法调用的参数。

在这方面，它再次与 C++ 的复制构造函数重叠，但值得强调一个关键区别：在 Rust 中，没有办法让编译器默默调用用户定义的代码——它要么是显式的（对 `.clone()` 的调用），要么不是用户定义的（按位复制）。

因为 `Copy` 具有 `Clone` 特征绑定，所以可以 `.clone()` 任何可复制的项目。然而，这不是一个好主意：按位复制总是比调用特征方法更快。Clippy（第 29 项）会就此发出警告：

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

与 `Clone` 一样，值得探讨何时应该或不应该实现 `Copy`：

* 显而易见：如果按位复制不会产生有效项，则不要实现 `Copy`。如果 `Clone` 需要手动实现而不是自动派生实现，则很可能是这种情况。
* 如果您的类型很大，实现 `Copy` 可能不是一个好主意。`Copy` 的基本承诺是按位复制是有效的；但是，这通常与复制速度快的假设相伴而生。如果不是这种情况，跳过 `Copy` 可防止意外的慢速复制。
* 如果您类型的某些组件反过来不可复制，则您无法实现 `Copy`。
* 如果您类型的所有组件都是可复制的，那么通常值得派生 `Copy`。编译器有一个默认关闭的 lint `missing_copy_implementations`，它指出了这种情况的机会。

## Default
`Default` 特征通过 `default()` 方法定义默认构造函数。此特征可从用户定义类型派生，前提是所有涉及的子类型都有自己的 `Default` 实现；如果没有，则必须手动实现该特征。继续与 C++ 进行比较，请注意必须显式触发默认构造函数 — 编译器不会自动创建默认构造函数。

`Default` 特征也可从枚举类型派生，只要有一个 `#[default]` 属性来提示编译器哪个变体是默认的：

```rust
#[derive(Default)]
enum IceCreamFlavor {
    Chocolate,
    Strawberry,
    #[default]
    Vanilla,
}
```

`Default` 特征最有用的方面是它与结构更新语法的结合。对于任何未明确初始化的字段，此语法允许通过从同一结构的现有实例复制或移动其内容来初始化结构字段。要复制的模板在初始化结束时给出，在 `..` 之后，`Default` 特征提供了一个理想的模板供使用：

```rust
#[derive(Default)]
struct Color {
    red: u8,
    green: u8,
    blue: u8,
    alpha: u8,
}

let c = Color {
    red: 128,
    ..Default::default()
};
```

这使得初始化具有大量字段的结构变得容易得多，其中只有一些字段具有非默认值。（构建器模式，第 7 项，也可能适用于这些情况。）

## PartialEq 和 Eq
`PartialEq` 和 `Eq` 特征允许您为用户定义类型定义相等性。这些特征具有特殊意义，因为如果它们存在，编译器将自动使用它们进行相等性 (`==`) 检查，类似于 C++ 中的运算符 `==`。默认的派生实现通过递归逐字段比较来执行此操作。

`Eq` 版本只是 `PartialEq` 的标记特征扩展，它添加了反身性假设：任何声称支持 `Eq` 的类型 `T` 都应确保 `x == x` 对于任何 `x：T` 都为真。

这很奇怪，立即引发了一个问题，什么时候 `x == x` 不会？这种分裂背后的主要原理与浮点数有关，特别是与特殊的“非数字”值 `NaN`（Rust 中的 `f32::NAN` / `f64::NAN`）有关。浮点规范要求没有任何东西与 `NaN` 进行比较相等，包括 `NaN` 本身；`PartialEq` 特征是这种连锁反应。

对于没有任何浮点相关特性的用户定义类型，您应该在实现 `PartialEq` 时实现 `Eq`。如果您想将类型用作 `HashMap` 中的键（以及 `Hash` 特征），则还需要完整的 `Eq` 特征。

如果您的类型包含任何不影响项目身份的字段，例如内部缓存和其他性能优化，则应该手动实现 `PartialEq`。（如果定义了 `Eq`，任何手动实现也将用于它，因为 `Eq` 只是一个没有自己的方法的标记特征。）

## PartialOrd 和 Ord

排序特征 `PartialOrd` 和 `Ord` 允许对同一类型的两个项进行比较，返回 `Less`、`Greater` 或 `Equal`。这些特征需要实现等效的相等特征（`PartialOrd` 需要 `PartialEq`；`Ord` 需要 `Eq`），并且两者必须一致（手动实现时尤其要注意这一点）。

与相等特征一样，比较特征具有特殊意义，因为编译器会自动将它们用于比较操作（`<`、`>`、`<=`、`>=`）。

由 `derive` 生成​​的默认实现按定义顺序按字典顺序比较字段（或枚举变量），因此如果这不正确，您需要手动实现这些特征（或重新排序字段）。

与 `PartialEq` 不同，`PartialOrd` 特征确实对应于各种实际情况。例如，它可用于表达集合之间的子集关系： `{1, 2}` 是 `{1, 2, 4}` 的子集，但 `{1, 3}` 不是 `{2, 4}` 的子集，反之亦然。

但是，即使偏序确实准确地模拟了类型的行为，也要注意不要只实现 `PartialOrd` 而不实现 `Ord`（这种情况很少见，与第 2 项中在类型系统中编码行为的建议相矛盾）——它可能会导致令人惊讶的结果：

```rust
// Inherit the `PartialOrd` behavior from `f32`.
#[derive(PartialOrd, PartialEq)]
struct Oddity(f32);

// Input data with NaN values is likely to give unexpected results.
let x = Oddity(f32::NAN);
let y = Oddity(f32::NAN);

// A self-comparison looks like it should always be true, but it may not be.
if x <= x {
    println!("This line doesn't get executed!");
}

// Programmers are also unlikely to write code that covers all possible
// comparison arms; if the types involved implemented `Ord`, then the
// second two arms could be combined.
if x <= y {
    println!("y is bigger"); // Not hit.
} else if y < x {
    println!("x is bigger"); // Not hit.
} else {
    println!("Neither is bigger");
}
```

## Hash
`Hash` 特征用于生成单个值，该值对于不同的项目而言很可能不同。此哈希值用作基于哈希桶的数据结构（如 `HashMap` 和 `HashSet`）的基础；因此，这些数据结构中的键的类型必须实现 `Hash`（和 `Eq`）。

反过来说，“相同”的项目（根据 `Eq`）始终生成相同的哈希值至关重要：如果 `x == y`（通过 `Eq`），则 `hash(x) == hash(y)` 必须始终为真。如果您有手动 `Eq` 实现，请检查是否还需要手动实现 `Hash` 以满足此要求。

## Debug 和 Display
`Debug` 和 `Display` 特征允许类型指定应如何将其包含在输出中，无论是用于正常目的（`{}` 格式参数）还是调试目的（`{:?}` 格式参数），大致类似于 C++ 中 `iostream` 的运算符`<<` 重载。

不过，这两个特征的意图之间的差异超出了需要哪种格式说明符：

* `Debug` 可以自动派生，`Display` 只能手动实现。
* `Debug` 输出的布局可能会在不同的 Rust 版本之间发生变化。如果输出将被其他代码解析，请使用 `Display`。
* `Debug` 面向程序员；`Display` 面向用户。一个有助于此的思想实验是考虑如果程序本地化为作者不会说的语言会发生什么——如果内容应该翻译，`Display` 是合适的，否则 `Debug`。
作为一般规则，除非您的类型包含敏感信息（个人详细信息、加密材料等），否则请为您的类型添加自动生成的 Debug 实现。为了使此建议更容易遵循，Rust 编译器包含一个 `missing_debug_implementations` lint，它指出没有 Debug 的类型。默认情况下，此 lint 是禁用的，但可以通过以下任一方式为您的代码启用：

```rust
#![warn(missing_debug_implementations)]
```
```rust
#![deny(missing_debug_implementations)]
```

如果自动生成的 `Debug` 实现会发出大量详细信息，那么包含一个手动实现的 `Debug` 来总结类型的内容可能更为合适。

如果您的类型旨在以文本输出的形式向最终用户显示，请实现 `Display`。

## 其他地方涵盖的标准特征
除了上一节中描述的常见特征外，标准库还包括其他不太常见的标准特征。在这些额外的标准特征中，以下是最重要的，但它们已在其他项目中介绍，因此这里不再深入介绍：

* `Fn`、`FnOnce` 和 `FnMut`：实现这些特征的项目表示可以调用的闭包。请参阅项目 2。
* `Error`：实现此特征的项目表示可以显示给用户或程序员的错误信息，并且可能包含嵌套的子错误信息。请参阅项目 4。
* `Drop`：实现此特征的项目在被销毁时执行处理，这对于 RAII 模式至关重要。请参阅项目 11。
* `From` 和 `TryFrom`：实现这些特征的项目可以从其他类型的项目中自动创建，但在后一种情况下可能会失败。参见第 5 条。
* `Deref` 和 `DerefMut`：实现这些特征的项目是指针类对象，可以取消引用以访问内部项目。参见第 8 条。
* `Iterator`和它的相关派生：实现这些特征的项目表示可以迭代的集合。参见第 9 条。
* `Send`：实现此特征的项目可以安全地在多个线程之间传输。参见第 17 条。
* `Sync`：实现此特征的项目可以安全地被多个线程引用。参见第 17 条。
这些特征都不是可派生的。

## 运算符重载
最后一类标准特征与运算符重载有关，Rust 允许各种内置的一元和二元运算符为用户定义类型重载，方法是实现 `std::ops` 模块中的各种标准特征。这些特征不可派生，通常仅用于表示“代数”对象的类型，这些类型对这些运算符有自然解释。

但是，C++ 的经验表明，最好避免为不相关的类型重载运算符，因为这通常会导致代码难以维护且具有意外的性能属性（例如，`x + y` 默默调用昂贵的 O(N) 方法）。

为了遵守最小惊讶原则，如果您实现任何运算符重载，则应实现一组连贯的运算符重载。例如，如果 `x + y` 有一个重载（`Add`），并且 `-y`（`Neg`）也有，那么您还应该实现 `x - y`（`Sub`）并确保它给出与 `x +(-y)`相同的答案。

传递给运算符重载特征的项目被移动，这意味着默认情况下将使用非复制类型。添加 `&'a MyType` 的实现可以帮助解决这个问题，但需要更多样板代码来涵盖所有可能性（例如，将引用/非引用参数组合到二元运算符有 `4 = 2 × 2` 种可能性）。

## 摘要

本条目涵盖了很多内容，因此需要一些表格来总结已涉及的标准特征。首先，表 2-1 涵盖了本条目深入涵盖的特征，除 `Display` 之外，所有特征都可以自动得出。

| Trait      |    Compiler Use    |      Bound      | Methods      |
|------------|:------------------:|:---------------:|--------------|
| Clone      |                    |                 | clone        |
| Copy       |     let y = x;     |      Clone      | Marker trait |
| Default    |                    |                 | default      |
| PartialEq  |       x == y       |                 | eq           |
| Eq         |       x == y       |    PartialEq    | Marker trait |
| PartialOrd |  x < y, x <= y, …  |    PartialEq    | partial_cmp  |
| Ord        |  x < y, x <= y, …  | Eq + PartialOrd | cmp          |
| Hash       |                    |                 | hash         |
| Debug      | format!("{:?}", x) |                 | fmt          |
| Display    |  format!("{}", x)  |                 | fmt          |

表 2-2 总结了运算符重载，其中没有一个是可以派生的。

| Trait        | Compiler Use | Bound | Methods       |
|--------------|:------------:|:-----:|---------------|
| Add          |     x + y    |       | add           |
| AddAssign    |    x += y    |       | add_assign    |
| BitAnd       |     x & y    |       | bitand        |
| BitAndAssign |    x &= y    |       | bitand_assign |
| BitOr        |     x ⎮ y    |       | bitor         |
| BitOrAssign  |    x ⎮= y    |       | bitor_assign  |
| BitXor       |     x ^ y    |       | bitxor        |
| BitXorAssign |    x ^= y    |       | bitxor_assign |
| Div          |     x / y    |       | div           |
| DivAssign    |    x /= y    |       | div_assign    |
| Mul          |     x * y    |       | mul           |
| MulAssign    |    x *= y    |       | mul_assign    |
| Neg          |      -x      |       | neg           |
| Not          |      !x      |       | not           |
| Rem          |     x % y    |       | rem           |
| RemAssign    |    x %= y    |       | rem_assign    |
| Shl          |    x << y    |       | shl           |
| ShlAssign    |    x <<= y   |       | shl_assign    |
| Shr          |    x >> y    |       | shr           |
| ShrAssign    |    x >>= y   |       | shr_assign    |
| Sub          |     x - y    |       | sub           |
| SubAssign    |    x -= y    |       | sub_assign    |

为了完整性，其他项目中涵盖的标准特征包含在表 2-3 中；这些特征都不是可派生的（但 Send 和 Sync 可以由编译器自动实现）。

| Trait               |   Item  |      Compiler Use     |      Bound      | Methods      |
|---------------------|:-------:|:---------------------:|:---------------:|--------------|
| Fn                  |  Item 2 |          x(a)         |      FnMut      | call         |
| FnMut               |  Item 2 |          x(a)         |      FnOnce     | call_mut     |
| FnOnce              |  Item 2 |          x(a)         |                 | call_once    |
| Error               |  Item 4 |                       | Display + Debug | [source]     |
| From                |  Item 5 |                       |                 | from         |
| TryFrom             |  Item 5 |                       |                 | try_from     |
| Into                |  Item 5 |                       |                 | into         |
| TryInto             |  Item 5 |                       |                 | try_into     |
| AsRef               |  Item 8 |                       |                 | as_ref       |
| AsMut               |  Item 8 |                       |                 | as_mut       |
| Borrow              |  Item 8 |                       |                 | borrow       |
| BorrowMut           |  Item 8 |                       |      Borrow     | borrow_mut   |
| ToOwned             |  Item 8 |                       |                 | to_owned     |
| Deref               |  Item 8 |         *x, &x        |                 | deref        |
| DerefMut            |  Item 8 |       *x, &mut x      |      Deref      | deref_mut    |
| Index               |  Item 8 |         x[idx]        |                 | index        |
| IndexMut            |  Item 8 |      x[idx] = ...     |      Index      | index_mut    |
| Pointer             |  Item 8 |   format("{:p}", x)   |                 | fmt          |
| Iterator            |  Item 9 |                       |                 | next         |
| IntoIterator        |  Item 9 |       for y in x      |                 | into_iter    |
| FromIterator        |  Item 9 |                       |                 | from_iter    |
| ExactSizeIterator   |  Item 9 |                       |     Iterator    | (size_hint)  |
| DoubleEndedIterator |  Item 9 |                       |     Iterator    | next_back    |
| Drop                | Item 11 |    } (end of scope)   |                 | drop         |
| Sized               | Item 12 |                       |                 | Marker trait |
| Send                | Item 17 | cross-thread transfer |                 | Marker trait |
| Sync                | Item 17 |    cross-thread use   |                 | Marker trait |

