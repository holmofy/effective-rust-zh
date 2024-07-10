## 条款4：首选惯用的错误类型

第 3 项条款描述了如何使用标准库为 `Option` 和 `Result` 类型提供的转换，以便使用 `?` 运算符简洁、惯用地处理结果类型。它没有讨论如何最好地处理作为 `Result<T, E>` 的第二个类型参数出现的各种不同错误类型 `E`；这是本项的主题。

这仅在存在各种不同错误类型时才有意义。如果函数遇到的所有不同错误都已经是同一类型，则它只需返回该类型即可。当存在不同类型的错误时，需要决定是否应保留子错误类型信息。

### Error Trait

了解标准 `Trait`（第 10 项）涉及的内容总是好的，这里相关的 `Trait` 是 `std::error::Error`。`Result` 的 `E` 类型参数不必是实现 `Error` 的类型，但它是一种允许包装器表达适当 `Trait` 界限的常见约定——因此最好为错误类型实现 `Error`。

首先要注意的是，`Error` 类型的唯一硬性要求是 `Trait` 界限：任何实现 `Error` 的类型还必须实现以下 `Trait`：

* Display Trait，意味着它可以用 {} 格式化！
* Debug Trait，意味着它可以用 {:?} 格式化！

换句话说，应该可以向用户和程序员显示 `Error` 类型。

该 `Trait` 中唯一的方法是 `source()`，它允许 `Error` 类型公开内部嵌套错误。此方法是可选的——它带有默认实现（第 13 项），返回 `None`，表示内部错误信息不可用。

最后要注意的一点是：如果你正在为 `no_std` 环境（第 33 条）编写代码，则可能无法实现 `Error` — `Error` 特征当前是在 `std` 中实现的，而不是`core`中实现的，因此不可用。

### 最小错误

如果不需要嵌套错误信息，则 `Error` 类型的实现不需要比 `String` 大很多 — 这是一种极少数情况下“字符串类型”变量可能合适的情况。但它确实需要比 `String` 大一点；虽然可以使用 `String` 作为 `E` 类型参数：

```rust
pub fn find_user(username: &str) -> Result<UserId, String> {
    let f = std::fs::File::open("/etc/passwd")
        .map_err(|e| format!("Failed to open password file: {:?}", e))?;
    // ...
}
```

`String` 未实现 `Error`，我们希望这样，以便其他代码区域可以处理 `Error`。无法为 `String` 实现 `Error`，因为特征和类型都不属于我们（所谓的孤儿规则）：

```rust
impl std::error::Error for String {}
```
```rust
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
  --> src/main.rs:18:5
   |
18 |     impl std::error::Error for String {}
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^------
   |     |                          |
   |     |                          `String` is not defined in the current crate
   |     impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

类型别名也无济于事，因为它不会创建新类型，因此不会改变错误消息：

```rust
pub type MyError = String;

impl std::error::Error for MyError {}
```
```rust
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
  --> src/main.rs:41:5
   |
41 |     impl std::error::Error for MyError {}
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^-------
   |     |                          |
   |     |                          `String` is not defined in the current crate
   |     impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

像往常一样，编译器错误消息会给出解决问题的提示。定义一个包装 `String` 类型的元组结构（“新类型模式”，第 6 条）可以实现 `Error`trait，前提是也实现了 `Debug` 和 `Display`：

```rust
#[derive(Debug)]
pub struct MyError(String);

impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::error::Error for MyError {}

pub fn find_user(username: &str) -> Result<UserId, MyError> {
    let f = std::fs::File::open("/etc/passwd").map_err(|e| {
        MyError(format!("Failed to open password file: {:?}", e))
    })?;
    // ...
}
```

为了方便起见，实现 `From<String>` 特征可能很有意义，以便将字符串值轻松转换为 `MyError` 实例（项目 5）：

```rust
impl From<String> for MyError {
    fn from(msg: String) -> Self {
        Self(msg)
    }
}
```

当遇到问号运算符 (`?`) 时，编译器将自动应用任何相关的 `From` 特征实现，这些实现是达到目标错误返回类型所需的。这允许进一步最小化：

```rust
pub fn find_user(username: &str) -> Result<UserId, MyError> {
    let f = std::fs::File::open("/etc/passwd")
        .map_err(|e| format!("Failed to open password file: {:?}", e))?;
    // ...
}
```

此处的错误路径涵盖以下步骤：

* `File::open` 返回 `std::io::Error` 类型的错误。
* `format!` 使用 `std::io::Error` 的 `Debug` 实现将其转换为字符串。
* `?` 使编译器查找并使用可将其从字符串转换为 `MyError` 的 `From` 实现。

### 嵌套错误

另一种情况是，嵌套错误的内容非常重要，应该保留并提供给调用者。

考虑一个库函数，它尝试将文件的第一行作为字符串返回，只要该行不太长。片刻的思考揭示了可能发生的（至少）三种不同类型的故障：

* 该文件可能不存在或无法读取。
* 该文件可能包含无效的 UTF-8 数据，因此无法转换为字符串。
* 该文件的第一行可能太长。

与第 1 条一致，您可以使用类型系统将所有这些可能性表达并包含在枚举中：

```rust
#[derive(Debug)]
pub enum MyError {
    Io(std::io::Error),
    Utf8(std::string::FromUtf8Error),
    General(String),
}
```

这个枚举定义包括一个 `derive(Debug)`，但是为了满足`Error` trait，还需要一个 `Display` 实现：

```rust
impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            MyError::Io(e) => write!(f, "IO error: {}", e),
            MyError::Utf8(e) => write!(f, "UTF-8 error: {}", e),
            MyError::General(s) => write!(f, "General error: {}", s),
        }
    }
}
```

为了轻松访问嵌套错误，覆盖默认的 source() 实现也是有意义的：
```rust
use std::error::Error;

impl Error for MyError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            MyError::Io(e) => Some(e),
            MyError::Utf8(e) => Some(e),
            MyError::General(_) => None,
        }
    }
}
```
使用枚举可以使错误处理简洁，同时仍然保留不同错误类别的所有类型信息：
```rust
use std::io::BufRead; // for `.read_until()`

/// Maximum supported line length.
const MAX_LEN: usize = 1024;

/// Return the first line of the given file.
pub fn first_line(filename: &str) -> Result<String, MyError> {
    let file = std::fs::File::open(filename).map_err(MyError::Io)?;
    let mut reader = std::io::BufReader::new(file);

    // (A real implementation could just use `reader.read_line()`)
    let mut buf = vec![];
    let len = reader.read_until(b'\n', &mut buf).map_err(MyError::Io)?;
    let result = String::from_utf8(buf).map_err(MyError::Utf8)?;
    if result.len() > MAX_LEN {
        return Err(MyError::General(format!("Line too long: {}", len)));
    }
    Ok(result)
}
```
为所有子错误类型实现 `From` trait也是一个好主意 (第 5 条)：
```rust
impl From<std::io::Error> for MyError {
    fn from(e: std::io::Error) -> Self {
        Self::Io(e)
    }
}
impl From<std::string::FromUtf8Error> for MyError {
    fn from(e: std::string::FromUtf8Error) -> Self {
        Self::Utf8(e)
    }
}
```

这样可以防止库用户自己受到孤儿规则的影响：他们不允许在 `MyError` 上实现 `From`，因为trait和struct都对他们来说是外部的。

更好的是，实现 `From` 可以实现更简洁，因为问号运算符将自动执行任何必要的 `From` 转换，从而无需 `.map_err()`：

```rust
use std::io::BufRead; // for `.read_until()`

/// Maximum supported line length.
pub const MAX_LEN: usize = 1024;

/// Return the first line of the given file.
pub fn first_line(filename: &str) -> Result<String, MyError> {
    let file = std::fs::File::open(filename)?; // `From<std::io::Error>`
    let mut reader = std::io::BufReader::new(file);
    let mut buf = vec![];
    let len = reader.read_until(b'\n', &mut buf)?; // `From<std::io::Error>`
    let result = String::from_utf8(buf)?; // `From<string::FromUtf8Error>`
    if result.len() > MAX_LEN {
        return Err(MyError::General(format!("Line too long: {}", len)));
    }
    Ok(result)
}
```

编写完整的错误类型可能涉及大量样板代码，这使其成为通过派生宏（第 28 项）实现自动化的良好候选。但是，您无需自己编写这样的宏：可以考虑使用 David Tolnay 的 [`thiserror`](https://docs.rs/thiserror) 包，它提供了此类宏的高质量、广泛使用的实现。`thiserror`生成的代码还小心避免使任何 `thiserror` 类型在生成的 API 中可见，这反过来意味着与第 24 项相关的问题不适用。

### trait对象

第一种处理嵌套错误的方法抛弃了所有子错误细节，只保留了一些字符串输出（`format!("{:?}", err)`）。第二种方法保留了所有可能子错误的完整类型信息，但需要对所有可能的子错误类型进行完整枚举。

这就提出了一个问题，这两种方法之间是否存在中间立场，即保留子错误信息而不需要手动包含每种可能的错误类型？

将子错误信息编码为特征对象避免了对每种可能性的枚举变量的需求，但会删除特定底层错误类型的细节。这种对象的接收者可以访问 Error 特征及其特征边界的方法——依次为 `source()`、`Display::fmt()` 和 `Debug::fmt()`——但不知道子错误的原始静态类型：

```rust
#[derive(Debug)]
pub enum WrappedError {
    Wrapped(Box<dyn Error>),
    General(String),
}

impl std::fmt::Display for WrappedError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Wrapped(e) => write!(f, "Inner error: {}", e),
            Self::General(s) => write!(f, "{}", s),
        }
    }
}
```

事实证明这是可能的，但它出奇地微妙。部分困难来自特征对象上的对象安全约束（第 12 条），但 Rust 的一致性规则也发挥了作用，它（粗略地）表明一个类型最多只能有一个特征实现。

假定的 `WrappedError` 类型天真地被期望实现以下两个：

* `Error`trait，因为它本身就是一个错误。
* `From<Error>`trait，允许轻松包装子错误。

这意味着可以从内部 `WrappedError` 创建 `WrappedError`，因为 `WrappedError` 实现了 `Error`，并且与 `From` 的全面反身实现相冲突：

```rust
impl Error for WrappedError {}

impl<E: 'static + Error> From<E> for WrappedError {
    fn from(e: E) -> Self {
        Self::Wrapped(Box::new(e))
    }
}
```
```rust
error[E0119]: conflicting implementations of trait `From<WrappedError>` for
              type `WrappedError`
   --> src/main.rs:279:5
    |
279 |     impl<E: 'static + Error> From<E> for WrappedError {
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |
    = note: conflicting implementation in crate `core`:
            - impl<T> From<T> for T;
```

David Tolnay 的 [`anyhow`](https://docs.rs/anyhow) 是一个已经解决了这些问题的包（通过 `Box` 添加额外的间接层），并且还添加了其他有用的功能（例如堆栈跟踪）。因此，它正在迅速成为错误处理的标准建议 — 此处附议此建议：考虑在应用程序中使用 `anyhow` 包进行错误处理。

### 库与应用程序

上一节的最终建议包括限定条件“…应用程序中的错误处理”。这是因为，为在库中重用而编写的代码与构成顶级应用程序的代码之间通常存在区别。

为库编写的代码无法预测代码的使用环境，因此最好发出具体、详细的错误信息，让调用者弄清楚如何使用这些信息。这倾向于前面描述的枚举样式嵌套错误（并且还避免了对库的公共 API 中的 `anyhow` 的依赖，参见第 24 条）。

但是，应用程序代码通常需要更多地关注如何向用户呈现错误。它还可能必须应对其依赖关系图中存在的所有库发出的所有不同错误类型（第 25 条）。因此，更动态的错误类型（例如 `anyhow::Error`）使整个应用程序的错误处理更简单、更一致。

### 要记住的事情

标准 `Error`trait对您要求不高，因此最好为您的错误类型实现它。

处理异构底层错误类型时，请确定是否有必要保留这些类型。

如果不需要，请考虑使用 `anyhow` 包装应用程序代码中的子错误。

如果是，请将它们编码为枚举并提供转换。考虑使用 `thiserror` 来帮助完成此操作。

考虑使用 `anyhow` 包在应用程序代码中方便地进行惯用错误处理。

这是您的决定，但无论您决定什么，都将其编码到类型系统中（第 1 条）。
