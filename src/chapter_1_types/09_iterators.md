## 考虑使用迭代器转换而不是显式循环

不起眼的循环经历了漫长的历史，变得越来越方便，越来越抽象。[B语言](https://web.archive.org/web/20150611114427/https://www.bell-labs.com/usr/dmr/www/kbman.pdf)（C 的前身）只有 `while (condition) { ... }`，但随着 C 的出现，通过添加 `for` 循环，遍历数组索引的常见场景变得更加方便：

```c
// C code
int i;
for (i = 0; i < len; i++) {
  Item item = collection[i];
  // body
}
```

C++ 的早期版本通过允许将循环变量声明嵌入到 `for` 语句中进一步提高了便利性和范围（这也被 C99 中的 C 所采用）：

```cpp
// C++98 code
for (int i = 0; i < len; i++) {
  Item item = collection[i];
  // ...
}
```

大多数现代语言进一步抽象了循环的概念：循环的核心功能通常是移动到某个容器的下一个项目。跟踪到达该项目所需的变量（`index++` 或 `++it`）大多是无关紧要的细节。这种认识产生了两个核心概念：

* **迭代器**：一种旨在重复发出容器的下一个项目，直到耗尽的类型
* **For-each 循环**：一种紧凑的循环表达式，用于迭代容器中的所有项目，将循环变量绑定到该项目而不是到达该项目的细节

这些概念允许循环代码更短，重要的是它更清楚地说明意图：

```cpp
// C++11 code
for (Item& item : collection) {
  // ...
}
```

一旦这些概念可用，它们就显然非常强大，以至于它们很快就被改造到那些尚未拥有它们的语言中（例如，for-each循环被添加到[Java 1.5](https://docs.oracle.com/javase/1.5.0/docs/guide/language/foreach.html)和C++ 11中）。

Rust 包含迭代器和 for-each 样式的循环，但它还包括抽象的下一步：允许将整个循环表示为**iterator transform**（迭代器转换有时也称为迭代器适配器）。与第 3 条对 `Option` 和 `Result` 的讨论一样，本条将尝试展示如何使用这些迭代器转换代替显式循环，并指导何时使用迭代器转换是好主意。特别是，迭代器转换比显式循环更有效，因为编译器可以跳过它可能需要执行的边界检查。

在本项的末尾，一个类似 C 的显式循环用于对向量的前五个偶数项的平方求和：

```rust
let values: Vec<u64> = vec![1, 1, 2, 3, 5 /* ... */];

let mut even_sum_squares = 0;
let mut even_count = 0;
for i in 0..values.len() {
    if values[i] % 2 != 0 {
        continue;
    }
    even_sum_squares += values[i] * values[i];
    even_count += 1;
    if even_count == 5 {
        break;
    }
}
```

应该开始感觉更自然地表达为函数式表达式：

```rust
let even_sum_squares: u64 = values
    .iter()
    .filter(|x| *x % 2 == 0)
    .take(5)
    .map(|x| x * x)
    .sum();
```

像这样的迭代器转换表达式可以大致分为三个部分：

* 初始源迭代器，来自实现 Rust 迭代器特征之一的类型的实例
* 迭代器转换序列
* 最终消费者方法，用于将迭代结果组合成最终值

前两个部分有效地将功能从循环体移出并移入 for 表达式；最后一个部分完全消除了对 for 语句的需求。

### 迭代器trait

核心 Iterator trait 具有一个非常简单的接口：一个单一方法 `next`，它会产生一些项，直到不产生（None）。发出的项的类型由trait的关联 Item 类型给出。

允许对其内容进行迭代的集合（在其他语言中称为可迭代对象）实现了 `IntoIterator` trait；此trait的 `into_iter` 方法使用 `Self` 并代替它发出 `Iterator`。编译器将自动将此特征用于以下形式的表达式：

```rust
for item in collection {
    // body
}
```

有效地将它们转换为大致如下的代码：

```rust
let mut iter = collection.into_iter();
loop {
    let item: Thing = match iter.next() {
        Some(item) => item,
        None => break,
    };
    // body
}
```

或者更简洁、更地道的写法：

```rust
let mut iter = collection.into_iter();
while let Some(item) = iter.next() {
    // body
}
```

为了让一切顺利运行，任何 `Iterator` 都有一个 `IntoIterator` 实现，它只返回自身；毕竟，将 `Iterator` 转换为 `Iterator` 很容易！

此初始形式是一个消费迭代器，在创建集合时使用它：

```rust
let collection = vec![Thing(0), Thing(1), Thing(2), Thing(3)];
for item in collection {
    println!("Consumed item {item:?}");
}
```

任何在迭代之后尝试使用该集合的行为都会失败：

```rust
println!("Collection = {collection:?}");
```

```rust
error[E0382]: borrow of moved value: `collection`
   --> src/main.rs:171:28
    |
163 |   let collection = vec![Thing(0), Thing(1), Thing(2), Thing(3)];
    |       ---------- move occurs because `collection` has type `Vec<Thing>`,
    |                  which does not implement the `Copy` trait
164 |   for item in collection {
    |               ---------- `collection` moved due to this implicit call to
    |                           `.into_iter()`
...
171 |   println!("Collection = {collection:?}");
    |                          ^^^^^^^^^^^^^^ value borrowed here after move
    |
note: `into_iter` takes ownership of the receiver `self`, which moves
      `collection`
```

虽然很容易理解，但这种消耗所有资源的行为通常是不受欢迎的；需要某种形式的迭代项借用。

为了确保行为清晰，此处的示例使用未实现 `Copy` 的 `Thing` 类型（第 10 条），因为这会隐藏所有权问题（第 15 条）——编译器会默默地在任何地方进行复制：

```rust
// Deliberately not `Copy`
#[derive(Clone, Debug, Eq, PartialEq)]
struct Thing(u64);

let collection = vec![Thing(0), Thing(1), Thing(2), Thing(3)];
```

如果迭代的集合以 & 为前缀：

```rust
for item in &collection {
    println!("{}", item.0);
}
println!("collection still around {collection:?}");
```

然后 Rust 编译器会为类型 &Collection 寻找 IntoIterator 的实现。设计合理的集合类型会提供这样的实现；这个实现仍然会使用 Self，但现在 Self 是 &Collection 而不是 Collection，关联的 Item 类型将是引用 &Thing。

这样在迭代后集合仍保持完整，等效的扩展代码如下：

```rust
let mut iter = (&collection).into_iter();
while let Some(item) = iter.next() {
    println!("{}", item.0);
}
```

如果提供对可变引用的迭代是有意义的，那么 `for item in &mut collection` 也适用类似的模式：编译器查找并使用 `&mut Collection` 的 `IntoIterator` 实现，其中每个 `Item` 都是 `&mut Thing` 类型。

按照惯例，标准容器还提供了一个 `iter()` 方法，该方法返回对底层项的引用的迭代器，以及等效的 `iter_mut()` 方法（如果适用），其行为与刚才描述的相同。这些方法可以在 for 循环中使用，但用作迭代器转换的起点时具有更明显的好处：

```rust
let result: u64 = (&collection).into_iter().map(|thing| thing.0).sum();
```

变成：

```rust
let result: u64 = collection.iter().map(|thing| thing.0).sum();
```

### 迭代器转换

`Iterator` trait 只有一个必需方法 (`next`)，但也提供了对迭代器执行转换的大量其他方法的默认实现 (第 13 项)。

其中一些转换会影响整个迭代过程：

* `take(n)`：限制迭代器最多发出 n 个项目。
* `skip(n)`：跳过迭代器的前 n 个元素。
* `step_by(n)`：转换迭代器，使其仅发出每第 n 个项目。
* `chain(other)`：将两个迭代器粘合在一起，以构建一个组合迭代器，该迭代器先穿过一个迭代器，然后穿过另一个迭代器。
* `cycle()`：将终止的迭代器转换为永远重复的迭代器，每当到达末尾时，都会从头开始。（迭代器必须支持 `Clone` 才能允许这样做。）
* `rev()`：反转迭代器的方向。（迭代器必须实现 `DoubleEndedIterator` 特征，它具有额外的 `next_back` 必需方法。）

其他转换会影响迭代器所针对的项目的性质：

* `map(|item| {...})`：重复应用闭包依次转换每个项目。这是最通用的版本；此列表中的以下几个条目是可以等效地实现为映射的便捷变体。
* `cloned()`：生成原始迭代器中所有项目的克隆；这对于 &Item 引用上的迭代器特别有用。（这显然需要底层 Item 类型实现 Clone。）
* `copied()`：生成原始迭代器中所有项目的副本；这对于 &Item 引用上的迭代器特别有用。（这显然需要底层 Item 类型实现 Copy，但如果是这种情况，它可能比 cloned() 更快。）
* `enumerate()`：将项目上的迭代器转换为 (usize, Item) 对上的迭代器，为迭代器中的项目提供索引。
* `zip(it)`：将一个迭代器与第二个迭代器连接起来，生成一个组合迭代器，该迭代器发出成对的项目，每个项目来自原始迭代器中的一个，直到两个迭代器中较短的一个完成。

还有其他转换对迭代器发出的项目执行过滤：

* `filter(|item| {...})`：将布尔返回闭包应用于每个项目引用，以确定是否应传递它。
* `take_while()`：根据谓词发出迭代器的初始子范围。`skip_while` 的镜像。
* `skip_while()`：根据谓词发出迭代器的最终子范围。`take_while` 的镜像。

`flatten()` 方法处理迭代器，其项本身也是迭代器，从而展平结果。就其本身而言，这似乎没什么帮助，但结合以下观察结果，就会发现 Option 和 Result 都充当迭代器：它们要么产生零个项（对于 None、Err(e)），要么产生一个项（对于 Some(v)、Ok(v)）。这意味着展平 Option/Result 值流是一种简单的方法，可以仅提取有效值，而忽略其余值。

总的来说，这些方法允许迭代器进行转换，以便它们精确生成大多数情况下所需的元素序列。

### 迭代器消费者

前两节描述了如何获取迭代器以及如何将其转换为精确迭代所需的正确形状。这种精确定位的迭代可以作为显式 for-each 循环进行：

```rust
let mut even_sum_squares = 0;
for value in values.iter().filter(|x| *x % 2 == 0).take(5) {
    even_sum_squares += value * value;
}
```

但是，大量的 Iterator 方法包​​括许多允许在单个方法调用中使用迭代的方法，从而无需显式 for 循环。

这些方法中最通用的方法是 for_each(|item| {...})，它为 Iterator 生成的每个项目运行一个闭包。这可以完成显式 for 循环可以完成的大部分操作（例外情况将在后面的部分中介绍），但它的通用性也使其使用起来有些尴尬——闭包需要使用对外部状态的可变引用才能发出任何内容：

```rust
let mut even_sum_squares = 0;
values
    .iter()
    .filter(|x| *x % 2 == 0)
    .take(5)
    .for_each(|value| {
        // closure needs a mutable reference to state elsewhere
        even_sum_squares += value * value;
    });
```

但是，如果 for 循环的主体与多种常见模式之一匹配，则有更具体的迭代器使用方法，它们更清晰、更短、更惯用。

这些模式包括从集合中构建单个值的快捷方式：

* `sum()`：对一组数值（整数或浮点数）求和。
* `product()`：将一组数值相乘。
* `min()`：查找集合的最小值，相对于 Item 的 Ord 实现（参见 Item 10）。
* `max()`：查找集合的最大值，相对于 Item 的 Ord 实现（参见 Item 10）。
* `min_by(f)`：查找集合的最小值，相对于用户指定的比较函数 f。
* `max_by(f)`：查找集合的最大值，相对于用户指定的比较函数 f。
* `reduce(f)`：通过在每个步骤中运行一个闭包，该闭包获取迄今为止累积的值和当前项，从而构建 Item 类型的累积值。这是一个更通用的操作，涵盖了前面的方法。
* `fold(f)`：通过在每个步骤中运行一个闭包，该闭包获取迄今为止累积的值和当前项，从而构建任意类型（不仅仅是 Iterator::Item 类型）的累积值。这是 Reduce 的泛化。
* `scan(init, f)`：通过在每个步骤运行一个闭包来构建任意类型的累积值，该闭包采用对某个内部状态和当前项目的可变引用。这是 Reduce 的略有不同的泛化。

还有一些方法可以从集合中选择单个值：

* `find(p)`：查找满足谓词的第一个项。
* `position(p)`：同样查找满足谓词的第一个项，但这次它返回该项的索引。
* `nth(n)`：返回迭代器的第 n 个元素（如果可用）。

有一些方法可以针对集合中的每个项目进行测试：

* `any(p)`：表示谓词对于集合中的任何项是否为真。
* `all(p)`：表示谓词对于集合中的所有项是否为真。

无论哪种情况，如果找到相关反例，迭代都会提前终止。

有些方法允许每个项目使用的闭包出现失败的可能性。在每种情况下，如果闭包返回某个项目的失败，迭代就会终止，整个操作会返回第一个失败：

* `try_for_each(f)`：行为类似于 for_each，但闭包可能会失败
* `try_fold(f)`：行为类似于 fold，但闭包可能会失败
* `try_find(f)`：行为类似于 find，但闭包可能会失败

最后，还有一些方法可以将所有迭代项累积到新集合中。其中最重要的是 `collect()`，它可用于构建实现 `FromIterator` 特征的任何集合类型的新实例。

FromIterator 特征已为所有标准库集合类型（Vec、HashMap、BTreeSet 等）实现，但这种普遍性也意味着您经常必须使用显式类型，因为否则编译器无法确定您是在尝试组装（例如）Vec<i32> 还是 HashSet<i32>：

```rust
use std::collections::HashSet;

// Build collections of even numbers.  Type must be specified, because
// the expression is the same for either type.
let myvec: Vec<i32> = (0..10).into_iter().filter(|x| x % 2 == 0).collect();
let h: HashSet<i32> = (0..10).into_iter().filter(|x| x % 2 == 0).collect();
```

此示例还说明了如何使用范围表达式来生成要迭代的初始数据。

其他（更晦涩难懂的）集合生成方法包括以下内容：

* `unzip()`：将一个对的迭代器分成两个集合
* `partition(p)`：根据应用于每个项目的谓词将迭代器分成两个集合

本条目涉及了多种迭代器方法，但这只是可用方法的子集；有关更多信息，请查阅迭代器文档或阅读《Rust 编程》第 2 版（O'Reilly）第 15 章，其中详细介绍了各种可能性。

这个丰富的迭代器转换集合可供使用。它生成的代码更符合语言习惯、更紧凑，并且意图更清晰。

将循环表达为迭代器转换还可以生成更高效的代码。出于安全考虑，Rust 对访问连续容器（如向量和切片）执行边界检查；尝试访问集合边界以外的值会触发恐慌，而不是访问无效数据。访问容器值（例如 values[i]）的旧式循环可能会受到这些运行时检查的影响，而生成一个又一个值的迭代器已知在范围内。

但是，与等效迭代器转换相比，旧式循环可能不需要额外的边界检查。Rust 编译器和优化器非常擅长分析切片访问周围的代码，以确定是否可以安全地跳过边界检查；Sergey“Shnatsel”Davidoff 的 2023 年文章探讨了其中的微妙之处。

### 从结果值构建集合

上一节描述了如何使用 `collect()` 从迭代器构建集合，但 `collect()` 在处理结果值时还有一个特别有用的功能。

考虑尝试将 i64 值的向量转换为字节 (u8)，乐观地期望它们都适合：

```rust
// In the 2021 edition of Rust, `TryFrom` is in the prelude, so this
// `use` statement is no longer needed.
use std::convert::TryFrom;

let inputs: Vec<i64> = vec![0, 1, 2, 3, 4];
let result: Vec<u8> = inputs
    .into_iter()
    .map(|v| <u8>::try_from(v).unwrap())
    .collect();
```

这种方法一直有效，直到出现一些意外的输入：

```rust
let inputs: Vec<i64> = vec![0, 1, 2, 3, 4, 512];
```

并导致运行时失败：

```rust
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value:
TryFromIntError(())', iterators/src/main.rs:266:36
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

根据第 3 项中的建议，我们希望保留 Result 类型，并使用 `?` 运算符使任何失败都成为调用代码的问题。发出 Result 的明显修改并没有真正起到作用：

```rust
let result: Vec<Result<u8, _>> =
    inputs.into_iter().map(|v| <u8>::try_from(v)).collect();
// Now what?  Still need to iterate to extract results and detect errors.
```

但是，`collect()` 还有一个替代版本，它可以组装一个包含 Vec 的 Result，而不是包含 Results 的 Vec。

强制使用此版本需要 turbofish (::<Result<Vec<_>, _>>)：

```rust
let result: Vec<u8> = inputs
    .into_iter()
    .map(|v| <u8>::try_from(v))
    .collect::<Result<Vec<_>, _>>()?;
```

将其与问号运算符结合使用可产生有用的行为：

如果迭代遇到错误值，则该错误值将发送给调用者并停止迭代。
如果没有遇到错误，则代码的其余部分可以处理正确类型的合理值集合。

### 循环转换

本条目的目的是让您相信，许多显式循环都可以被视为可以转换为迭代器转换的东西。对于不习惯它的程序员来说，这可能感觉有些不自然，所以让我们一步一步地进行转换。

从一个非常类似 C 的显式循环开始，对向量的前五个偶数项的平方求和：

```rust
let mut even_sum_squares = 0;
let mut even_count = 0;
for i in 0..values.len() {
    if values[i] % 2 != 0 {
        continue;
    }
    even_sum_squares += values[i] * values[i];
    even_count += 1;
    if even_count == 5 {
        break;
    }
}
```

第一步是在 for-each 循环中直接使用迭代器替换向量索引：

```rust
let mut even_sum_squares = 0;
let mut even_count = 0;
for value in values.iter() {
    if value % 2 != 0 {
        continue;
    }
    even_sum_squares += value * value;
    even_count += 1;
    if even_count == 5 {
        break;
    }
}
```

循环的初始部分使用 continue 来跳过一些项目，自然地表示为 `filter()`：

```rust
let mut even_sum_squares = 0;
let mut even_count = 0;
for value in values.iter().filter(|x| *x % 2 == 0) {
    even_sum_squares += value * value;
    even_count += 1;
    if even_count == 5 {
        break;
    }
}
```

接下来，一旦发现五个偶数项，就会提前退出循环，映射到 `take(5)`：

```rust
let mut even_sum_squares = 0;
for value in values.iter().filter(|x| *x % 2 == 0).take(5) {
    even_sum_squares += value * value;
}
```

循环的每次迭代都只使用值 * 值组合中的项的平方，这使其成为 `map()` 的理想目标：

```rust
let mut even_sum_squares = 0;
for val_sqr in values.iter().filter(|x| *x % 2 == 0).take(5).map(|x| x * x)
{
    even_sum_squares += val_sqr;
}
```

对原始循环进行这些重构后，得到的循环体就成为了 `sum()` 方法的完美钉子：

```rust
let even_sum_squares: u64 = values
    .iter()
    .filter(|x| *x % 2 == 0)
    .take(5)
    .map(|x| x * x)
    .sum();
```

### 何时显式更好

本条目强调了迭代器转换的优势，特别是在简洁和清晰方面。那么，何时迭代器转换不合适或不合时宜呢？

如果循环体很大和/或多功能，则将其保留为显式体而不是将其压缩到闭包中是有意义的。
如果循环体涉及导致周围函数提前终止的错误条件，则通常最好将其保留为显式 - try_..() 方法只能提供一点帮助。但是，collect() 将 Result 值集合转换为包含值集合的 Result 的能力通常允许仍然使用 ? 运算符处理错误条件。
如果性能至关重要，则应优化涉及闭包的迭代器转换，使其与等效的显式代码一样快。但是如果核心循环的性能如此重要，请测量不同的变体并进行适当的调整：
请务必确保您的测量结果反映了实际性能——编译器的优化器可能会对测试数据产生过于乐观的结果（如第 30 条所述）。
Godbolt 编译器资源管理器是一款出色的工具，可用于探索编译器输出的内容。
最重要的是，如果转换是强制的或不方便的，请不要将循环转换为迭代转换。这当然是一个品味问题——但请注意，随着您越来越熟悉函数式风格，您的品味可能会发生变化。