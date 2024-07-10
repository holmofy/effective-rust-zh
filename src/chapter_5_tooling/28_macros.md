## 条款28：谨慎使用宏

> “在某些情况下，决定编写宏而不是函数很容易，因为只有宏才能完成所需的操作。” – Paul Graham, "[On Lisp (Prentice Hall)](https://www.paulgraham.com/onlisp.html)"

Rust 的宏系统允许您执行元编程：编写代码将代码发送到您的项目中。当存在确定性和重复性的“样板”代码块并且否则需要手动保持同步时，这最有价值。

接触 Rust 的程序员可能以前遇到过 C/C++ 预处理器提供的宏，它们对输入文本的标记执行文本替换。Rust 的宏是另一种野兽，因为它们可以处理程序的解析标记或程序的抽象语法树 (AST)，而不仅仅是其文本内容。

这意味着 Rust 宏可以了解代码结构，从而可以避免整个与宏相关的错误类别。特别是，我们在以下部分中看到 Rust 的声明性宏是[卫生的](https://en.wikipedia.org/wiki/Hygienic_macro) - 它们不会意外引用（“捕获”）周围代码中的局部变量。

思考宏的一种方法是将它们视为代码中不同级别的抽象。一种简单的抽象形式是函数：它抽象出相同类型的不同值之间的差异，实现代码可以使用该类型的任何特性和方法，而不管当前操作的值是什么。泛型是不同级别的抽象：它抽象出满足特征边界的不同类型之间的差异，实现代码可以使用特征边界提供的任何方法，而不管当前操作的类型是什么。

宏抽象出程序中扮演相同角色（类型、标识符、表达式等）的不同片段之间的差异；然后实现可以包含以相同角色使用这些片段的任何代码。

Rust 提供了两种定义宏的方法：

* **声明宏**，也称为“示例宏”，允许根据宏的输入参数（根据其在 AST 中的角色分类）将任意 Rust 代码插入程序中。
* **过程宏**允许根据源代码的解析标记将任意 Rust 代码插入程序中。这最常用于派生宏，它可以根据数据结构定义的内容生成代码。

### 声明宏

虽然本条目不是重现声明性宏文档的地方，但还是需要提醒大家注意一些细节。

首先，请注意使用声明性宏的范围规则与其他 Rust 条目不同。如果在源代码文件中定义了声明性宏，则只有宏定义之后的代码才能使用它：

```rust
fn before() {
    println!("[before] square {} is {}", 2, square!(2));
}

/// Macro that squares its argument.
macro_rules! square {
    { $e:expr } => { $e * $e }
}

fn after() {
    println!("[after] square {} is {}", 2, square!(2));
}
```
```rust
error: cannot find macro `square` in this scope
 --> src/main.rs:4:45
  |
4 |     println!("[before] square {} is {}", 2, square!(2));
  |                                             ^^^^^^
  |
  = help: have you added the `#[macro_use]` on the module/import?
```

`#[macro_export]` 属性使宏更加广泛地可见，但这也有一个奇怪之处：宏出现在包的顶层，即使它是在模块中定义的：

```rust
mod submod {
    #[macro_export]
    macro_rules! cube {
        { $e:expr } => { $e * $e * $e }
    }
}

mod user {
    pub fn use_macro() {
        // Note: *not* `crate::submod::cube!`
        let cubed = crate::cube!(3);
        println!("cube {} is {}", 3, cubed);
    }
}
```

Rust 的声明性宏是所谓的卫生的：宏主体中的展开代码不允许使用局部变量绑定。例如，假设某个变量 x 存在的宏：

```rust
// Create a macro that assumes the existence of a local `x`.
macro_rules! increment_x {
    {} => { x += 1; };
}
```

使用时会触发编译失败：

```rust
let mut x = 2;
increment_x!();
println!("x = {}", x);
```
```rust
error[E0425]: cannot find value `x` in this scope
   --> src/main.rs:55:13
    |
55  |     {} => { x += 1; };
    |             ^ not found in this scope
...
314 |     increment_x!();
    |     -------------- in this macro invocation
    |
    = note: this error originates in the macro `increment_x`
```

这种卫生特性意味着 Rust 的宏比 C 预处理器宏更安全。但是，在使用它们时仍有几个小问题需要注意。

首先要意识到，即使宏调用看起来像函数调用，但实际上并不是。宏在调用时生成代码，并且生成的代码可以对其参数执行操作：

```rust
macro_rules! inc_item {
    { $x:ident } => { $x.contents += 1; }
}
```

这意味着关于参数是否被移动或`&`-引用的正常直觉不适用：

```rust
let mut x = Item { contents: 42 }; // type is not `Copy`

// Item is *not* moved, despite the (x) syntax,
// but the body of the macro *can* modify `x`.
inc_item!(x);

println!("x is {x:?}");
```
```rust
x is Item { contents: 43 }
```

如果我们记得宏在调用时插入代码（在本例中，添加一行增加 x.contents 的代码），这一点就变得很清楚了。cargo-expand 工具显示了编译器在宏扩展后看到的代码：

```rust
let mut x = Item { contents: 42 };
x.contents += 1;
{
    ::std::io::_print(format_args!("x is {0:?}\n", x));
};
```

扩展的代码包括通过项目的所有者而不是引用进行的修改。（同样有趣的是，println! 的扩展版本依赖于稍后讨论的 format_args! 宏。）1

因此，感叹号起到了警告的作用：宏的扩展代码可能会对其参数执行任意操作。

扩展的代码还可以包括调用代码中不可见的控制流操作，无论是循环、条件、返回语句还是使用 ? 运算符。显然，这很可能违反最小惊讶原则，因此在可能和适当的情况下，最好选择行为与正常 Rust 一致的宏。（另一方面，如果宏的目的是允许奇怪的控制流，那就去做吧！但要确保控制流行为有明确的记录，以帮助您的用户。）

例如，考虑一个宏（用于检查 HTTP 状态代码），它在其主体中默默地包含一个返回：

```rust
/// Check that an HTTP status is successful; exit function if not.
macro_rules! check_successful {
    { $e:expr } => {
        if $e.group() != Group::Successful {
            return Err(MyError("HTTP operation failed"));
        }
    }
}
```

使用此宏检查某种 HTTP 操作的结果的代码最终可能会产生有些模糊的控制流：

```rust
let rc = perform_http_operation();
check_successful!(rc); // may silently exit the function

// ...
```

生成发出结果的代码的宏的替代版本：

```rust
/// Convert an HTTP status into a `Result<(), MyError>` indicating success.
macro_rules! check_success {
    { $e:expr } => {
        match $e.group() {
            Group::Successful => Ok(()),
            _ => Err(MyError("HTTP operation failed")),
        }
    }
}
```

给出更容易理解的代码：

```rust
let rc = perform_http_operation();
check_success!(rc)?; // error flow is visible via `?`

// ...
```

使用声明式宏时要注意的第二件事是与 C 预处理器共有的一个问题：如果宏的参数是具有副作用的表达式，则要注意不要在宏中重复使用该参数。前面定义的 square! 宏将任意表达式作为参数，然后使用该参数两次，这可能会导致意外：

```rust
let mut x = 1;
let y = square!({
    x += 1;
    x
});
println!("x = {x}, y = {y}");
// output: x = 3, y = 6
```

假设这种行为不是有意的，那么修复该问题的一种方法就是简单地对表达式进行一次求值，然后将结果赋给局部变量：

```rust
macro_rules! square_once {
    { $e:expr } => {
        {
            let x = $e;
            x*x // Note: there's a detail here to be explained later...
        }
    }
}
// output now: x = 2, y = 4
```

另一种选择是不允许将任意表达式作为宏的输入。如果将 expr 语法片段说明符替换为 ident 片段说明符，则宏将只接受标识符作为输入，并且尝试向其输入任意表达式将不再编译。

#### 格式化值

一种常见的声明式宏样式涉及组装一条消息，该消息包含来自代码当前状态的各种值。例如，标准库包括 `format!` 用于组装字符串、`println!` 用于打印到标准输出、`eprintln!` 用于打印到标准错误，等等。文档描述了格式化指令的语法，它们大致相当于 C 的 `printf` 语句。但是，格式参数是类型安全的，并在编译时进行检查，并且宏的实现使用第 10 条中描述的 `Display` 和 `Debug` 特征来格式化单个值。

您可以（并且应该）对任何执行类似功能的宏使用相同的格式化语法。例如，`log` crate 提供的日志记录宏使用与 `format!` 相同的语法。为此，请对执行参数格式化的宏使用 `format_args!`，而不是尝试重新发明轮子：

```rust
/// Log an error including code location, with `format!`-like arguments.
/// Real code would probably use the `log` crate.
macro_rules! my_log {
    { $($arg:tt)+ } => {
        eprintln!("{}:{}: {}", file!(), line!(), format_args!($($arg)+));
    }
}
```
```rust
let x = 10u8;
// Format specifiers:
// - `x` says print as hex
// - `#` says prefix with '0x'
// - `04` says add leading zeroes so width is at least 4
//   (this includes the '0x' prefix).
my_log!("x = {:#04x}", x);
```
```rust
src/main.rs:331: x = 0x0a
```

### 过程宏

Rust 还支持过程宏，通常称为 `proc` 宏。与声明性宏一样，过程宏能够将任意 Rust 代码插入程序的源代码中。但是，宏的输入不再只是传递给它的特定参数；相反，过程宏可以访问与原始源代码的某些块相对应的解析标记。这提供了接近动态语言（如 Lisp）灵活性的表达能力水平——但仍然具有编译时保证。它还有助于减轻 Rust 中反射的限制，如第 19 项所述。

过程宏必须在使用它们的单独包（包类型为 `proc-macro`）中定义，并且该包几乎肯定需要依赖 `proc-macro`（由标准工具链提供）或 `proc-macro2`（由 David Tolnay 提供）作为支持库，以便能够使用输入标记。

过程宏有三种不同的类型：

* 函数类宏：使用参数调用
* 属性宏：附加到程序中的一些语法块
* 派生宏：附加到数据结构的定义

#### 函数式宏

函数式宏使用参数调用，宏定义可以访问组成参数的解析标记，并发出任意标记作为结果。请注意，上一句说的是“参数”，单数——即使函数式宏使用看起来像多个参数的参数调用：

```rust
my_func_macro!(15, x + y, f32::consts::PI);
```

宏本身接收单个参数，即解析后的标记流。宏实现仅在编译时打印流的内容：

```rust
use proc_macro::TokenStream;

// Function-like macro that just prints (at compile time) its input stream.
#[proc_macro]
pub fn my_func_macro(args: TokenStream) -> TokenStream {
    println!("Input TokenStream is:");
    for tt in args {
        println!("  {tt:?}");
    }
    // Return an empty token stream to replace the macro invocation with.
    TokenStream::new()
}
```

显示与输入对应的流：

```rust
Input TokenStream is:
  Literal { kind: Integer, symbol: "15", suffix: None,
            span: #0 bytes(10976..10978) }
  Punct { ch: ',', spacing: Alone, span: #0 bytes(10978..10979) }
  Ident { ident: "x", span: #0 bytes(10980..10981) }
  Punct { ch: '+', spacing: Alone, span: #0 bytes(10982..10983) }
  Ident { ident: "y", span: #0 bytes(10984..10985) }
  Punct { ch: ',', spacing: Alone, span: #0 bytes(10985..10986) }
  Ident { ident: "f32", span: #0 bytes(10987..10990) }
  Punct { ch: ':', spacing: Joint, span: #0 bytes(10990..10991) }
  Punct { ch: ':', spacing: Alone, span: #0 bytes(10991..10992) }
  Ident { ident: "consts", span: #0 bytes(10992..10998) }
  Punct { ch: ':', spacing: Joint, span: #0 bytes(10998..10999) }
  Punct { ch: ':', spacing: Alone, span: #0 bytes(10999..11000) }
  Ident { ident: "PI", span: #0 bytes(11000..11002) }
```

此输入流的低级性质意味着宏实现必须自行进行解析。例如，分离出看似单独的宏参数需要查找包含分隔参数的逗号的 `TokenTree::Punct` 标记。`syn` crate（来自 David Tolnay）提供了一个解析库，可以帮助实现这一点，如下一节所述。

因此，使用声明性宏通常比使用类似函数的过程宏更容易，因为宏输入的预期结构可以在匹配模式中表达。

这种手动处理需求的另一面是，类似函数的过程宏可以灵活地接受不能解析为普通 Rust 代码的输入。这通常不需要（或不合理），因此类似函数的宏相对较少。

#### 属性宏

属性宏的调用方式是将它们放在程序中某个项之前，而该项的解析标记是宏的输入。宏可以再次发出任意标记作为输出，但输出通常是输入的某种转换。

例如，属性宏可用于包装函数的主体：

```rust
#[log_invocation]
fn add_three(x: u32) -> u32 {
    x + 3
}
```

以便记录函数的调用：

```rust
let x = 2;
let y = add_three(x);
println!("add_three({x}) = {y}");
```
```rust
log: calling function 'add_three'
log: called function 'add_three' => 5
add_three(2) = 5
```

这个宏的实现太大，无法包含在这里，因为代码需要检查输入标记的结构并构建新的输出标记，但 `sync` crate 可以再次帮助完成这个处理。

#### 派生宏

最后一种程序宏是派生宏，它允许生成的代码自动附加到数据结构定义（结构、枚举或联合）。这类似于属性宏，但需要注意一些派生特定方面。

首先，派生宏会添加到输入标记中，而不是完全替换它们。这意味着数据结构定义保持不变，但宏有机会附加相关代码。

第二，派生宏可以声明相关的辅助属性，然后可以使用这些属性标记标记需要特殊处理的数据结构部分。例如，`serde` 的 `Deserialize` 派生宏有一个 `serde` 辅助属性，可以提供元数据来指导反序列化过程：

```rust
fn generate_value() -> String {
    "unknown".to_string()
}

#[derive(Debug, Deserialize)]
struct MyData {
    // If `value` is missing when deserializing, invoke
    // `generate_value()` to populate the field instead.
    #[serde(default = "generate_value")]
    value: String,
}
```

需要注意的派生宏的最后一个方面是，`sync` crate 可以处理将输入标记解析为 AST 中的等效节点所涉及的大部分繁重工作。`syn::parse_macro_input!` 宏将标记转换为描述项目内容的 `syn::DeriveInput` 数据结构，而 `DeriveInput` 比原始标记流更容易处理。

在实践中，派生宏是最常见的过程宏类型 - 能够逐个字段（对于结构）或逐个变量（对于枚举）生成实现，这使得程序员几乎不费吹灰之力就可以提供许多功能 - 例如，通过添加一行代码，如 `#[derive(Debug, Clone, PartialEq, Eq)]`。

由于派生实现是自动生成的，因此这也意味着实现会自动与数据结构定义保持同步。例如，如果要向结构添加新字段，则需要手动更新 Debug 的手动实现，而自动派生版本将无需额外努力即可显示新字段（如果不可能，则编译失败）。

### 何时使用宏

使用宏的主要原因是避免重复代码 — 尤其是那些必须手动与代码的其他部分保持同步的重复代码。在这方面，编写宏只是编程中通常使用的同一种泛化过程的扩展：

* 如果您对特定类型的多个值重复完全相同的代码，请将该代码封装到一个通用函数中，并从所有重复的位置调用该函数。
* 如果您对多种类型重复完全相同的代码，请将该代码封装到具有特征绑定的泛型中，并从所有重复的位置使用该泛型。
* 如果您在多个位置重复相同结构的代码，请将该代码封装到宏中，并从所有重复的位置使用该宏。

例如，只有通过宏才能避免对不同枚举变量起作用的代码重复：

```rust
enum Multi {
    Byte(u8),
    Int(i32),
    Str(String),
}

/// Extract copies of all the values of a specific enum variant.
#[macro_export]
macro_rules! values_of_type {
    { $values:expr, $variant:ident } => {
        {
            let mut result = Vec::new();
            for val in $values {
                if let Multi::$variant(v) = val {
                    result.push(v.clone());
                }
            }
            result
        }
    }
}

fn main() {
    let values = vec![
        Multi::Byte(1),
        Multi::Int(1000),
        Multi::Str("a string".to_string()),
        Multi::Byte(2),
    ];

    let ints = values_of_type!(&values, Int);
    println!("Integer values: {ints:?}");

    let bytes = values_of_type!(&values, Byte);
    println!("Byte values: {bytes:?}");

    // Output:
    //   Integer values: [1000]
    //   Byte values: [1, 2]
}
```

宏有助于避免手动重复的另一种情况是，当有关数据值集合的信息原本会分散在代码的不同区域时。

例如，考虑一个对有关 HTTP 状态代码的信息进行编码的数据结构；宏可以帮助将所有相关信息放在一起：

```rust
// http.rs module

#[derive(Debug, PartialEq, Eq, Clone, Copy)]
pub enum Group {
    Informational, // 1xx
    Successful,    // 2xx
    Redirection,   // 3xx
    ClientError,   // 4xx
    ServerError,   // 5xx
}

// Information about HTTP response codes.
http_codes! {
    Continue           => (100, Informational, "Continue"),
    SwitchingProtocols => (101, Informational, "Switching Protocols"),
    // ...
    Ok                 => (200, Successful, "Ok"),
    Created            => (201, Successful, "Created"),
    // ...
}
```

宏调用保存每个 HTTP 状态代码的所有相关信息（数值、组、描述），充当一种领域特定语言 (DSL)，保存数据的真实来源。

然后，宏定义描述生成的代码；形式为 $( ... )+ 的每一行都会扩展为生成的代码中的多行，每个宏参数一行：

```rust
macro_rules! http_codes {
    { $( $name:ident => ($val:literal, $group:ident, $text:literal), )+ } => {
        #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
        #[repr(i32)]
        enum Status {
            $( $name = $val, )+
        }
        impl Status {
            fn group(&self) -> Group {
                match self {
                    $( Self::$name => Group::$group, )+
                }
            }
            fn text(&self) -> &'static str {
                match self {
                    $( Self::$name => $text, )+
                }
            }
        }
        impl core::convert::TryFrom<i32> for Status {
            type Error = ();
            fn try_from(v: i32) -> Result<Self, Self::Error> {
                match v {
                    $( $val => Ok(Self::$name), )+
                    _ => Err(())
                }
            }
        }
    }
}
```

因此，宏的整体输出负责生成从真实源值派生的所有代码：

* 包含所有变体的枚举的定义
* `group()` 方法的定义，指示 HTTP 状态属于哪个组
* `text()` 方法的定义，将状态映射到文本描述
* `TryFrom<i32>` 的实现，用于将数字转换为状态枚举值

如果稍后需要添加额外的值，则只需添加一行：

```rust
ImATeapot => (418, ClientError, "I'm a teapot"),
```

如果没有该宏，则必须手动更新四个不同位置。编译器会指出其中一些（因为匹配表达式需要涵盖所有情况），但不会指出全部 — `TryFrom<i32>` 很容易被遗忘。

由于宏在调用代码中就地展开，因此它们还可用于自动发出其他诊断信息 — 特别是通过使用标准库的 `file!()` 和 `line!()` 宏，它们会发出源代码位置信息：

```rust
macro_rules! log_failure {
    { $e:expr } => {
        {
            let result = $e;
            if let Err(err) = &result {
                eprintln!("{}:{}: operation '{}' failed: {:?}",
                          file!(),
                          line!(),
                          stringify!($e),
                          err);
            }
            result
        }
    }
}
```

发生故障时，日志文件会自动包含有关发生故障的原因和位置的详细信息：

```rust
use std::convert::TryInto;

let x: Result<u8, _> = log_failure!(512.try_into()); // too big for `u8`
let y = log_failure!(std::str::from_utf8(b"\xc3\x28")); // invalid UTF-8
```
```rust
src/main.rs:340: operation '512.try_into()' failed: TryFromIntError(())
src/main.rs:341: operation 'std::str::from_utf8(b"\xc3\x28")' failed:
                 Utf8Error { valid_up_to: 0, error_len: Some(1) }
```

### 宏的缺点

使用宏的主要缺点是它对代码的可读性和可维护性的影响。前面的“声明性宏”部分解释了宏允许您创建 DSL 来简洁地表达代码和数据的关键特性。然而，这意味着现在任何阅读或维护代码的人除了了解 Rust 之外，还必须了解这个 DSL——以及它在宏定义中的实现。例如，上一节中的 `http_codes!` 示例创建了一个名为 `Status` 的 Rust 枚举，但它在用于宏调用的 DSL 中不可见。

基于宏的代码的这种潜在的不可渗透性超出了其他工程师的范围：分析和与 Rust 代码交互的各种工具可能会将代码视为不透明的，因为它不再遵循 Rust 代码的语法约定。前面显示的 `square_once!` 宏提供了一个简单的例子：宏的主体没有按照正常的 rustfmt 规则进行格式化：

```rust
{
    let x = $e;
    // The `rustfmt` tool doesn't really cope with code in
    // macros, so this has not been reformatted to `x * x`.
    x*x
}
```

另一个例子是早期的 `http_codes!` 宏，其中 DSL 使用 `Group` 枚举变体名称（如 `Informational`），既没有 `Group::` 前缀，也没有 `use` 语句，这可能会使某些代码导航工具感到困惑。

甚至编译器本身也没什么帮助：它的错误消息并不总是遵循宏使用和定义的链条。（但是，工具生态系统 [参见第 31 条] 的某些部分可以帮助解决这个问题，例如之前使用的 David Tolnay 的 `cargo-expand`。）

使用宏的另一个可能的缺点是代码膨胀的可能性——一行宏调用可能会导致生成数百行代码，这些代码对于粗略查看代码来说是不可见的。当代码首次编写时，这不太可能成为问题，因为那时需要代码，并且可以节省相关人员自己编写代码的时间。但是，如果代码随后不再需要，则可能删除大量代码并不那么明显。

### 建议

尽管上一节列出了宏的一些缺点，但当需要保持一致但无法以其他方式合并的不同代码块时，宏仍然是完成这项工作的正确工具：当只有宏才能确保不同的代码保持同步时，请使用宏。

当需要压缩样板代码时，宏也是可以使用的工具：对于无法合并为函数或泛型的重复样板代码，请使用宏。

为了减少对可读性的影响，请尽量避免宏中的语法与 Rust 的正常语法规则相冲突；要么让宏调用看起来像正常代码，要么让它看起来足够不同，这样就不会有人混淆这两者。特别是，请遵循以下准则：

* 尽可能避免插入引用的宏扩展 - 像 `my_macro!(&list)` 这样的宏调用比 `my_macro!(list)` 更符合正常的 Rust 代码。
* 尽量避免在宏中使用非局部控制流操作，这样任何阅读代码的人都可以跟踪流程，而无需了解宏的细节。

这种对 Rust 式可读性的偏好有时会影响声明性宏和过程宏之间的选择。如果您需要为结构的每个字段或枚举的每个变体发出代码，则最好使用派生宏而不是发出类型的过程宏（尽管前面部分显示了示例）——它更符合惯用语，并使代码更易于阅读。

但是，如果您要添加具有非特定于项目的功能的派生宏，请检查外部包是否已经提供了您所需的功能（参见第 25 条）。例如，将整数值转换为类似 C 的枚举的适当变体的问题已经得到很好的解决：[`enumn::N`](https://docs.rs/enumn/latest/enumn/derive.N.html)、[`num_enum::TryFromPrimitive`](https://docs.rs/num_enum/latest/num_enum/derive.TryFromPrimitive.html)、[`num_derive::FromPrimitive`](https://docs.rs/num-derive/latest/num_derive/derive.FromPrimitive.html) 和 [`strum::FromRepr`](https://docs.rs/strum/latest/strum/derive.FromRepr.html) 都涵盖了这个问题的某些方面。



