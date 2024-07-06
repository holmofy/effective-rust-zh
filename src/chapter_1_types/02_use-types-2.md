## 使用类型系统来表达公共行为

第 1 条讨论了如何在类型系统中表达数据结构；本项继续讨论 Rust 类型系统中的行为编码。

本项中描述的机制通常会让人感觉很熟悉，因为它们在其他语言中都有直接类似物：

* 函数：将代码块与名称和参数列表关联的通用机制。
* 方法：与特定数据结构的实例关联的函数。方法在面向对象作为编程范式出现后创建的编程语言中很常见。
* 函数指针：C 家族中的大多数语言（包括 C++ 和 Go）都支持函数指针，作为一种允许在调用其他代码时增加额外间接级别的机制。
* 闭包：最初在 Lisp 语言家族中最常见，但已被改造到许多流行的编程语言中，包括 C++（自 C++11 以来）和 Java（自 Java 8 以来）。
* 特征：描述所有适用于同一底层项目的相关功能集合。特征在许多其他语言中都有大致的对应物，包括 C++ 中的抽象类和 Go 和 Java 中的接口。

当然，所有这些机制都有 Rust 特有的细节，本条款将介绍这些细节。

在前面的列表中，特征对本书最为重要，因为它们描述了 Rust 编译器和标准库提供的大量行为。第 2 章重点介绍了为设计和实现特征提供建议的条目，但它们的普遍性意味着它们也经常出现在本章的其他条目中。

### 函数和方法

与其他编程语言一样，Rust 使用函数将代码组织成命名块以供重用，并将代码的输入表示为参数。与其他所有静态类型语言一样，参数的类型和返回值都是明确指定的：

```rust
/// Return `x` divided by `y`.
fn div(x: f64, y: f64) -> f64 {
    if y == 0.0 {
        // Terminate the function and return a value.
        return f64::NAN;
    }
    // The last expression in the function body is implicitly returned.
    x / y
}

/// Function called just for its side effects, with no return value.
/// Can also write the return value as `-> ()`.
fn show(x: f64) {
    println!("x = {x}");
}
```

如果函数与特定数据结构密切相关，则将其表示为方法。方法作用于该类型的项，由 `self` 标识，并包含在 `impl DataStructure` 块中。这以与其他语言类似的面向对象方式将相关数据和代码封装在一起；然而，在 Rust 中，方法可以添加到枚举类型以及结构类型中，以保持 Rust 枚举的普遍性（项目 1）：

```rust
enum Shape {
    Rectangle { width: f64, height: f64 },
    Circle { radius: f64 },
}

impl Shape {
    pub fn area(&self) -> f64 {
        match self {
            Shape::Rectangle { width, height } => width * height,
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        }
    }
}
```

方法的名称为其编码的行为创建了一个标签，方法签名为其输入和输出提供了类型信息。方法的第一个输入将是 `self` 的一些变体，表示该方法可能对数据结构执行的操作：

* `&self` 参数表示可以读取数据结构的内容但不能修改。
* `&mut self` 参数表示该方法可能会修改数据结构的内容。
* `self` 参数表示该方法使用数据结构。

### 函数指针

上一节描述了如何将名称（和参数列表）与某些代码关联起来。但是，调用函数总是会导致执行相同的代码；每次调用之间唯一不同的是函数所操作的数据。这涵盖了很多可能的情况，但如果代码需要在运行时发生变化怎么办？

允许这种情况的最简单的行为抽象是函数指针：指向（仅）某些代码的指针，其类型反映函数的签名：

```rust
fn sum(x: i32, y: i32) -> i32 {
    x + y
}
// Explicit coercion to `fn` type is required...
let op: fn(i32, i32) -> i32 = sum;
```

类型在编译时进行检查，因此在程序运行时，该值只是指针的大小。函数指针没有与之关联的其他数据，因此可以以各种方式将它们视为值：

```rust
// `fn` types implement `Copy`
let op1 = op;
let op2 = op;
// `fn` types implement `Eq`
assert!(op1 == op2);
// `fn` implements `std::fmt::Pointer`, used by the {:p} format specifier.
println!("op = {:p}", op);
// Example output: "op = 0x101e9aeb0"
```

需要注意一个技术细节：需要显式强制转换为 fn 类型，因为仅使用函数名称并不能为您提供 fn 类型的东西：

```rust
let op1 = sum;
let op2 = sum;
// Both op1 and op2 are of a type that cannot be named in user code,
// and this internal type does not implement `Eq`.
assert!(op1 == op2);
```

```rust
error[E0369]: binary operation `==` cannot be applied to type
              `fn(i32, i32) -> i32 {main::sum}`
   --> src/main.rs:102:17
    |
102 |     assert!(op1 == op2);
    |             --- ^^ --- fn(i32, i32) -> i32 {main::sum}
    |             |
    |             fn(i32, i32) -> i32 {main::sum}
    |
help: use parentheses to call these
    |
102 |     assert!(op1(/* i32 */, /* i32 */) == op2(/* i32 */, /* i32 */));
    |                ++++++++++++++++++++++       ++++++++++++++++++++++
```

相反，编译器错误表明该类型类似于 `fn(i32, i32) -> i32 {main::sum}`，这种类型完全是编译器内部的类型（即不能用用户代码编写），并且标识特定函数及其签名。换句话说，出于优化原因，`sum` 的类型对函数的签名及其位置进行了编码；此类型可以自动强制（条目 5）为 `fn` 类型。

### 闭包

裸函数指针具有局限性，因为调用函数可用的输入只有那些明确作为参数值传递的输入。例如，考虑一些使用函数指针修改切片的每个元素的代码：

```rust
// In real code, an `Iterator` method would be more appropriate.
pub fn modify_all(data: &mut [u32], mutator: fn(u32) -> u32) {
    for value in data {
        *value = mutator(*value);
    }
}
```

这适用于切片的简单变异：

```rust
fn add2(v: u32) -> u32 {
    v + 2
}
let mut data = vec![1, 2, 3];
modify_all(&mut data, add2);
assert_eq!(data, vec![3, 4, 5]);
```

但是，如果修改依赖于任何其他状态，则不可能将其隐式传递给函数指针：

```rust
let amount_to_add = 3;
fn add_n(v: u32) -> u32 {
    v + amount_to_add
}
let mut data = vec![1, 2, 3];
modify_all(&mut data, add_n);
assert_eq!(data, vec![3, 4, 5]);
```

```rust
error[E0434]: can't capture dynamic environment in a fn item
   --> src/main.rs:125:13
    |
125 |         v + amount_to_add
    |             ^^^^^^^^^^^^^
    |
    = help: use the `|| { ... }` closure form instead
```

错误消息指向了适合该任务的正确工具：闭包。闭包是一段代码，看起来像函数定义（lambda 表达式）的主体，但以下情况除外：

* 它可以作为表达式的一部分构建，因此不需要与其关联的名称。
* 输入参数以竖线 |param1, param2| 给出（它们的关联类型通常可以由编译器自动推断）。
* 它可以捕获周围环境的部分：

```rust
let amount_to_add = 3;
let add_n = |y| {
    // a closure capturing `amount_to_add`
    y + amount_to_add
};
let z = add_n(5);
assert_eq!(z, 8);
```

为了（粗​​略地）理解捕获的工作原理，假设编译器创建一个一次性的内部类型，该类型保存 lambda 表达式中提到的所有环境部分。创建闭包时，会创建此临时类型的一个实例来保存相关值，调用闭包时，该实例将用作附加上下文：

```rust
let amount_to_add = 3;
// *Rough* equivalent to a capturing closure.
struct InternalContext<'a> {
    // references to captured variables
    amount_to_add: &'a u32,
}
impl<'a> InternalContext<'a> {
    fn internal_op(&self, y: u32) -> u32 {
        // body of the lambda expression
        y + *self.amount_to_add
    }
}
let add_n = InternalContext {
    amount_to_add: &amount_to_add,
};
let z = add_n.internal_op(5);
assert_eq!(z, 8);
```

在这个概念上下文中保存的值通常是引用（第 8 条），就像这里一样，但它们也可以是对环境中事物的可变引用，或者是完全移出环境的值（通过在输入参数前使用 move 关键字）。

回到modify_all示例，在需要函数指针的地方不能使用闭包：

```rust
error[E0308]: mismatched types
   --> src/main.rs:199:31
    |
199 |         modify_all(&mut data, |y| y + amount_to_add);
    |         ----------            ^^^^^^^^^^^^^^^^^^^^^ expected fn pointer,
    |         |                                           found closure
    |         |
    |         arguments to this function are incorrect
    |
    = note: expected fn pointer `fn(u32) -> u32`
                  found closure `[closure@src/main.rs:199:31: 199:34]`
note: closures can only be coerced to `fn` types if they do not capture any
      variables
   --> src/main.rs:199:39
    |
199 |         modify_all(&mut data, |y| y + amount_to_add);
    |                                       ^^^^^^^^^^^^^ `amount_to_add`
    |                                                     captured here
note: function defined here
   --> src/main.rs:60:12
    |
60  |     pub fn modify_all(data: &mut [u32], mutator: fn(u32) -> u32) {
    |            ^^^^^^^^^^                   -----------------------
```

相反，接收闭包的代码必须接受 `Fn*` 特征之一的实例：

```rust
pub fn modify_all<F>(data: &mut [u32], mut mutator: F)
where
    F: FnMut(u32) -> u32,
{
    for value in data {
        *value = mutator(*value);
    }
}
```

Rust 有三种不同的 `Fn*` trait，它们之间表达了这种环境捕获行为的一些区别：

* `FnOnce`：描述只能调用一次的闭包。如果环境的某个部分被移动到闭包的上下文中，并且闭包的主体随后将其移出闭包的上下文，那么这些移动只能发生一次 - 没有其他源项的副本可以移动 - 因此闭包只能被调用一次。
* `FnMut`：描述可以重复调用的闭包，它可以更改其环境，因为它可变地从环境中借用。
* `Fn`：描述可以重复调用的闭包，并且只能不可变地从环境中借用值。

编译器会自动为代码中的任何 lambda 表达式实现这些 `Fn*` trait的适当子集；无法手动实现任何这些trait（与 C++ 的 `operator()` 重载不同）。

回到前面闭包的粗略思维模型，编译器自动实现的特征大致对应于捕获的环境上下文是否具有以下元素：

* `FnOnce`：任何移动的值
* `FnMut`：任何对值的可变引用（`&mut T`）
* `Fn`：仅对值的正常引用（`&T`）

此列表中的后两个特征各自具有前一个特征的特征界限，当您考虑使用闭包的事物时，这是有道理的：

* 如果某些东西只需要调用一次闭包（通过接收 FnOnce 表示），则可以向其传递一个可以重复调用的闭包 (FnMut)。
* 如果某些东西需要重复调​​用可能会改变其环境的闭包（通过接收 FnMut 表示），则可以向其传递一个不需要改变其环境的闭包 (Fn)。

裸函数指针类型 `fn` 名义上也属于此列表的末尾；任何（非不安全）的 `fn` 类型都会自动实现所有 `Fn*` 特征，因为它不会从环境中借用任何东西。

因此，在编写接受闭包的代码时，**请使用最通用的 Fn* 特征**，以便为调用者提供最大的灵活性 - 例如，对于仅使用一次的闭包，接受 FnOnce。同样的道理也导致建议**优先使用 `Fn*` 特征界限而不是裸函数指针 (`fn`)**。

### trait

`Fn*` 特征比裸函数指针更灵活，但它们仍然只能描述单个函数的行为，而且即使这样也只能根据函数的签名来描述。

但是，它们本身就是 Rust 类型系统中描述行为的另一种机制的示例，即特征。特征定义了一组相关函数，某些底层项会将其公开；此外，这些函数通常（但不一定是）是方法，以 self 的一些变体作为其第一个参数。

特征中的每个函数也都有一个名称，提供一个标签，允许编译器消除具有相同签名的函数的歧义，更重要的是，允许程序员推断函数的意图。

Rust 特征大致类似于 Go 和 Java 中的“接口”，或 C++ 中的“抽象类”（所有虚方法，没有数据成员）。特征的实现必须提供所有函数（但请注意，特征定义可以包括默认实现；第 13 项），并且还可以具有这些实现使用的关联数据。这意味着代码和数据以一种面向对象 (OO) 的方式封装在一个通用抽象中。

接受结构并调用其函数的代码只能与该特定类型一起使用。如果有多个类型实现通用行为，那么定义一个封装该通用行为的特征并让代码使用该特征的函数（而不是涉及特定结构的函数）会更加灵活。

这导致了与其他受 OO 影响的语言相同的建议：如果预期未来灵活性，则优先接受特征类型而不是具体类型。

有时，您希望在类型系统中区分某些行为，但它无法在特征定义中表达为某些特定函数签名。例如，考虑用于对集合进行排序的 Sort 特征；实现可能是稳定的（比较相同的元素将在排序前后以相同的顺序出现），但没有办法在排序方法参数中表达这一点。

在这种情况下，仍然值得使用类型系统来跟踪此要求，使用标记特征：

```rust
pub trait Sort {
    /// Rearrange contents into sorted order.
    fn sort(&mut self);
}

/// Marker trait to indicate that a [`Sort`] sorts stably.
pub trait StableSort: Sort {}
```

标记特征没有函数，但实现仍必须声明它正在实现该特征——这充当了实现者的承诺：“我庄严宣誓我的实现排序稳定”。依赖稳定排序的代码可以指定 StableSort 特征界限，依靠荣誉系统来保留其不变量。使用标记特征来区分无法在特征函数签名中表达的行为。

一旦行为作为特征封装到 Rust 的类型系统中，它就可以以两种方式使用：

* 作为特征界限，它限制了在编译时通用数据类型或函数可以接受的类型
* 作为特征对象，它限制了在运行时可以存储或传递给函数的类型

以下部分描述了这两种可能性，第 12 条更详细地介绍了它们之间的权衡。

### 特征界限

