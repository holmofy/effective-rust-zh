## 条款1：用类型系统去表示你的数据结构

> “谁叫他们是程序员而不是打字员”—— [@thingskatedid](https://twitter.com/thingskatedid/status/1400213496785108997)

此条款提供了Rust类型系统的快速教程，从编译器提供的基本类型开始，然后转到将值组合进数据结构中的多种方法。

然后Rust枚举类型是主角。尽管它的基础版本与其他语言提供的枚举是等效的，但将枚举变体与数据字段相结合的能力增强了其灵活性和表达能力。

### 基本类型

对其他强类型编程语言（如C++、Go或Java）的任何人来说，Rust 类型系统的基础知识非常熟悉。有一批特定大小的整数类型，包括有符号 ([i8](https://doc.rust-lang.org/std/primitive.i8.html), [i16](https://doc.rust-lang.org/std/primitive.i16.html), [i32](https://doc.rust-lang.org/std/primitive.i32.html), [i64](https://doc.rust-lang.org/std/primitive.i64.html), [i128](https://doc.rust-lang.org/std/primitive.i128.html)) 和无符号 ([u8](https://doc.rust-lang.org/std/primitive.u8.html), [u16](https://doc.rust-lang.org/std/primitive.u16.html), [u32](https://doc.rust-lang.org/std/primitive.u32.html), [u64](https://doc.rust-lang.org/std/primitive.u64.html), [u128](https://doc.rust-lang.org/std/primitive.u128.html)) 。

还有两种整数类型，其大小与目标系统上的指针大小匹配：有符号的isize和无符号的usize。Rust 并不是那种会在指针和整数之间进行大量转换的语言，所以这种特性并不是特别相关。然而，标准集合返回它们的大小作为一个 usize（来自 .len()），所以集合索引意味着 usize 值非常常见 —— 从容量的角度来看，这是显然没有问题的，因为内存中的集合不可能有比系统上的内存地址更多的项。

整数类型确实让我们第一次意识到 Rust 是一个比 C++ 更严格的世界 —— 尝试将一个较大的整数类型（i32）赋值给更小的整数类型（i16）会在编译时产生错误。

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

这让人感到安心：当程序员进行有风险的操作时，Rust不会坐视不理。这也早早地表明，尽管 Rust 有更严格的规则，但它也有助于编译器消息指向如何遵守规则的方法。

```rust
let x: i32 = 66_000;
let y: i16 = x; // What would this value be?
```

建议的解决方案是抛出一个问题，即如何处理转换会改变值的情况，关于错误处理（第4条）和使用 panic!（第18条）我们将在后面有更多的讨论。

Rust 也不允许一些可能看起来“安全”的操作：

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




