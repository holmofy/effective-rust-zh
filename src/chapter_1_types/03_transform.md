## 首选Option和Result转换而不是显式匹配表达式

第 1 项阐述了枚举的优点，并展示了匹配表达式如何迫使程序员考虑所有可能性。第 1 项还介绍了 Rust 标准库提供的两个普遍存在的枚举：

* Option<T>：表示值（类型 T）可能存在也可能不存在
* Result<T, E>：表示返回值（类型 T）的操作可能不成功，而可能返回错误（类型 E）

本项探讨了在哪些情况下应尽量避免对这些特定枚举使用显式匹配表达式，而是倾向于使用标准库为这些类型提供的各种转换方法。使用这些转换方法（它们本身通常作为匹配表达式在幕后实现）会使代码更紧凑、更符合地道，并且意图更清晰。

第一种不需要匹配的情况是，只有值相关，而值的缺失（以及任何相关错误）可以忽略：

```rust
struct S {
    field: Option<i32>,
}

let s = S { field: Some(42) };
match &s.field {
    Some(i) => println!("field is {i}"),
    None => {}
}
```

对于这种情况，`if let` 表达式短了一行，更重要的是更清晰：

```rust
if let Some(i) = &s.field {
    println!("field is {i}");
}
```

但是，大多数情况下，程序员需要提供相应的 else 分支：缺少值（Option::None），可能伴有相关错误（Result::Err(e)），这是程序员需要处理的问题。设计软件以应对失败路径很难，而且其中大部分都是基本复杂性，任何语法支持都无法解决 - 具体来说，决定操作失败时应该发生什么。

在某些情况下，正确的决定是执行鸵鸟策略 - 把头埋在沙子里，明确不应对失败。您不能完全忽略错误分支，因为 Rust 要求代码处理 Error 枚举的两种变体，但您可以选择将失败视为致命的。执行 panic! 失败意味着程序终止，但其余代码可以假设成功编写。使用显式匹配执行此操作会不必要地冗长：

```rust
let result = std::fs::File::open("/etc/passwd");
let f = match result {
    Ok(f) => f,
    Err(_e) => panic!("Failed to open /etc/passwd!"),
};
// Assume `f` is a valid `std::fs::File` from here onward.
```

Option 和 Result 都提供了一对方法来提取其内部值，如果不存在则恐慌！：unwrap 和 expect。后者允许在失败时个性化错误消息，但无论哪种情况，生成的代码都更短更简单 - 错误处理委托给 .unwrap() 后缀（但仍然存在）：

```rust
let f = std::fs::File::open("/etc/passwd").unwrap();
```

![](https://effective-rust.com/images/transform.svg)