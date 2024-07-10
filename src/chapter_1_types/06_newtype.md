## 条款6：拥抱新类型模式

条款 1 描述了元组结构体，其中结构体的字段没有名称，而是通过数字 (self.0) 引用。本条款重点介绍具有某个现有类型的单个条款的元组结构，从而创建一个可以容纳与封闭类型完全相同的值范围的新类型。这种模式在 Rust 中非常普遍，值得拥有自己的条目并拥有自己的名称：新类型模式。

新类型模式最简单的用途是指示[类型的附加语义](https://doc.rust-lang.org/book/ch19-04-advanced-types.html#using-the-newtype-pattern-for-type-safety-and-abstraction)，超出其正常行为。为了说明这一点，想象一个要向火星发送卫星的项目。这是一个大项目，因此不同的小组构建了项目的不同部分。一个小组负责火箭发动机的代码：

```rust
/// Fire the thrusters. Returns generated impulse in pound-force seconds.
pub fn thruster_impulse(direction: Direction) -> f64 {
    // ...
    return 42.0;
}
```

而另一个小组负责惯性制导系统：

```rust
/// Update trajectory model for impulse, provided in Newton seconds.
pub fn update_trajectory(force: f64) {
    // ...
}
```

最终需要将这些不同的部分连接在一起：

```rust
let thruster_force: f64 = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

Rust 包含一个类型别名功能，它允许不同的群体更清楚地表达他们的意图：

```rust
/// Units for force.
pub type PoundForceSeconds = f64;

/// Fire the thrusters. Returns generated impulse.
pub fn thruster_impulse(direction: Direction) -> PoundForceSeconds {
    // ...
    return 42.0;
}
```

```rust
/// Units for force.
pub type NewtonSeconds = f64;

/// Update trajectory model for impulse.
pub fn update_trajectory(force: NewtonSeconds) {
    // ...
}
```

但是，类型别名实际上只是文档；它们比以前版本的文档注释更有力，但没有什么可以阻止在需要 NewtonSeconds 值的地方使用 PoundForceSeconds 值：

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

这是 newtype 模式有帮助的地方：

```rust
/// Units for force.
pub struct PoundForceSeconds(pub f64);

/// Fire the thrusters. Returns generated impulse.
pub fn thruster_impulse(direction: Direction) -> PoundForceSeconds {
    // ...
    return PoundForceSeconds(42.0);
}
```

```rust
/// Units for force.
pub struct NewtonSeconds(pub f64);

/// Update trajectory model for impulse.
pub fn update_trajectory(force: NewtonSeconds) {
    // ...
}
```

顾名思义，newtype 是一种新类型，因此当类型不匹配时，编译器会反对 - 这里尝试将 PoundForceSeconds 值传递给需要 NewtonSeconds 值的对象：

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

```rust
error[E0308]: mismatched types
  --> src/main.rs:76:43
   |
76 |     let new_direction = update_trajectory(thruster_force);
   |                         ----------------- ^^^^^^^^^^^^^^ expected
   |                         |        `NewtonSeconds`, found `PoundForceSeconds`
   |                         |
   |                         arguments to this function are incorrect
   |
note: function defined here
  --> src/main.rs:66:8
   |
66 | pub fn update_trajectory(force: NewtonSeconds) {
   |        ^^^^^^^^^^^^^^^^^ --------------------
help: call `Into::into` on this expression to convert `PoundForceSeconds` into
      `NewtonSeconds`
   |
76 |     let new_direction = update_trajectory(thruster_force.into());
   |                                                         +++++++
```

如第 5 条所述，添加标准 From 特征的实现：

```rust
impl From<PoundForceSeconds> for NewtonSeconds {
    fn from(val: PoundForceSeconds) -> NewtonSeconds {
        NewtonSeconds(4.448222 * val.0)
    }
}
```

允许使用 .into() 执行必要的单位和类型转换：

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force.into());
```

使用新类型来标记类型的附加“单元”语义的相同模式也有助于使纯布尔参数不那么含糊。重新审视第 1 条中的示例，使用新类型使参数的含义变得清晰：

```rust
struct DoubleSided(pub bool);

struct ColorOutput(pub bool);

fn print_page(sides: DoubleSided, color: ColorOutput) {
    // ...
}
```

```rust
print_page(DoubleSided(true), ColorOutput(false));
```

如果需要考虑大小效率或二进制兼容性，那么 #[repr(transparent)] 属性可确保新类型在内存中的表示形式与内部类型相同。

这是 newtype 的简单用法，也是第 1 项的一个具体示例 — 将语义编码到类型系统中，以便编译器负责监管这些语义。

### 绕过特征的孤儿规则

需要使用新类型模式的另一种常见但更微妙的场景围绕着 Rust 的孤儿规则展开。粗略地说，这意味着只有满足以下条件之一时，包才能为类型实现特征：

* 包已定义特征
* 包已定义类型

尝试为外部类型实现外部特征：

```rust
use std::fmt;

impl fmt::Display for rand::rngs::StdRng {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        write!(f, "<StdRng instance>")
    }
}
```

导致编译器错误（这又指向新类型的方向）：

```rust
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
   --> src/main.rs:146:1
    |
146 | impl fmt::Display for rand::rngs::StdRng {
    | ^^^^^^^^^^^^^^^^^^^^^^------------------
    | |                     |
    | |                     `StdRng` is not defined in the current crate
    | impl doesn't use only types from inside the current crate
    |
    = note: define and implement a trait or new type instead
```

这种限制的原因是由于存在歧义风险：如果依赖关系图（第 25 项）中的两个不同的包都（比如说）为 rand::rngs::StdRng 实现 impl std::fmt::Display，那么编译器/链接器就无法在它们之间进行选择。

这经常会导致挫败感：例如，如果您尝试序列化包含来自另一个包的类型的数据，则孤儿规则会阻止您为 somecrate::SomeType.3 编写 impl serde::Serialize

但 newtype 模式意味着您正在定义一个新类型，它是当前包的一部分，因此孤儿特征规则的第二部分适用。现在可以实现外部特征：

```rust
struct MyRng(rand::rngs::StdRng);

impl fmt::Display for MyRng {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        write!(f, "<MyRng instance>")
    }
}
```

### Newtype 限制

newtype 模式解决了这两类问题——防止单位转换和绕过孤儿规则——但它确实带来了一些尴尬：每个涉及 newtype 的操作都需要转发到内部类型。

从简单的层面上讲，这意味着代码必须始终使用 thing.0，而不仅仅是 thing，但这很容易，编译器会告诉您在哪里需要它。

更重要的尴尬是内部类型的任何特征实现都会丢失：因为 newtype 是一种新类型，所以现有的内部实现不适用。

对于可派生特征，这仅意味着 newtype 声明最终会得到大量派生：

```rust
#[derive(Debug, Copy, Clone, Eq, PartialEq, Ord, PartialOrd)]
pub struct NewType(InnerType);
```

然而，对于更复杂的特征，需要一些转发样板来恢复内部类型的实现，例如：

```rust
use std::fmt;
impl fmt::Display for NewType {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        self.0.fmt(f)
    }
}
```