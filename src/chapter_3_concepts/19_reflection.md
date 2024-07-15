## 条款19：避免反射

从其他语言转到 Rust 的程序员通常习惯于将反射作为他们工具箱中的一种工具。他们可能会浪费大量时间尝试在 Rust 中实现基于反射的设计，结果却发现他们尝试的东西做得不好，甚至根本做不出来。本条款希望通过描述 Rust 在反射方面能做什么和不能做什么，以及可以用什么来代替，来节省那些浪费在探索死胡同上的时间。

反射是程序在运行时检查自身的能力。给定一个运行时的数据项，它涵盖了以下问题：

* 可以确定关于数据项类型的哪些信息？
* 可以用这些信息做什么？

具有完全反射支持的编程语言对这些问题有广泛的答案。根据反射信息，具有反射功能的语言通常在运行时支持以下部分或全部功能：

* 确定数据项的类型
* 探索其内容
* 修改其字段
* 调用其方法

具有此级别反射支持的语言也往往是动态类型语言（例如 [Python](https://docs.python.org/3/library/types.html#module-types)、Ruby），但也有一些著名的静态类型语言也支持反射，特别是 [Java](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/package-summary.html) 和 [Go](https://golang.org/pkg/reflect/)。

Rust 不支持这种类型的反射，这使得避免反射的建议在这个级别上很容易遵循——这根本不可能。对于来自支持完全反射的语言的程序员来说，这种缺失乍一看似乎是一个很大的差距，但 Rust 的其他功能提供了解决许多相同问题的替代方法。

C++ 有一种更有限的反射形式，称为运行时类型识别 (RTTI，**r**un-**t**ime **t**ype **i**dentification)。 [`typeid`](https://en.cppreference.com/w/cpp/language/typeid) 运算符为每种类型返回一个唯一标识符，适用于多态类型的对象（大致为：具有虚函数的类）：

* `typeid`：可以恢复通过基类引用引用的对象的具体类
* [`dynamic_cast<T>`](https://en.cppreference.com/w/cpp/language/dynamic_cast)：允许将基类引用转换为派生类，前提是这样做是安全且正确的

Rust 也不支持这种 RTTI 风格的反射，继续主题，即本条目的建议很容易遵循。

Rust 确实支持在 [std::any](https://doc.rust-lang.org/std/any/index.html) 模块中提供类似功能的一些函数，但它们受到限制（我们将以某种方式进行探索），因此最好避免使用，除非没有其他可行的替代方案。

`std::any` 中的第一个类似反射的功能乍一看就像魔术一样——一种确定数据项类型名称的方法。以下示例使用用户定义的 `tname()` 函数：

```rust
let x = 42u32;
let y = vec![3, 4, 2];
println!("x: {} = {}", tname(&x), x);
println!("y: {} = {:?}", tname(&y), y);
```

显示类型和值：

```rust
x: u32 = 42
y: alloc::vec::Vec<i32> = [3, 4, 2]
```

`tname()` 的实现揭示了编译器的秘密：这个函数是范型的（根据条款 12），所以每次调用它实际上都是一个不同的函数（`tname::<u32>` 或 `tname::<Square>`）：

```rust
fn tname<T: ?Sized>(_v: &T) -> &'static str {
    std::any::type_name::<T>()
}
```

该实现由 `std::any::type_name<T>` 库函数提供，该函数也是范型的。此函数只能访问编译时信息；没有代码运行来确定运行时的类型。返回第 12 项中使用的特征对象类型可以演示这一点：

```rust
let square = Square::new(1, 2, 2);
let draw: &dyn Draw = &square;
let shape: &dyn Shape = &square;

println!("square: {}", tname(&square));
println!("shape: {}", tname(&shape));
println!("draw: {}", tname(&draw));
```

仅特征对象的类型可用，而不是具体基础项目的类型（`Square`）：

```rust
square: reflection::Square
shape: &dyn reflection::Shape
draw: &dyn reflection::Draw
```

`type_name` 返回的字符串仅适用于诊断 — 它显然是一个“尽力而为”的帮助程序，其内容可能会发生变化，并且可能不是唯一的 — 因此不要尝试解析 type_name 结果。如果您需要全局唯一的类型标识符，请改用 `TypeId`：

```rust
use std::any::TypeId;

fn type_id<T: 'static + ?Sized>(_v: &T) -> TypeId {
    TypeId::of::<T>()
}
```
```rust
println!("x has {:?}", type_id(&x));
println!("y has {:?}", type_id(&y));
```
```rust
x has TypeId { t: 18349839772473174998 }
y has TypeId { t: 2366424454607613595 }
```

输出对人类的帮助不大，但唯一性的保证意味着结果可以在代码中使用。但是，通常最好不要直接使用 `TypeId`，而是使用 `std::any::Any` 特征，因为标准库具有用于处理 Any 实例的附加功能（如下所述）。

`Any` 特征有一个方法 type_id()，它返回实现该特征的类型的 TypeId 值。但是，您无法自己实现此特征，因为 Any 已经为大多数任意类型 T 提供了统一实现：

```rust
impl<T: 'static + ?Sized> Any for T {
    fn type_id(&self) -> TypeId {
        TypeId::of::<T>()
    }
}
```

全面实现并不涵盖所有类型 `T`：`T：'static` 生命周期约束意味着如果 `T` 包含任何具有非 `'static` 生命周期的引用，则不会为 `T` 实现 `TypeId`。这是故意施加的限制，因为生命周期并非类型的完全组成部分：`TypeId::of::<&'a T>` 与 `TypeId::of::<&'b T>` 相同，尽管生命周期不同，这增加了混淆和代码不合理的可能性。

回想一下第 8 条，特征对象是一个胖指针，它包含指向底层项的指针以及指向特征实现的 `vtable` 的指针。对于 `Any`，`vtable` 只有一个条目，用于返回项类型的 `type_id()` 方法，如图 3-4 所示：

```rust
let x_any: Box<dyn Any> = Box::new(42u64);
let y_any: Box<dyn Any> = Box::new(Square::new(3, 4, 3));
```
![`any`trait对象，每个都有指向具体项和 `vtable` 的指针](https://effective-rust.com/images/anytraitobj.svg)

除了几个间接引用之外，`dyn Any` 特征对象实际上是原始指针和类型标识符的组合。这意味着标准库可以提供一些为 `dyn Any` 特征对象定义的其他通用方法；这些方法在某些其他类型 `T` 上是通用的：

* `is::<T>()`：指示特征对象的类型是否等于某个特定的其他类型 `T`
* `downcast_ref::<T>()`：返回对具体类型 `T` 的引用，前提是特征对象的类型与 `T` 匹配
* `downcast_mut::<T>()`：返回对具体类型 `T` 的可变引用，前提是特征对象的类型与 `T` 匹配

请注意，`Any` trait仅近似于反射功能：程序员选择（在编译时）明确构建某些东西（`&dyn Any`）来跟踪项目的编译时类型及其位置。只有在构建 `Any` 特征对象的开销已经发生的情况下，才有可能（比如说）向下转换回原始类型。

Rust 具有与项目相关的不同编译时和运行时类型的场景相对较少。其中最主要的是特征对象：具体类型 Square 的项目可以强制转换为该类型实现的特征的特征对象 dyn Shape。此强制从简单指针（对象/项目）构建一个胖指针（对象 + `vtable`）。

还记得第 12 条中提到的 Rust 的特征对象并不是真正面向对象的。并不是 `Square` 是 `Shape`；而只是 `Square` 实现了 `Shape` 的接口。trait边界也是如此：trait边界 `Shape: Draw` 并不意味着是；它只是意味着也实现了，因为 `Shape` 的 `vtable` 包含 `Draw` 方法的条目。

对于一些简单的特征边界：

```rust
trait Draw: Debug {
    fn bounds(&self) -> Bounds;
}

trait Shape: Draw {
    fn render_in(&self, bounds: Bounds);
    fn render(&self) {
        self.render_in(overlap(SCREEN_BOUNDS, self.bounds()));
    }
}
```

等效特征对象：

```rust
let square = Square::new(1, 2, 2);
let draw: &dyn Draw = &square;
let shape: &dyn Shape = &square;
```

有一个带箭头的布局（如图 3-5 所示；重复自 Item 12），使问题清晰：给定一个 `dyn Shape` 对象，没有立即构建 `dyn Draw` trait对象的方法，因为没有办法返回到 `impl Draw for Square` 的 `vtable`——即使其内容的相关部分（`Square::bounds()` 方法的地址）理论上是可恢复的。（这在 Rust 的后续版本中可能会发生变化；请参阅本 Item 的最后一节。）

![特征边界的特征对象，具有 Draw 和 Shape 的不同 vtable](https://effective-rust.com/images/traitbounds.svg)

将此图与上图进行比较，很明显，显式构造的 `&dyn Any` trait对象没有帮助。`Any` 允许恢复底层项的原始具体类型，但没有运行时方法来查看它实现的特征，或访问可能允许创建特征对象的相关 `vtable`。

那么有什么可用的呢？

主要工具是特征定义，这符合对其他语言的建议——Effective Java Item 65 建议“优先使用接口而不是反射”。如果代码需要依赖于某个项的某些行为的可用性，请将该行为编码为特征（Item 2）。即使所需的行为不能用一组方法签名来表示，也可以使用标记特征来指示符合所需的行为——它比（比如说）自省类的名称来检查特定前缀更安全、更有效。

期望特征对象的代码也可以与具有在程序链接时不可用的支持代码的对象一起使用，因为它已在运行时动态加载（通过 dlopen(3) 或等效方法）——这意味着泛型的单态化（条目 12）是不可能的。

相关地，反射有时也用于其他语言，以允许将同一依赖库的多个不兼容版本同时加载到程序中，从而绕过只能有一个的链接约束。这在 Rust 中是不必要的，因为 Cargo 已经可以处理同一库的多个版本（条目 25）。

最后，宏（尤其是派生宏）可用于自动生成在编译时理解项目类型的辅助代码，作为在运行时解析项目内容的代码的更高效、更类型安全的等效代码。条目 28 讨论了 Rust 的宏系统。

### Rust 未来版本中的向上转型

本条目的文本最初写于 2021 年，一直准确无误，直到 2024 年准备出版这本书时——届时 Rust 将添加一项新功能，以更改一些细节。

当 U 是 T 的超特征之一（`trait T: U {...}`）时，此新“特征向上转型”功能可实现将特征对象 `dyn T` 转换为特征对象 `dyn U` 的向上转型。该功能在正式发布之前已在 `#![feature(trait_upcasting)]` 上进行门控，预计为 Rust 版本 1.76。

对于前面的示例，这意味着 `&dyn Shape` 特征对象现在可以转换为 `&dyn Draw` 特征对象，更接近 Liskov 替换的 is-a 关系。允许这种转换会对 vtable 实现的内部细节产生连锁反应，这可能会比前面的图表中显示的版本更复杂。

但是，此项的中心点不受影响 - `Any` 特征没有超特征，因此上行转换的能力不会对其功能产生任何影响。