## 理解生命周期

本条款描述了 Rust 的生命周期，这是对之前编译语言（如 C 和 C++）中存在的概念的更精确的表述——即使不是在理论上，也是在实践中。生命周期是条目 15 中描述的借用检查器的必需输入；这些特性合在一起构成了 Rust 内存安全保障的核心。

### 堆栈简介

生命周期从根本上与堆栈有关，因此需要快速介绍/提醒。

当程序运行时，它使用的内存被划分为不同的块，有时称为段。其中一些块是固定大小的，例如保存程序代码或程序全局数据的块，但其中两个块——堆和堆栈——会随着程序的运行而改变大小。为了实现这一点，它们通常排列在程序虚拟内存空间的两端，因此一个可以向下增长，另一个可以向上增长（至少在程序内存耗尽并崩溃之前），如图3-1所示。

![程序内存分布，包括堆往上增长和栈往下增长](https://effective-rust.com/images/memorylayout.svg)

在这两个动态大小的块中，堆栈用于保存与当前正在执行的函数相关的状态。此状态可以包括以下元素：

传递给函数的参数

函数中使用的局部变量

函数内计算的临时值

函数调用者代码中的返回地址

调用函数 f() 时，会在堆栈中添加一个新的堆栈帧，超出调用函数的堆栈帧结束的位置，并且 CPU 通常会更新寄存器（堆栈指针）以指向新的堆栈帧。

当内部函数 f() 返回时，堆栈指针将重置为调用之前的位置，这将是调用者的堆栈帧，完整无缺且未经修改。

如果调用者随后调用不同的函数 g()，则该过程将再次发生，这意味着 g() 的堆栈帧将重用 f() 之前使用的同一内存区域（如图 3-2 所示）：

```rust
fn caller() -> u64 {
    let x = 42u64;
    let y = 19u64;
    f(x) + g(y)
}

fn f(f_param: u64) -> u64 {
    let two = 2u64;
    f_param + two
}

fn g(g_param: u64) -> u64 {
    let arr = [2u64, 3u64];
    g_param + arr[1]
}
```

![随着函数被调用和返回，堆栈使用情况的演变](https://effective-rust.com/images/stackuse.svg)

当然，这只是对实际发生情况的一个大大简化的版本——将数据放入堆栈和从堆栈中取出需要时间，因此实际处理器会进行许多优化。但是，简化的概念图足以理解本条款的主题。

### 生命周期的演化

上一节解释了参数和局部变量如何存储在堆栈中，并指出这些值只是暂时存储的。

从历史上看，这会导致一些危险的意外情况：如果你持有指向这些暂时堆栈值之一的指针会发生什么？

从 C 开始，返回指向局部变量的指针是完全可以的（尽管现代编译器会对此发出警告）：

```c
/* C code. */
struct File {
  int fd;
};

struct File* open_bugged() {
  struct File f = { open("README.md", O_RDONLY) };
  return &f;  /* return address of stack object! */
}
```

如果您运气不好并且调用代码立即使用返回值，那么您可能会摆脱这种情况：

```c
struct File* f = open_bugged();
printf("in caller: file at %p has fd=%d\n", f, f->fd);
```

```c
in caller: file at 0x7ff7bc019408 has fd=3
```

这很不幸，因为它只是表面上有效。一旦发生任何其他函数调用，堆栈区域将被重用，用于保存对象的内存将被覆盖：

```c
investigate_file(f);
```

```c
/* C code. */
void investigate_file(struct File* f) {
  long array[4] = {1, 2, 3, 4}; // put things on the stack
  printf("in function: file at %p has fd=%d\n", f, f->fd);
}
```

```c
in function: file at 0x7ff7bc019408 has fd=1592262883
```

丢弃对象的内容对于此示例还有另外一个不良影响：与打开的文件相对应的文件描述符丢失，因此程序会泄漏数据结构中保存的资源。

随着时间的流逝，到了 C++，后者的资源访问问题通过包含析构函数得到解决，从而启用了 RAII（参见第 11 条）。现在，堆栈上的内容可以自行整理：如果对象保存某种资源，则析构函数可以整理它，并且 C++ 编译器保证堆栈上对象的析构函数在整理堆栈框架时被调用：

```cpp
// C++ code.
File::~File() {
  std::cout << "~File(): close fd " << fd << "\n";
  close(fd);
  fd = -1;
}
```

调用者现在获得一个指向已被销毁且其资源已被回收的对象的（无效）指针：

```cpp
File* f = open_bugged();
printf("in caller: file at %p has fd=%d\n", f, f->fd);
```

```cpp
~File(): close fd 3
in caller: file at 0x7ff7b6a7c438 has fd=-1
```

然而，C++ 并没有采取任何措施来解决悬垂指针的问题：仍然可以保留指向已经消失的对象的指针（使用已经调用的析构函数）：

```cpp
// C++ code.
void investigate_file(File* f) {
  long array[4] = {1, 2, 3, 4}; // put things on the stack
  std::cout << "in function: file at " << f << " has fd=" << f->fd << "\n";
}
```

```cpp
in function: file at 0x7ff7b6a7c438 has fd=-183042004
```

作为 C/C++ 程序员，您必须注意这一点，并确保不要取消引用指向已消失内容的指针。或者，如果您是攻击者，并且发现了其中一个悬空指针，您更有可能在利用漏洞的过程中疯狂地咯咯笑着并兴高采烈地取消引用该指针。

进入 Rust。Rust 的核心吸引力之一是它从根本上解决了悬空指针的问题，立即解决了很大一部分安全问题。

这样做需要将生命周期的概念从后台（C/C++ 程序员只需知道注意它们，无需任何语言支持）移到前台：每个包含 & 符号的类型都有一个关联的生命周期（'a），即使编译器允许您在大部分时间省略提及它。

### 生命周期的范围

堆栈上项目的生命周期是保证该项目停留在同一位置的时间段；换句话说，这正是保证该项目的引用（指针）不会失效的时间段。

这从创建项目的位置开始，一直延伸到项目被丢弃（Rust 相当于 C++ 中的对象销毁）或移动的位置。

后者的普遍性有时会让来自 C/C++ 的程序员感到惊讶：在很多情况下，Rust 将项目从堆栈上的一个位置移动到另一个位置，或者从堆栈移动到堆，或者从堆移动到堆栈。

项目自动丢弃的确切位置取决于项目是否有名称。

局部变量和函数参数都有名称，相应项的生命周期从创建项并填充名称时开始：

对于局部变量：在 `let var = ...` 声明中

对于函数参数：作为设置函数调用执行框架的一部分
当项被移动到其他地方或名称超出范围时，命名项的生命周期结束：

```rust
#[derive(Debug, Clone)]
/// Definition of an item of some kind.
pub struct Item {
    contents: u32,
}
```

```rust
{
    let item1 = Item { contents: 1 }; // `item1` created here
    let item2 = Item { contents: 2 }; // `item2` created here
    println!("item1 = {item1:?}, item2 = {item2:?}");
    consuming_fn(item2); // `item2` moved here
} // `item1` dropped here
```

还可以“即时”构建一个项，作为表达式的一部分，然后将其输入到其他内容中。这些未命名的临时项在不再需要时将被删除。一种过于简单但有用的思考方法是想象表达式的每个部分都会扩展为其自己的块，临时变量由编译器插入。例如，如下表达式：

```rust
let x = f((a + b) * 2);
```

大致相当于：

```rust
let x = {
    let temp1 = a + b;
    {
        let temp2 = temp1 * 2;
        f(temp2)
    } // `temp2` dropped here
}; // `temp1` dropped here
```

当执行到达原始行末尾的分号时，临时变量已全部删除。

查看编译器计算的项目生命周期的一种方法是插入一个故意的错误以供借用检查器（第 15 条）检测。例如，保留对超出项目生命周期范围的项目的引用：

```rust
let r: &Item;
{
    let item = Item { contents: 42 };
    r = &item;
}
println!("r.contents = {}", r.contents);
```

错误消息指示了项目生命周期的确切终点：

```rust
error[E0597]: `item` does not live long enough
   --> src/main.rs:190:13
    |
189 |         let item = Item { contents: 42 };
    |             ---- binding `item` declared here
190 |         r = &item;
    |             ^^^^^ borrowed value does not live long enough
191 |     }
    |     - `item` dropped here while still borrowed
192 |     println!("r.contents = {}", r.contents);
    |                                 ---------- borrow later used here
```

类似地，对于未命名的临时文件：

```rust
let r: &Item = fn_returning_ref(&mut Item { contents: 42 });
println!("r.contents = {}", r.contents);
```

错误消息显示表达式末尾的端点：

```rust
error[E0716]: temporary value dropped while borrowed
   --> src/main.rs:209:46
    |
209 | let r: &Item = fn_returning_ref(&mut Item { contents: 42 });
    |                                      ^^^^^^^^^^^^^^^^^^^^^ - temporary
    |                                      |           value is freed at the
    |                                      |           end of this statement
    |                                      |
    |                                      creates a temporary value which is
    |                                      freed while still in use
210 | println!("r.contents = {}", r.contents);
    |                             ---------- borrow later used here
    |
    = note: consider using a `let` binding to create a longer lived value
```

关于引用生命周期的最后一点：如果编译器可以证明在代码中的某个点之外没有使用引用，那么它会将引用生命周期的终点视为最后使用的位置，而不是封闭范围的末尾。这个特性称为非词法生命周期，它使借用检查器更加慷慨：

```rust
{
    // `s` owns the `String`.
    let mut s: String = "Hello, world".to_string();

    // Create a mutable reference to the `String`.
    let greeting = &mut s[..5];
    greeting.make_ascii_uppercase();
    // .. no use of `greeting` after this point

    // Creating an immutable reference to the `String` is allowed,
    // even though there's a mutable reference still in scope.
    let r: &str = &s;
    println!("s = '{}'", r); // s = 'HELLO, world'
} // The mutable reference `greeting` would naively be dropped here.
```

### 生命周期代数

尽管在 Rust 中处理引用时，生命周期无处不在，但您无法详细指定它们——无法说“我正在处理从 ref.rs 的第 17 行到第 32 行的生命周期”。相反，您的代码引用具有任意名称的生命周期，通常是 'a、'b、'c、...，并且编译器有自己的内部、不可访问的表示，表示源代码中的内容。（唯一的例外是 'static 生命周期，这是一个特殊情况，将在后续部分中介绍。）

您无法用这些生命周期名称做太多事情；最可能的事情是将一个名称与另一个名称进行比较，重复一个名称以表示两个生命周期是“相同的”。

这个生命周期代数最容易用函数签名来说明：如果函数的输入和输出处理引用，那么它们的生命周期之间的关系是什么？

最常见的情况是，一个函数接收单个引用作为输入，并发出一个引用作为输出。输出引用必须具有生命周期，但它可以是什么呢？只有一种可能性（除了 'static）可供选择：输入的生命周期，这意味着它们都共享同一个名称，例如 'a。将该名称作为生命周期注释添加到这两种类型中，将得到：

```rust
pub fn first<'a>(data: &'a [Item]) -> Option<&'a Item> {
    // ...
}
```

由于这种变体非常常见，并且几乎无法选择输出生命周期，因此 Rust 具有生命周期省略规则，这意味着您不必为这种情况明确编写生命周期名称。相同函数签名的更惯用版本如下：

```rust
pub fn first(data: &[Item]) -> Option<&Item> {
    // ...
}
```

所涉及的引用仍然具有生命周期——省略规则只是意味着您不必编造一个任意的生命周期名称并在两个地方使用它。

如果有多个输入生命周期选择映射到输出生命周期怎么办？在这种情况下，编译器无法确定该怎么做：

```rust
pub fn find(haystack: &[u8], needle: &[u8]) -> Option<&[u8]> {
    // ...
}
```

```rust
error[E0106]: missing lifetime specifier
   --> src/main.rs:56:55
   |
56 | pub fn find(haystack: &[u8], needle: &[u8]) -> Option<&[u8]> {
   |                       -----          -----            ^ expected named
   |                                                     lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the
           signature does not say whether it is borrowed from `haystack` or
           `needle`
help: consider introducing a named lifetime parameter
   |
56 | pub fn find<'a>(haystack: &'a [u8], needle: &'a [u8]) -> Option<&'a [u8]> {
   |            ++++            ++                ++                  ++
```

根据函数和参数名称做出的精明猜测是，此处输出的预期寿命预计与 haystack 输入相匹配：

```rust
pub fn find<'a, 'b>(
    haystack: &'a [u8],
    needle: &'b [u8],
) -> Option<&'a [u8]> {
    // ...
}
```

有趣的是，编译器建议了一种不同的替代方案：让函数的两个输入使用相同的生命周期 'a。例如，以下函数中的这种生命周期组合可能有意义：

```rust
pub fn smaller<'a>(left: &'a Item, right: &'a Item) -> &'a Item {
    // ...
}
```

这似乎意味着两个输入生命周期是“相同的”，但引号（此处和之前）的加入表明情况并非如此。

生命周期的存在理由是确保对项目的引用不会比项目本身的寿命更长；考虑到这一点，与输入生命周期 'a“相同”的输出生命周期 'a 只是意味着输入必须比输出存活更久。

当有两个输入生命周期 'a“相同”时，这只意味着输出生命周期必须包含在两个输入的生命周期内：

```rust
{
    let outer = Item { contents: 7 };
    {
        let inner = Item { contents: 8 };
        {
            let min = smaller(&inner, &outer);
            println!("smaller of {inner:?} and {outer:?} is {min:?}");
        } // `min` dropped
    } // `inner` dropped
} // `outer` dropped
```

换句话说，输出生命周期必须包含在两个输入生命周期中较小的一个内。

相反，如果输出生命周期与其中一个输入的生命周期无关，则不需要这些生命周期嵌套：

```rust
{
    let haystack = b"123456789"; // start of  lifetime 'a
    let found = {
        let needle = b"234"; // start of lifetime 'b
        find(haystack, needle)
    }; // end of lifetime 'b
    println!("found={:?}", found); // `found` used within 'a, outside of 'b
} // end of lifetime 'a
```

### 生命周期省略规则

除了前面描述的“一进一出”省略规则外，还有另外两个省略规则，这意味着可以省略生命周期名称。

第一条规则发生在函数输出中没有引用时；在这种情况下，每个输入引用都会自动获得自己的生命周期，与任何其他输入参数都不同。

第二种规则发生在使用对 self 的引用（&self 或 &mut self）的方法中；在这种情况下，编译器假定任何输出引用都采用 self 的生命周期，因为事实证明这是（迄今为止）最常见的情况。

以下是函数省略规则的摘要：

* 一个输入，一个或多个输出：假定输出具有与输入“相同”的生命周期：
    ```rust
    fn f(x: &Item) -> (&Item, &Item)
    // ... is equivalent to ...
    fn f<'a>(x: &'a Item) -> (&'a Item, &'a Item)
    ```
* 多个输入，没有输出：假设所有输入都有不同的生命周期：
    ```rust
    fn f(x: &Item, y: &Item, z: &Item) -> i32
    // ... is equivalent to ...
    fn f<'a, 'b, 'c>(x: &'a Item, y: &'b Item, z: &'c Item) -> i32
    ```
* 包括 &self 在内的多个输入，一个或多个输出：假设输出生命周期与 &self 的生命周期“相同”：
    ```rust
    fn f(&self, y: &Item, z: &Item) -> &Thing
    // ... is equivalent to ...
    fn f(&'a self, y: &'b Item, z: &'c Item) -> &'a Thing
    ```
当然，如果省略的生命周期名称不符合您的要求，您可以随时明确编写生命周期名称来指定哪些生命周期相互关联。实际上，这很可能是由编译器错误触发的，该错误表明省略的生命周期与函数或其调用者使用相关引用的方式不匹配。

### `'static`生命周期

上一节描述了函数的输入和输出引用生命周期之间的各种可能映射，但忽略了一种特殊情况。如果没有输入生命周期，但输出返回值包含引用，会发生什么情况？

```rust
pub fn the_answer() -> &Item {
    // ...
}
```
```rust
error[E0106]: missing lifetime specifier
   --> src/main.rs:471:28
    |
471 |     pub fn the_answer() -> &Item {
    |                            ^ expected named lifetime parameter
    |
    = help: this function's return type contains a borrowed value, but there
            is no value for it to be borrowed from
help: consider using the `'static` lifetime
    |
471 |     pub fn the_answer() -> &'static Item {
    |                             +++++++
```

唯一允许的可能性是返回的引用具有保证永远不会超出范围的生命周期。这由特殊生命周期 'static 表示，它也是唯一具有特定名称而不是任意占位符名称的生命周期：

```rust
pub fn the_answer() -> &'static Item {
```

获取具有 'static 生命周期的东西的最简单方法是引用被标记为静态的全局变量：

```rust
static ANSWER: Item = Item { contents: 42 };

pub fn the_answer() -> &'static Item {
    &ANSWER
}
```

Rust 编译器保证静态项在整个程序运行期间始终具有相同的地址，并且永远不会移动。这意味着对静态项的引用具有 'static 生命周期，这在逻辑上是足够的。

在许多情况下，对 const 项的引用也会被提升为具有 'static 生命周期，但需要注意几个小问题。首先，如果涉及的类型具有析构函数或内部可变性，则不会发生此提升：

```rust
pub struct Wrapper(pub i32);

impl Drop for Wrapper {
    fn drop(&mut self) {}
}

const ANSWER: Wrapper = Wrapper(42);

pub fn the_answer() -> &'static Wrapper {
    // `Wrapper` has a destructor, so the promotion to the `'static`
    // lifetime for a reference to a constant does not apply.
    &ANSWER
}
```

```rust
error[E0515]: cannot return reference to temporary value
   --> src/main.rs:520:9
    |
520 |         &ANSWER
    |         ^------
    |         ||
    |         |temporary value created here
    |         returns a reference to data owned by the current function
```

第二个潜在的复杂因素是，只有 const 的值才能保证在任何地方都相同；无论在何处使用该变量，编译器都可以根据需要进行多次复制。如果您正在执行依赖于 'static 引用背后的底层指针值的恶意操作，请注意可能涉及多个内存位置。

还有一种可能的方法可以获得具有 'static 生命周期的东西。'static 的关键承诺是生命周期应该比程序中的任何其他生命周期都长；在堆上分配但从未释放的值也满足此约束。

普通的堆分配 Box<T> 不适用于这种情况，因为无法保证（如下一节所述）该项目不会在途中被丢弃：

```rust
{
    let boxed = Box::new(Item { contents: 12 });
    let r: &'static Item = &boxed;
    println!("'static item is {:?}", r);
}
```

```rust
error[E0597]: `boxed` does not live long enough
   --> src/main.rs:344:32
    |
343 |     let boxed = Box::new(Item { contents: 12 });
    |         ----- binding `boxed` declared here
344 |     let r: &'static Item = &boxed;
    |            -------------   ^^^^^^ borrowed value does not live long enough
    |            |
    |            type annotation requires that `boxed` is borrowed for `'static`
345 |     println!("'static item is {:?}", r);
346 | }
    | - `boxed` dropped here while still borrowed
```

但是，Box::leak 函数将拥有的 Box<T> 转换为对 T 的可变引用。该值不再有所有者，因此永远不会被删除 - 这满足了“静态生命周期”的要求：

```rust
{
    let boxed = Box::new(Item { contents: 12 });

    // `leak()` consumes the `Box<T>` and returns `&mut T`.
    let r: &'static Item = Box::leak(boxed);

    println!("'static item is {:?}", r);
} // `boxed` not dropped here, as it was already moved into `Box::leak()`

// Because `r` is now out of scope, the `Item` is leaked forever.
```

无法删除该项目还意味着保存该项目的内存永远无法使用安全 Rust 回收，这可能会导致永久性内存泄漏。（请注意，泄漏内存并不违反 Rust 的内存安全保证 - 您无法再访问的内存中的项目仍然是安全的。）

### 生命周期和堆

到目前为止，讨论都集中在堆栈上项目的生命周期上，无论是函数参数、局部变量还是临时变量。但是堆上的项目呢？

关于堆值，关键是要意识到每个项目都有一个所有者（除了上一节中描述的故意泄漏等特殊情况）。例如，一个简单的 Box<T> 将 T 值放在堆上，所有者是保存 Box<T> 的变量：

```rust
{
    let b: Box<Item> = Box::new(Item { contents: 42 });
} // `b` dropped here, so `Item` dropped too.
```

拥有它的 Box<Item> 在超出范围时会丢弃其内容，因此 Item 在堆上的生存期与 Box<Item> 变量在堆栈上的生存期相同。

堆上值的所有者本身可能在堆上而不是在堆栈上，但那么谁拥有所有者呢？

```rust
{
    let b: Box<Item> = Box::new(Item { contents: 42 });
    let bb: Box<Box<Item>> = Box::new(b); // `b` moved onto heap here
} // `bb` dropped here, so `Box<Item>` dropped too, so `Item` dropped too.
```

所有权链必须在某处结束，并且只有两种可能性：

链在局部变量或函数参数处结束 - 在这种情况下，链中所有内容的生命周期只是该堆栈变量的生命周期 'a。当堆栈变量超出范围时，链中的所有内容也会被丢弃。
链在标记为静态的全局变量处结束 - 在这种情况下，链中所有内容的生命周期都是 'static。静态变量永远不会超出范围，因此链中的任何内容都不会自动被丢弃。
因此，堆上项目的生命周期从根本上与堆栈生命周期相关。

### 数据结构中的生命周期

前面关于生命周期代数的部分集中讨论了函数的输入和输出，但当引用存储在数据结构中时也存在类似的问题。

如果我们试图将引用偷偷放入数据结构中而不提及相关的生命周期，编译器会严厉警告我们：

```rust
pub struct ReferenceHolder {
    pub index: usize,
    pub item: &Item,
}
```

```rust
error[E0106]: missing lifetime specifier
   --> src/main.rs:548:19
    |
548 |         pub item: &Item,
    |                   ^ expected named lifetime parameter
    |
help: consider introducing a named lifetime parameter
    |
546 ~     pub struct ReferenceHolder<'a> {
547 |         pub index: usize,
548 ~         pub item: &'a Item,
    |
```

像往常一样，编译器错误消息会告诉我们该怎么做。第一部分很简单：为引用类型指定一个显式的生命周期名称 'a，因为在数据结构中使用引用时没有生命周期省略规则。

第二部分不太明显，但影响更深远：数据结构本身必须具有一个生命周期参数 <'a>，该参数与其中包含的引用的生命周期相匹配：

```rust
// Lifetime parameter required due to field with reference.
pub struct ReferenceHolder<'a> {
    pub index: usize,
    pub item: &'a Item,
}
```

数据结构的生命周期参数具有传染性：任何使用该类型的包含数据结构也必须获取生命周期参数：

```rust
// Lifetime parameter required due to field that is of a
// type that has a lifetime parameter.
pub struct RefHolderHolder<'a> {
    pub inner: ReferenceHolder<'a>,
}
```

如果数据结构包含切片类型，则也需要生命周期参数，因为这些也是对借用数据的引用。

如果数据结构包含具有相关生命周期的多个字段，则必须选择合适的生命周期组合。在一对字符串中查找公共子字符串的示例是具有独立生命周期的良好候选者：

```rust
/// Locations of a substring that is present in
/// both of a pair of strings.
pub struct LargestCommonSubstring<'a, 'b> {
    pub left: &'a str,
    pub right: &'b str,
}

/// Find the largest substring present in both `left`
/// and `right`.
pub fn find_common<'a, 'b>(
    left: &'a str,
    right: &'b str,
) -> Option<LargestCommonSubstring<'a, 'b>> {
    // ...
}
```

而引用同一字符串中多个位置的数据结构将具有共同的生命周期：

```rust
/// First two instances of a substring that is repeated
/// within a string.
pub struct RepeatedSubstring<'a> {
    pub first: &'a str,
    pub second: &'a str,
}

/// Find the first repeated substring present in `s`.
pub fn find_repeat<'a>(s: &'a str) -> Option<RepeatedSubstring<'a>> {
    // ...
}
```

生命周期参数的传播是有意义的：任何包含引用的东西，无论嵌套程度如何，都只在引用项的生命周期内有效。如果移动或删除了该项，则整个数据结构链将不再有效。

但是，这也意味着涉及引用的数据结构更难使用——数据结构的所有者必须确保所有生命周期都对齐。因此，尽可能优先选择拥有其内容的数据结构，特别是如果代码不需要高度优化（第 20 条）。如果这不可能，第 8 条中描述的各种智能指针类型（例如 Rc）可以帮助解开生命周期约束。

### 匿名生命周期

当无法坚持使用拥有其内容的数据结构时，数据结构必然会以生命周期参数结束，如上一节所述。这可能会与本​​条目前面描述的生命周期省略规则产生略微不利的交互。

例如，考虑一个返回具有生命周期参数的数据结构的函数。此函数的完全显式签名使所涉及的生命周期清晰可见：

```rust
pub fn find_one_item<'a>(items: &'a [Item]) -> ReferenceHolder<'a> {
    // ...
}
```

但是，省略生命周期的相同签名可能会有点误导：

```rust
pub fn find_one_item(items: &[Item]) -> ReferenceHolder {
    // ...
}
```

由于返回类型的生命周期参数被省略，因此阅读代码的人无法获得太多有关生命周期的提示。

匿名生命周期 '_ 允许您将省略的生命周期标记为存在，而无需完全恢复所有生命周期名称：

```rust
pub fn find_one_item(items: &[Item]) -> ReferenceHolder<'_> {
    // ...
}
```

粗略地说，'_ 标记要求编译器为我们发明一个唯一的生命周期名称，我们可以在不需要在其他地方使用该名称的情况下使用该名称。

这意味着它对于其他生命周期省略场景也很有用。例如，Debug 特征的 fmt 方法的声明使用匿名生命周期来指示 Formatter 实例具有与 &self 不同的生命周期，但该生命周期的名称是什么并不重要：

```rust
pub trait Debug {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error>;
}
```

### 要记住的事情

* 所有 Rust 引用都有一个关联的生命周期，由生命周期标签（例如 'a）表示。函数参数和返回值的生命周期标签在某些常见情况下可以省略（但仍存在于幕后）。
* 任何（可传递地）包含引用的数据结构都有一个关联的生命周期参数；因此，使用拥有其内容的数据结构通常更容易。
* 'static 生命周期用于保证永远不会超出范围的项目的引用，例如全局数据或堆上已明确泄漏的项目。
* 生命周期标签只能用于指示生命周期是“相同的”，这意味着输出生命周期包含在输入生命周期内。
* 匿名生命周期标签 '_ 可用于不需要特定生命周期标签的地方。