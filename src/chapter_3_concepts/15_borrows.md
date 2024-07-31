## 理解借用检查器

Rust 中的值有所有者，但该所有者可以将值借给代码中的其他地方。这种借用机制涉及引用的创建和使用，受借用检查器（本条目的主题）监管的规则约束。

在幕后，Rust 的引用使用与 C 或 C++ 代码中非常普遍的相同类型的指针值（条目 8），但有规则和限制，以确保避免 C/C++ 的错误。快速比较：

* 与 C/C++ 指针一样，Rust 引用是用 `&value` 符号创建的。
* 与 C++ 引用一样，Rust 引用永远不会为 `nullptr`。
* 与 C/C++ 指针一样，Rust 引用可以在创建后修改以引用其他内容。
* 与 C++ 不同，从值生成引用始终涉及显式 (`&`) 转换 - 如果您看到像 `f(value)` 这样的代码，您就知道 `f` 正在接收该值的所有权。 （但是，如果值的类型实现了 `Copy`，则它可能是该项目副本的所有权——请参阅第 10 条。）
* 与 C/C++ 不同，新创建的引用的可变性始终是显式的 (`&mut`)。如果您看到像 `f(&value)` 这样的代码，您就知道该值不会被修改（即，在 C/C++ 术语中是 `const`）。只有像 `f(&mut value)` 这样的表达式才有可能改变值的内容。
C/C++ 指针和 Rust 引用之间最重要的区别是用术语“借用”表示的：您可以获取对某个项目的引用（指针），但不能永远保留该引用。具体来说，您不能将其保留的时间超过底层项的生命周期，如编译器所跟踪并在第 14 项中探讨的那样。

这些对引用使用的限制使 Rust 能够保证其内存安全，但它们也意味着您必须接受借用规则的认知成本，并接受它将改变您设计软件的方式——特别是其数据结构。

本项首先描述 Rust 引用可以做什么，以及借用检查器使用它们的规则。本项的其余部分重点介绍如何处理这些规则的后果：如何重构、重新设计和重新设计您的代码，以便您能够赢得与借用检查器的斗争。

## 访问控制
有三种方式可以访问 Rust 项目的内容：通过项目的所有者 (`item`)、引用 (`&item`) 或可变引用 (`&mut item`)。访问项目的每种方式都具有对项目的不同权限。粗略地按照存储的 CRUD（创建/读取/更新/删除）模型来表示（使用 Rust 的删除术语代替删除）：

* 项目的所有者可以创建、读取、更新和删除它。
* 可变引用可用于读取底层项目并更新它。
* （普通）引用只能用于读取底层项目。
这些数据访问规则有一个重要的 Rust 特定方面：只有项目的所有者才能移动项目。如果您将移动视为创建（在新位置）和删除项目内存（在旧位置）的某种组合，那么这是有道理的。

这可能会导致对项目具有可变引用的代码出现一些奇怪之处。例如，覆盖`Option`是可以的：

```rust
/// Some data structure used by the code.
#[derive(Debug)]
pub struct Item {
    pub contents: i64,
}

/// Replace the content of `item` with `val`.
pub fn replace(item: &mut Option<Item>, val: Item) {
    *item = Some(val);
}
```

但修改为同时返回先前的值却违反了移动限制：

```rust
/// Replace the content of `item` with `val`, returning the previous
/// contents.
pub fn replace(item: &mut Option<Item>, val: Item) -> Option<Item> {
    let previous = *item; // move out
    *item = Some(val); // replace
    previous
}
```
```rust
error[E0507]: cannot move out of `*item` which is behind a mutable reference
  --> src/main.rs:34:24
   |
34 |         let previous = *item; // move out
   |                        ^^^^^ move occurs because `*item` has type
   |                              `Option<inner::Item>`, which does not
   |                              implement the `Copy` trait
   |
help: consider removing the dereference here
   |
34 -         let previous = *item; // move out
34 +         let previous = item; // move out
   |
```

虽然从可变引用读取是有效的，但此代码试图将值移出，就在用新值替换移动的值之前——试图避免复制原始值。借用检查器必须保守，并注意到两行之间有一段时间可变引用没有引用有效值。

作为人类，我们可以看到这种组合操作——提取旧值并用新值替换它——既安全又有用，因此标准库提供了 `std::mem::replace` 函数来执行它。在幕后，`replace` 使用 `unsafe`（根据第 16 条）一次性执行交换：

```rust
/// Replace the content of `item` with `val`, returning the previous
/// contents.
pub fn replace(item: &mut Option<Item>, val: Item) -> Option<Item> {
    std::mem::replace(item, Some(val)) // returns previous value
}
```

特别是对于 Option 类型，这是一个足够常见的模式，Option 本身也有一个 replace 方法：

```rust
/// Replace the content of `item` with `val`, returning the previous
/// contents.
pub fn replace(item: &mut Option<Item>, val: Item) -> Option<Item> {
    item.replace(val) // returns previous value
}
```

## 借用规则

在 Rust 中借用引用时，需要记住两个关键规则。

第一条规则是，任何引用的范围都必须小于它所引用项的生命周期。生命周期在条目 14 中进行了详细探讨，但值得注意的是，编译器对引用生命周期有特殊行为；非词汇生命周期功能允许缩短引用生命周期，使其在最后一次使用时结束，而不是在封闭块处结束。

借用引用的第二条规则是，除了项的所有者之外，还可以有以下任一项：

* 对项的任意数量的不可变引用
* 对项的单个可变引用
但是，不能同时存在两者（在代码中的同一点）。

因此，可以将对同一项的引用提供给接受多个不可变引用的函数：

```rust
/// Indicate whether both arguments are zero.
fn both_zero(left: &Item, right: &Item) -> bool {
    left.contents == 0 && right.contents == 0
}

let item = Item { contents: 0 };
assert!(both_zero(&item, &item));
```

但采用可变引用的则不能：

```rust
/// Zero out the contents of both arguments.
fn zero_both(left: &mut Item, right: &mut Item) {
    left.contents = 0;
    right.contents = 0;
}

let mut item = Item { contents: 42 };
zero_both(&mut item, &mut item);
```
```rust
error[E0499]: cannot borrow `item` as mutable more than once at a time
   --> src/main.rs:131:26
    |
131 |     zero_both(&mut item, &mut item);
    |     --------- ---------  ^^^^^^^^^ second mutable borrow occurs here
    |     |         |
    |     |         first mutable borrow occurs here
    |     first borrow later used by call
```

对于使用可变和不可变引用混合的函数也存在同样的限制：

```rust
/// Set the contents of `left` to the contents of `right`.
fn copy_contents(left: &mut Item, right: &Item) {
    left.contents = right.contents;
}

let mut item = Item { contents: 42 };
copy_contents(&mut item, &item);
```
```rust
error[E0502]: cannot borrow `item` as immutable because it is also borrowed
              as mutable
   --> src/main.rs:159:30
    |
159 |     copy_contents(&mut item, &item);
    |     ------------- ---------  ^^^^^ immutable borrow occurs here
    |     |             |
    |     |             mutable borrow occurs here
    |     mutable borrow later used by call
```

借用规则允许编译器在别名方面做出更好的决策：跟踪两个不同的指针何时可能或不可能引用内存中的同一底层项。如果编译器可以确保（如在 Rust 中）不可变引用集合指向的内存位置不能通过别名可变引用进行更改，那么它可以生成具有以下优势的代码：

* 它得到了更好的优化：例如，值可以缓存在寄存器中，同时确保底层内存内容不会改变。
* 它更安全：线程之间不同步访问内存（第 17 条）不​​可能发生数据争用。

## 所有者操作

关于引用存在的规则的一个重要结果是，它们还会影响项目所有者可以执行的操作。一种有助于理解这一点的方法是想象涉及所有者的操作是通过在幕后创建和使用引用来执行的。

例如，尝试通过其所有者更新项目相当于创建临时可变引用，然后通过该引用更新项目。如果已经存在另一个引用，则无法创建这个名义上的第二个可变引用：

```rust
let mut item = Item { contents: 42 };
let r = &item;
item.contents = 0;
// ^^^ Changing the item is roughly equivalent to:
//   (&mut item).contents = 0;
println!("reference to item is {:?}", r);
```
```rust
error[E0506]: cannot assign to `item.contents` because it is borrowed
   --> src/main.rs:200:5
    |
199 |     let r = &item;
    |             ----- `item.contents` is borrowed here
200 |     item.contents = 0;
    |     ^^^^^^^^^^^^^^^^^ `item.contents` is assigned to here but it was
    |                       already borrowed
...
203 |     println!("reference to item is {:?}", r);
    |                                           - borrow later used here
```

另一方面，因为允许多个不可变引用，所以当存在不可变引用时，所有者可以从该项目中读取：

```rust
let item = Item { contents: 42 };
let r = &item;
let contents = item.contents;
// ^^^ Reading from the item is roughly equivalent to:
//   let contents = (&item).contents;
println!("reference to item is {:?}", r);
```

但如果存在可变引用则不然：

```rust
let mut item = Item { contents: 42 };
let r = &mut item;
let contents = item.contents; // i64 implements `Copy`
r.contents = 0;
```
```rust
error[E0503]: cannot use `item.contents` because it was mutably borrowed
   --> src/main.rs:231:20
    |
230 |     let r = &mut item;
    |             --------- `item` is borrowed here
231 |     let contents = item.contents; // i64 implements `Copy`
    |                    ^^^^^^^^^^^^^ use of borrowed `item`
232 |     r.contents = 0;
    |     -------------- borrow later used here
```

最后，任何类型的活动引用的存在都会阻止物品的所有者移动或丢弃该物品，因为这意味着引用现在指向一个无效物品：

```rust
let item = Item { contents: 42 };
let r = &item;
let new_item = item; // move
println!("reference to item is {:?}", r);
```
```rust
error[E0505]: cannot move out of `item` because it is borrowed
   --> src/main.rs:170:20
    |
168 |     let item = Item { contents: 42 };
    |         ---- binding `item` declared here
169 |     let r = &item;
    |             ----- borrow of `item` occurs here
170 |     let new_item = item; // move
    |                    ^^^^ move out of `item` occurs here
171 |     println!("reference to item is {:?}", r);
    |                                           - borrow later used here
```

在这种情况下，第 14 条中描述的非词汇生命周期特性特别有用，因为（粗略地说）它会在引用最后一次使用时终止引用的生命周期，而不是在封闭范围的末尾。在移动发生之前将引用的最后一次使用上移意味着编译错误消失：

```rust
let item = Item { contents: 42 };
let r = &item;
println!("reference to item is {:?}", r);

// Reference `r` is still in scope but has no further use, so it's
// as if the reference has already been dropped.
let new_item = item; // move works OK
```

## 赢得与借用检查器的斗争

Rust 新手（甚至是经验丰富的人！）经常会觉得他们正在花时间与借用检查器作斗争。什么样的事情可以帮助你赢得这些战斗？

### 本地代码重构

第一个策略是注意编译器的错误消息，因为 Rust 开发人员已经付出了很多努力使它们尽可能有用：

```rust
/// If `needle` is present in `haystack`, return a slice containing it.
pub fn find<'a, 'b>(haystack: &'a str, needle: &'b str) -> Option<&'a str> {
    haystack
        .find(needle)
        .map(|i| &haystack[i..i + needle.len()])
}

// ...

let found = find(&format!("{} to search", "Text"), "ex");
if let Some(text) = found {
    println!("Found '{text}'!");
}
```
```rust
error[E0716]: temporary value dropped while borrowed
   --> src/main.rs:353:23
    |
353 | let found = find(&format!("{} to search", "Text"), "ex");
    |                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^       - temporary value
    |                   |                 is freed at the end of this statement
    |                   |
    |                   creates a temporary value which is freed while still in
    |                   use
354 | if let Some(text) = found {
    |                     ----- borrow later used here
    |
    = note: consider using a `let` binding to create a longer lived value
```

错误消息的第一部分是重要的部分，因为它描述了编译器认为您违反了什么借用规则以及原因。当您遇到足够多的此类错误时（您会遇到），您可以对借用检查器建立一种直觉，该直觉与前面所述的规则中封装的更理论化的版本相匹配。

错误消息的第二部分包括编译器对如何修复问题的建议，在本例中很简单：

```rust
let haystack = format!("{} to search", "Text");
let found = find(&haystack, "ex");
if let Some(text) = found {
    println!("Found '{text}'!");
}
// `found` now references `haystack`, which outlives it
```

这是两个简单的代码调整之一，可以帮助安抚借用检查器：

* 生命周期延长：使用 `let` 绑定将临时变量（其生命周期仅延伸到表达式的末尾）转换为新的命名局部变量（其生命周期延伸到块的末尾）。
* 生命周期缩短：在引用的使用周围添加一个额外的块 `{ ... }`，以便其生命周期在新块的末尾结束。
* 后者不太常见，因为存在非词汇生命周期：编译器通常可以在块末尾的正式放置点之前找出不再使用的引用。但是，如果您确实发现自己在类似的小块代码周围反复引入人工块，请考虑是否应将该代码封装到自己的方法中。

编译器建议的修复对于较简单的问题很有帮助，但随着您编写更复杂的代码，您可能会发现这些建议不再有用，并且破坏借用规则的解释更难理解：

```rust
let x = Some(Rc::new(RefCell::new(Item { contents: 42 })));

// Call function with signature: `check_item(item: Option<&Item>)`
check_item(x.as_ref().map(|r| r.borrow().deref()));
```
```rust
error[E0515]: cannot return reference to temporary value
   --> src/main.rs:293:35
    |
293 |     check_item(x.as_ref().map(|r| r.borrow().deref()));
    |                                   ----------^^^^^^^^
    |                                   |
    |                                   returns a reference to data owned by the
    |                                       current function
    |                                   temporary value created here
```

在这种情况下，临时引入一系列局部变量会有所帮助，每个局部变量对应一个复杂转换的步骤，每个变量都有一个明确的类型注释：

```rust
let x: Option<Rc<RefCell<Item>>> =
    Some(Rc::new(RefCell::new(Item { contents: 42 })));

let x1: Option<&Rc<RefCell<Item>>> = x.as_ref();
let x2: Option<std::cell::Ref<Item>> = x1.map(|r| r.borrow());
let x3: Option<&Item> = x2.map(|r| r.deref());
check_item(x3);
```
```rust
error[E0515]: cannot return reference to function parameter `r`
   --> src/main.rs:305:40
    |
305 |     let x3: Option<&Item> = x2.map(|r| r.deref());
    |                                        ^^^^^^^^^ returns a reference to
    |                                      data owned by the current function
```

这缩小了编译器所抱怨的精确转换范围，从而允许重组代码：

```rust
let x: Option<Rc<RefCell<Item>>> =
    Some(Rc::new(RefCell::new(Item { contents: 42 })));

let x1: Option<&Rc<RefCell<Item>>> = x.as_ref();
let x2: Option<std::cell::Ref<Item>> = x1.map(|r| r.borrow());
match x2 {
    None => check_item(None),
    Some(r) => {
        let x3: &Item = r.deref();
        check_item(Some(x3));
    }
}
```

一旦底层问题清楚并且得到解决，您就可以自由地将局部变量重新合并在一起，以便您可以假装您一直以来都是正确的：

```rust
let x = Some(Rc::new(RefCell::new(Item { contents: 42 })));

match x.as_ref().map(|r| r.borrow()) {
    None => check_item(None),
    Some(r) => check_item(Some(r.deref())),
};
```

## 数据结构设计

下一个有助于对抗借用检查器的策略是，在设计数据结构时要考虑到借用检查器。灵丹妙药是您的数据结构拥有它们使用的所有数据，避免使用任何引用以及随之而来的生命周期注释的传播，如第 14 项所述。

但是，对于现实世界的数据结构来说，这并不总是可能的；任何时候，数据结构的内部连接形成一个比树形模式更相互关联的图形（一个拥有多个分支的`Root`，每个分支拥有多个`Leaf`等），那么简单的单一所有权就是不可能的。

举一个简单的例子，想象一个简单的登记册，按客人到达的顺序记录客人的详细信息：

```rust
#[derive(Clone, Debug)]
pub struct Guest {
    name: String,
    address: String,
    // ... many other fields
}

/// Local error type, used later.
#[derive(Clone, Debug)]
pub struct Error(String);

/// Register of guests recorded in order of arrival.
#[derive(Default, Debug)]
pub struct GuestRegister(Vec<Guest>);

impl GuestRegister {
    pub fn register(&mut self, guest: Guest) {
        self.0.push(guest)
    }
    pub fn nth(&self, idx: usize) -> Option<&Guest> {
        self.0.get(idx)
    }
}
```

如果此代码还需要能够有效地按到达时间和按姓名字母顺序查找客人，那么从根本上讲，涉及两个不同的数据结构，并且只有其中一个可以拥有数据。

如果涉及的数据既小又不可变，那么只需克隆数据就可以成为一种快速解决方案：

```rust
mod cloned {
    use super::Guest;

    #[derive(Default, Debug)]
    pub struct GuestRegister {
        by_arrival: Vec<Guest>,
        by_name: std::collections::BTreeMap<String, Guest>,
    }

    impl GuestRegister {
        pub fn register(&mut self, guest: Guest) {
            // Requires `Guest` to be `Clone`
            self.by_arrival.push(guest.clone());
            // Not checking for duplicate names to keep this
            // example shorter.
            self.by_name.insert(guest.name.clone(), guest);
        }
        pub fn named(&self, name: &str) -> Option<&Guest> {
            self.by_name.get(name)
        }
        pub fn nth(&self, idx: usize) -> Option<&Guest> {
            self.by_arrival.get(idx)
        }
    }
}
```

但是，如果数据可以修改，这种克隆方法就无法很好地应对。例如，如果需要更新 `Guest` 的地址，则必须找到两个版本并确保它们保持同步。

另一种可能的方法是添加另一层间接层，将 `Vec<Guest>` 视为所有者，并使用该向量中的索引进行名称查找：

```rust
mod indexed {
    use super::Guest;

    #[derive(Default)]
    pub struct GuestRegister {
        by_arrival: Vec<Guest>,
        // Map from guest name to index into `by_arrival`.
        by_name: std::collections::BTreeMap<String, usize>,
    }

    impl GuestRegister {
        pub fn register(&mut self, guest: Guest) {
            // Not checking for duplicate names to keep this
            // example shorter.
            self.by_name
                .insert(guest.name.clone(), self.by_arrival.len());
            self.by_arrival.push(guest);
        }
        pub fn named(&self, name: &str) -> Option<&Guest> {
            let idx = *self.by_name.get(name)?;
            self.nth(idx)
        }
        pub fn named_mut(&mut self, name: &str) -> Option<&mut Guest> {
            let idx = *self.by_name.get(name)?;
            self.nth_mut(idx)
        }
        pub fn nth(&self, idx: usize) -> Option<&Guest> {
            self.by_arrival.get(idx)
        }
        pub fn nth_mut(&mut self, idx: usize) -> Option<&mut Guest> {
            self.by_arrival.get_mut(idx)
        }
    }
}
```

在这种方法中，每个来宾都由一个单独的 `Guest` 项表示，这允许 `named_mut()` 方法返回对该项的可变引用。这反过来意味着更改来宾的地址可以正常工作 - (单个) `Guest` 由 `Vec` 拥有，并且始终可以通过这种方式在幕后联系到：

```rust
let new_address = "123 Bigger House St";
// Real code wouldn't assume that "Bob" exists...
ledger.named_mut("Bob").unwrap().address = new_address.to_string();

assert_eq!(ledger.named("Bob").unwrap().address, new_address);
```

但是，如果客人可以取消注册，则很容易无意中引入错误：

```rust
// Deregister the `Guest` at position `idx`, moving up all
// subsequent guests.
pub fn deregister(&mut self, idx: usize) -> Result<(), super::Error> {
    if idx >= self.by_arrival.len() {
        return Err(super::Error::new("out of bounds"));
    }
    self.by_arrival.remove(idx);

    // Oops, forgot to update `by_name`.

    Ok(())
}
```

现在 `Vec` 可以被混洗了，其中的 `by_name` 索引实际上就像指针一样，我们重新引入了一个世界，在这个世界中，一个错误可能导致这些“指针”指向虚无（超出 `Vec` 边界）或指向不正确的数据：

```rust
ledger.register(alice);
ledger.register(bob);
ledger.register(charlie);
println!("Register starts as: {ledger:?}");

ledger.deregister(0).unwrap();
println!("Register after deregister(0): {ledger:?}");

let also_alice = ledger.named("Alice");
// Alice still has index 0, which is now Bob
println!("Alice is {also_alice:?}");

let also_bob = ledger.named("Bob");
// Bob still has index 1, which is now Charlie
println!("Bob is {also_bob:?}");

let also_charlie = ledger.named("Charlie");
// Charlie still has index 2, which is now beyond the Vec
println!("Charlie is {also_charlie:?}");
```

这里的代码使用了自定义的 `Debug` 实现（未显示），以减少输出的大小；这个截断的输出如下：

```rust
Register starts as: {
  by_arrival: [{n: 'Alice', ...}, {n: 'Bob', ...}, {n: 'Charlie', ...}]
  by_name: {"Alice": 0, "Bob": 1, "Charlie": 2}
}
Register after deregister(0): {
  by_arrival: [{n: 'Bob', ...}, {n: 'Charlie', ...}]
  by_name: {"Alice": 0, "Bob": 1, "Charlie": 2}
}
Alice is Some(Guest { name: "Bob", address: "234 Bobton" })
Bob is Some(Guest { name: "Charlie", address: "345 Charlieland" })
Charlie is None
```

前面的示例显示了`deregister`代码中的一个错误，但即使修复了该错误，也无法阻止调用者挂起索引值并将其与 `nth()` 一起使用——从而获得意外或无效的结果。

核心问题是两个数据结构需要保持同步。处理此问题的更好方法是改用 Rust 的智能指针（第 8 条）。转换为 `Rc` 和 `RefCell` 的组合可避免使用索引作为伪指针的无效问题。更新示例（但保留其中的错误）可得到以下内容：

```rust
mod rc {
    use super::{Error, Guest};
    use std::{cell::RefCell, rc::Rc};

    #[derive(Default)]
    pub struct GuestRegister {
        by_arrival: Vec<Rc<RefCell<Guest>>>,
        by_name: std::collections::BTreeMap<String, Rc<RefCell<Guest>>>,
    }

    impl GuestRegister {
        pub fn register(&mut self, guest: Guest) {
            let name = guest.name.clone();
            let guest = Rc::new(RefCell::new(guest));
            self.by_arrival.push(guest.clone());
            self.by_name.insert(name, guest);
        }
        pub fn deregister(&mut self, idx: usize) -> Result<(), Error> {
            if idx >= self.by_arrival.len() {
                return Err(Error::new("out of bounds"));
            }
            self.by_arrival.remove(idx);

            // Oops, still forgot to update `by_name`.

            Ok(())
        }
        // ...
    }
}
```
```rust
Register starts as: {
  by_arrival: [{n: 'Alice', ...}, {n: 'Bob', ...}, {n: 'Charlie', ...}]
  by_name: [("Alice", {n: 'Alice', ...}), ("Bob", {n: 'Bob', ...}),
            ("Charlie", {n: 'Charlie', ...})]
}
Register after deregister(0): {
  by_arrival: [{n: 'Bob', ...}, {n: 'Charlie', ...}]
  by_name: [("Alice", {n: 'Alice', ...}), ("Bob", {n: 'Bob', ...}),
            ("Charlie", {n: 'Charlie', ...})]
}
Alice is Some(RefCell { value: Guest { name: "Alice",
                                       address: "123 Aliceville" } })
Bob is Some(RefCell { value: Guest { name: "Bob",
                                     address: "234 Bobton" } })
Charlie is Some(RefCell { value: Guest { name: "Charlie",
                                         address: "345 Charlieland" } })
```

输出中不再有不匹配的名称，但是 `Alice` 的条目仍然存在，直到我们通过确保两个集合保持同步来修复该错误：

```rust
pub fn deregister(&mut self, idx: usize) -> Result<(), Error> {
    if idx >= self.by_arrival.len() {
        return Err(Error::new("out of bounds"));
    }
    let guest: Rc<RefCell<Guest>> = self.by_arrival.remove(idx);
    self.by_name.remove(&guest.borrow().name);
    Ok(())
}
```
```rust
Register after deregister(0): {
  by_arrival: [{n: 'Bob', ...}, {n: 'Charlie', ...}]
  by_name: [("Bob", {n: 'Bob', ...}), ("Charlie", {n: 'Charlie', ...})]
}
Alice is None
Bob is Some(RefCell { value: Guest { name: "Bob",
                                     address: "234 Bobton" } })
Charlie is Some(RefCell { value: Guest { name: "Charlie",
                                         address: "345 Charlieland" } })
```

## 智能指针

上一节的最后一个变体是更通用方法的示例：使用 Rust 的智能指针来连接数据结构。

第 8 项描述了 Rust 标准库提供的最常见的智能指针类型：

* `Rc` 允许共享所有权，多个事物引用同一个项目。`Rc` 通常与 `RefCell` 结合使用。
* `RefCell` 允许内部可变性，因此无需可变引用即可修改内部状态。这是以将借用检查从编译时移到运行时为代价的。
* `Arc` 是 `Rc` 的多线程等效项。
* `Mutex`（和 `RwLock`）允许在多线程环境中实现内部可变性，大致相当于 `RefCell`。
* `Cell` 允许 `Copy` 类型的内部可变性。

对于从 C++ 过渡到 Rust 的程序员来说，最常用的工具是 `Rc<T>`（及其线程安全的同类 `Arc<T>`），通常与 `RefCell`（或线程安全的替代方案 `Mutex`）结合使用。将共享指针（甚至是 `std::shared_ptrs`）简单转换为 `Rc<RefCell<T>>` 实例通常会产生一些在 Rust 中可以正常工作的东西，而不会引起借用检查器的太多抱怨。

但是，这种方法意味着您会错过 Rust 为您提供的一些保护。 特别是，当同一项被可变地借用（通过 `borrow_mut()`）而另一个引用存在时，会导致运行时恐慌！ 而不是编译时错误。

例如，一种打破树状数据结构中所有权单向流动的模式是当有一个“所有者”指针从一个项目指向拥有它的东西时，如图 3-3 所示。 这些所有者链接对于在数据结构中移动很有用；例如，向 `Leaf` 添加新的兄弟需要涉及拥有它的 `Branch`。

![树形数据结构布局](https://effective-rust.com/images/treedatastructure.svg)

在 Rust 中实现这种模式可以利用 `Rc<T>` 的更具尝试性的伙伴 `Weak<T>`：

```rust
use std::{
    cell::RefCell,
    rc::{Rc, Weak},
};

// Use a newtype for each identifier type.
struct TreeId(String);
struct BranchId(String);
struct LeafId(String);

struct Tree {
    id: TreeId,
    branches: Vec<Rc<RefCell<Branch>>>,
}

struct Branch {
    id: BranchId,
    leaves: Vec<Rc<RefCell<Leaf>>>,
    owner: Option<Weak<RefCell<Tree>>>,
}

struct Leaf {
    id: LeafId,
    owner: Option<Weak<RefCell<Branch>>>,
}
```

弱引用不会增加主引用计数，因此必须明确检查底层项目是否已消失：

```rust
impl Branch {
    fn add_leaf(branch: Rc<RefCell<Branch>>, mut leaf: Leaf) {
        leaf.owner = Some(Rc::downgrade(&branch));
        branch.borrow_mut().leaves.push(Rc::new(RefCell::new(leaf)));
    }

    fn location(&self) -> String {
        match &self.owner {
            None => format!("<unowned>.{}", self.id.0),
            Some(owner) => {
                // Upgrade weak owner pointer.
                let tree = owner.upgrade().expect("owner gone!");
                format!("{}.{}", tree.borrow().id.0, self.id.0)
            }
        }
    }
}
```

如果 Rust 的智能指针似乎不能满足您的数据结构的需求，那么最后总会有使用原始（并且绝对不智能）指针编写不安全代码的退路。但是，根据第 16 条，这应该是最后的手段——其他人可能已经在安全接口中实现了您想要的语义，如果您搜索标准库和 crates.io，您可能会找到适合这项工作的工具。

例如，假设您有一个函数，它有时会返回对其输入之一的引用，但有时需要返回一些新分配的数据。与第 1 条一致，对这两种可能性进行编码的枚举是在类型系统中表达这一点的自然方式，然后您可以实现第 8 条中描述的各种指针特征。但您不必这样做：标准库已经包含了 `std::borrow::Cow` 类型，一旦您知道它存在，它就会完全覆盖这种情况。

## 自引用数据结构

与借用检查器之间的一个特殊斗争总是阻碍着从其他语言转到 Rust 的程序员：尝试创建自引用数据结构，其中包含自有数据以及对该自有数据内部的引用：

```rust
struct SelfRef {
    text: String,
    // The slice of `text` that holds the title text.
    title: Option<&str>,
}
```

从语法层面上看，此代码无法编译，因为它不符合第 14 条中描述的生命周期规则：引用需要生命周期注释，这意味着包含的数据结构也需要生命周期参数。但生命周期是针对此 `SelfRef` 结构外部的某些东西，这并非本意：引用的数据是结构内部的。

值得从更语义的层面考虑这种限制的原因。Rust 中的数据结构可以移动：从堆栈到堆，从堆到堆栈，从一个地方到另一个地方。如果发生这种情况，“内部”标题指针将不再有效，并且无法保持同步。

这种情况的一个简单替代方法是使用前面探讨的索引方法：文本中的偏移范围不会因移动而失效，并且对借用检查器不可见，因为它不涉及引用：

```rust
struct SelfRefIdx {
    text: String,
    // Indices into `text` where the title text is.
    title: Option<std::ops::Range<usize>>,
}
```

但是，这种索引方法仅适用于简单示例，并且具有前面提到的相同缺点：索引本身变成伪指针，可能会不同步，甚至引用不再存在的文本范围。

当编译器处理异步代码时，会出现更通用的自引用问题。4 粗略地说，编译器将待处理的异步代码块捆绑到一个闭包中，该闭包包含代码和代码使用的任何捕获的环境部分（如第 2 条所述）。这个捕获的环境可以包含值和对这些值的引用。这本质上是一个自引用数据结构，因此异步支持是标准库中 Pin 类型的主要动机。此指针类型将其值“固定”在适当位置，强制该值保持在内存中的同一位置，从而确保内部自引用保持有效。

因此，`Pin` 可用于自引用类型，但正确使用起来很棘手——请务必阅读官方文档。

尽可能避免使用自引用数据结构，或者尝试寻找能够为您解决困难的库箱（例如 ouroborous）。

## 要记住的事情

* Rust 的引用是借用的，这表明它们不能被永远持有。
* 借用检查器允许对一个项目进行多个不可变引用或单个可变引用，但不能同时进行。由于非词汇生命周期，引用的生命周期在最后一次使用时停止，而不是在封闭范围的末尾。
* 借用检查器的错误可以通过多种方式处理：
* 添加额外的 `{ ... }` 范围可以缩短值的生命周期。
* 为值添加命名局部变量将值的生命周期延长到范围的末尾。
* 临时添加多个局部变量可以帮助缩小借用检查器抱怨的范围。
* Rust 的智能指针类型提供了绕过借用检查器规则的方法，因此对于互连数据结构很有用。
* 然而，自引用数据结构在 Rust 中仍然很难处理。