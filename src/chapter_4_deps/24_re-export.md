## 重新导出依赖项的API类型

本条款的标题有点复杂，但通过一个例子可以更清楚地理解。

第 25 项描述了 Cargo 如何以透明的方式支持将同一库包的不同版本链接到单个二进制文件中。考虑一个使用 rand 包的二进制文件——更具体地说，是一个使用 0.8 版包的二进制文件：

```rust
# Cargo.toml file for a top-level binary crate.

[dependencies]
# The binary depends on the `rand` crate from crates.io
rand = "=0.8.5"

# It also depends on some other crate (`dep-lib`).
dep-lib = "0.1.0"
```

```rust
let mut rng = rand::thread_rng(); // rand 0.8
let max: usize = rng.gen_range(5..10);
let choice = dep_lib::pick_number(max);
```

最后一行代码还使用了一个名义上的 `dep-lib` 包作为另一个依赖项。此包可能是来自 crates.io 的另一个包，也可能是通过 Cargo 的路径机制定位的本地包。

此 `dep-lib` 包内部使用 `rand` 包的 0.7 版本：

```rust
# Cargo.toml file for the `dep-lib` library crate.

[dependencies]
# The library depends on the `rand` crate from crates.io
rand = "=0.7.3"
```

```rust
//! The `dep-lib` crate provides number picking functionality.
use rand::Rng;

/// Pick a number between 0 and n (exclusive).
pub fn pick_number(n: usize) -> usize {
    rand::thread_rng().gen_range(0, n)
}
```

眼尖的读者可能会注意到两个代码示例之间的区别：

* 在 `rand` 的 0.7.x 版本中（由 dep-lib 库包使用），`rand::gen_range()` 方法采用两个参数，`low` 和 `high`。
* 在 `rand` 的 0.8.x 版本中（由二进制包使用），`rand::gen_range()` 方法采用单个参数范围。

这不是向后兼容的更改，因此 rand 相应地增加了其最左边的版本组件，这是语义版本控制的要求（第 21 项）。尽管如此，结合两个不兼容版本的二进制文件工作得很好 - cargo 会把所有东西都整理好。

但是，如果 dep-lib 库包的 API 公开了其依赖项的类型，使该依赖项成为公共依赖项，事情就会变得更加尴尬。

例如，假设 dep-lib 入口点涉及一个 Rng 项 - 但具体来说是版本 0.7 的 Rng 项：

```rust
/// Pick a number between 0 and n (exclusive) using
/// the provided `Rng` instance.
pub fn pick_number_with<R: Rng>(rng: &mut R, n: usize) -> usize {
    rng.gen_range(0, n) // Method from the 0.7.x version of Rng
}
```

另外，在您的 API 中使用另一个 crate 的类型之前请仔细考虑：它将您的 crate 与依赖项的类型紧密联系在一起。例如，依赖项的主要版本升级（第 21 项）也将自动要求您的 crate 的主要版本升级。

在这种情况下，rand 是一个半标准 crate，被广泛使用并且只引入了少量自己的依赖项（第 25 项），因此在 crate API 中包含其类型可能总体上是可以的。

回到示例，尝试从顶级二进制文件使用此入口点失败：

```rust
let mut rng = rand::thread_rng();
let max: usize = rng.gen_range(5..10);
let choice = dep_lib::pick_number_with(&mut rng, max);
```

对于 Rust 来说，编译器错误消息不太有用，这是很不寻常的：

```rust
error[E0277]: the trait bound `ThreadRng: rand_core::RngCore` is not satisfied
  --> src/main.rs:22:44
   |
22 |     let choice = dep_lib::pick_number_with(&mut rng, max);
   |                  ------------------------- ^^^^^^^^ the trait
   |                  |                `rand_core::RngCore` is not
   |                  |                 implemented for `ThreadRng`
   |                  |
   |                  required by a bound introduced by this call
   |
   = help: the following other types implement trait `rand_core::RngCore`:
             &'a mut R
```

调查所涉及的类型会导致混淆，因为相关特征似乎确实已实现 - 但调用者实际上实现了（名义上的）RngCore_v0_8_5，而库期望实现 RngCore_v0_7_3。

一旦你最终破译了错误消息并意识到版本冲突是根本原因，你该如何修复它？关键的观察是意识到虽然二进制文件不能直接使用同一个包的两个不同版本，但它可以间接地这样做（如前面显示的原始示例所示）。

从二进制文件作者的角度来看，可以通过添加中间包装器包来解决这个问题，该包装器包隐藏了对 rand v0.7 类型的裸露使用。包装器包与二进制包不同，因此可以独立于二进制包对 rand v0.8 的依赖而独立于对 rand v0.7 的依赖。

这很尴尬，库包的作者可以使用更好的方法。它可以通过明确重新导出以下任一内容来让用户的生活更轻松：

* API 中涉及的类型
* 整个依赖包

对于此示例，后一种方法效果最好：除了提供 0.7 版 Rng 和 RngCore 类型外，它还提供构造类型实例的方法（如 thread_rng()）：

```rust
// Re-export the version of `rand` used in this crate's API.
pub use rand;
```

调用代码现在有一种不同的方式直接引用 rand 的 0.7 版本，如 dep_lib::rand：

```rust
let mut prev_rng = dep_lib::rand::thread_rng(); // v0.7 Rng instance
let choice = dep_lib::pick_number_with(&mut prev_rng, max);
```

记住这个例子，项目标题中给出的建议现在应该不那么晦涩难懂了：重新导出类型出现在你的 API 中的依赖项。

