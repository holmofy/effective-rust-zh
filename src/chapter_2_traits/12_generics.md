## 理解泛型和特征对象之间的权衡

第 2 项描述了使用特征将行为封装在类型系统中，作为相关方法的集合，并观察到有两种使用特征的方法：作为泛型的特征边界或在特征对象中。本项探讨了这两种可能性之间的权衡。

作为一个运行示例，考虑一个涵盖显示图形对象功能的特征：

```rust
#[derive(Debug, Copy, Clone)]
pub struct Point {
    x: i64,
    y: i64,
}

#[derive(Debug, Copy, Clone)]
pub struct Bounds {
    top_left: Point,
    bottom_right: Point,
}

/// Calculate the overlap between two rectangles, or `None` if there is no
/// overlap.
fn overlap(a: Bounds, b: Bounds) -> Option<Bounds> {
    // ...
}

/// Trait for objects that can be drawn graphically.
pub trait Draw {
    /// Return the bounding rectangle that encompasses the object.
    fn bounds(&self) -> Bounds;

    // ...
}
```

## 范型

Rust 的泛型大致相当于 C++ 的模板：它们允许程序员编写适用于任意类型 T 的代码，泛型代码的具体用途在编译时生成 - 这一过程在 Rust 中称为单态化，在 C++ 中称为模板实例化。与 C++ 不同，Rust 以泛型的特征边界形式在类型系统中明确编码对类型 T 的期望。

例如，使用特征的 bounds() 方法的泛型函数具有显式 Draw 特征边界：

```rust
/// Indicate whether an object is on-screen.
pub fn on_screen<T>(draw: &T) -> bool
where
    T: Draw,
{
    overlap(SCREEN_BOUNDS, draw.bounds()).is_some()
}
```

也可以通过将特征绑定放在泛型参数之后来更紧凑地编写此代码：

```rust
pub fn on_screen<T: Draw>(draw: &T) -> bool {
    overlap(SCREEN_BOUNDS, draw.bounds()).is_some()
}
```

或者使用 impl Trait 作为参数的类型：

```rust
pub fn on_screen(draw: &impl Draw) -> bool {
    overlap(SCREEN_BOUNDS, draw.bounds()).is_some()
}
```

如果类型实现了该特征：

```rust
#[derive(Clone)] // no `Debug`
struct Square {
    top_left: Point,
    size: i64,
}

impl Draw for Square {
    fn bounds(&self) -> Bounds {
        Bounds {
            top_left: self.top_left,
            bottom_right: Point {
                x: self.top_left.x + self.size,
                y: self.top_left.y + self.size,
            },
        }
    }
}
```

然后可以将该类型的实例传递给泛型函数，将其单态化以生成特定于某一特定类型的代码：

```rust
let square = Square {
    top_left: Point { x: 1, y: 2 },
    size: 2,
};
// Calls `on_screen::<Square>(&Square) -> bool`
let visible = on_screen(&square);
```

如果将相同的泛型函数与实现相关特征绑定的不同类型一起使用：

```rust
#[derive(Clone, Debug)]
struct Circle {
    center: Point,
    radius: i64,
}

impl Draw for Circle {
    fn bounds(&self) -> Bounds {
        // ...
    }
}
```

然后使用不同的单态代码：

```rust
let circle = Circle {
    center: Point { x: 3, y: 4 },
    radius: 1,
};
// Calls `on_screen::<Circle>(&Circle) -> bool`
let visible = on_screen(&circle);
```

换句话说，程序员编写一个通用函数，但编译器会针对调用该函数的每种不同类型输出该函数的不同单态版本。

## Trait Objects

相比之下，特征对象是胖指针（项目 8），它将指向底层具体项的指针与指向 vtable 的指针结合在一起，而 vtable 又保存了所有特征实现方法的函数指针，如图 2-1 所示：

```rust
let square = Square {
    top_left: Point { x: 1, y: 2 },
    size: 2,
};
let draw: &dyn Draw = &square;
```

![trait object内存布局](https://effective-rust.com/images/draw.svg)

这意味着接受特征对象的函数不需要是通用的，也不需要单态化：程序员使用特征对象编写函数，编译器只输出该函数的单个版本，该版本可以接受来自多种输入类型的特征对象：

```rust
/// Indicate whether an object is on-screen.
pub fn on_screen(draw: &dyn Draw) -> bool {
    overlap(SCREEN_BOUNDS, draw.bounds()).is_some()
}
```

```rust
// Calls `on_screen(&dyn Draw) -> bool`.
let visible = on_screen(&square);
// Also calls `on_screen(&dyn Draw) -> bool`.
let visible = on_screen(&circle);
```

## 基本比较

这些基本事实已经允许对两种可能性进行一些直接比较：

* 泛型可能会导致更大的代码大小，因为编译器会为使用 `on_screen` 函数的泛型版本的每个类型 `T` 生成代码的新副本 (`on_screen::<T>(&T)`)。相比之下，函数的特征对象版本 (`on_screen(&dyn T)`) 只需要一个实例。
* 从泛型调用特征方法通常比从使用特征对象的代码调用它要快一点，因为后者需要执行两次取消引用才能找到代码的位置（特征对象到 vtable，vtable 到实现位置）。
* 泛型的编译时间可能会更长，因为编译器正在构建更多代码，并且链接器需要做更多工作来折叠重复项。
在大多数情况下，这些并不是显著的差异——只有当您衡量了影响并发现它确实有效果（速度瓶颈或有问题的占用率增加）时，您才应该将与优化相关的问题作为主要决策驱动因素。

更重要的差异是，通用特征边界可用于有条件地提供不同的功能，具体取决于类型参数是否实现多个特征：

```rust
// The `area` function is available for all containers holding things
// that implement `Draw`.
fn area<T>(draw: &T) -> i64
where
    T: Draw,
{
    let bounds = draw.bounds();
    (bounds.bottom_right.x - bounds.top_left.x)
        * (bounds.bottom_right.y - bounds.top_left.y)
}

// The `show` method is available only if `Debug` is also implemented.
fn show<T>(draw: &T)
where
    T: Debug + Draw,
{
    println!("{:?} has bounds {:?}", draw, draw.bounds());
}
```

```rust
let square = Square {
    top_left: Point { x: 1, y: 2 },
    size: 2,
};
let circle = Circle {
    center: Point { x: 3, y: 4 },
    radius: 1,
};

// Both `Square` and `Circle` implement `Draw`.
println!("area(square) = {}", area(&square));
println!("area(circle) = {}", area(&circle));

// `Circle` implements `Debug`.
show(&circle);

// `Square` does not implement `Debug`, so this wouldn't compile:
// show(&square);
```

特征对象仅为单个特征编码实现 vtable，因此执行等效操作会更加尴尬。例如，可以为 show() 案例定义组合 DebugDraw 特征，并结合一个总体实现以使生活更轻松：

```rust
trait DebugDraw: Debug + Draw {}

/// Blanket implementation applies whenever the individual traits
/// are implemented.
impl<T: Debug + Draw> DebugDraw for T {}
```

然而，如果存在多种不同特征的组合，那么这种方法的组合显然很快就会变得难以处理。

## 更多特征界限

除了使用特征界限来限制泛型函数可接受的类型参数外，您还可以将它们应用于特征定义本身：

```rust
/// Anything that implements `Shape` must also implement `Draw`.
trait Shape: Draw {
    /// Render that portion of the shape that falls within `bounds`.
    fn render_in(&self, bounds: Bounds);

    /// Render the shape.
    fn render(&self) {
        // Default implementation renders that portion of the shape
        // that falls within the screen area.
        if let Some(visible) = overlap(SCREEN_BOUNDS, self.bounds()) {
            self.render_in(visible);
        }
    }
}
```

在此示例中，`render()` 方法的默认实现（条目 13）利用了特征边界，依赖于 `Draw` 中 `bounds()` 方法的可用性。

来自面向对象语言的程序员经常将特征边界与继承相混淆，误以为这样的特征边界意味着 `Shape` 是 `Draw`。事实并非如此：这两种类型之间的关系最好表达为 `Shape` 也实现了 `Draw`。

在幕后，具有特征边界的特征的特征对象：

```rust
let square = Square {
    top_left: Point { x: 1, y: 2 },
    size: 2,
};
let draw: &dyn Draw = &square;
let shape: &dyn Shape = &square;
```

具有单个组合 vtable，其中包含顶级特征的方法以及所有特征边界的方法。如图 2-2 所示：`Shape` 的 vtable 包含来自 `Draw` 特征的边界方法，以及来自 `Shape` 特征本身的两种方法。

![特征边界的特征对象，具有 Draw 和 Shape 的不同 vtable](https://effective-rust.com/images/traitbounds.svg)

在撰写本文时（以及从 Rust 1.70 开始），这意味着无法从 `Shape` “上溯” 到 `Draw`，因为（纯）`Draw` vtable 无法在运行时恢复；无法在相关特征对象之间进行转换，这反过来意味着没有 [Liskov 李氏替换](https://en.wikipedia.org/wiki/Liskov_substitution_principless)。但是，这在 Rust 的后续版本中可能会发生变化——有关更多信息，请参阅第 19 条。

用不同的词重复同一点，接受 `Shape` 特征对象的方法具有以下特征：

* 它可以使用 `Draw` 中的方法（因为 `Shape` 也实现了 `Draw`，并且相关函数指针存在于 `Shape` vtable 中）。
* 它不能（暂时）将特征对象传递给另一个需要 `Draw` 特征对象的方法（因为 `Shape` 不是 `Draw`，并且 `Draw` vtable 不可用）。
相反，接受实现 `Shape` 的项目的泛型方法具有以下特征：

* 它可以使用 `Draw` 中的方法。
* 它可以将该项目传递给具有 `Draw` 特征绑定的另一个泛型方法，因为特征绑定在编译时被单态化以使用具体类型的 `Draw` 方法。

## 特征对象安全
特征对象的另一个限制是对象安全要求：只有符合以下两个规则的特征才能用作特征对象：

* 特征的方法不得是通用的。
* 特征的方法不得涉及包含 `Self` 的类型，接收者（调用该方法的对象）除外。
第一个限制很容易理解：通用方法 `f` 实际上是一组无限的方法，可能包含 `f::<i16>`、`f::<i32>`、`f::<i64>`、`f::<u8>` 等。另一方面，特征对象的 vtable 在很大程度上是函数指针的有限集合，因此不可能将无限的单态化实现集合放入其中。

第二个限制稍微微妙一些，但往往是实践中更常见的限制——施加 `Copy` 或 `Clone` 特征界限（第 10 条）的特征立即属于此规则，因为它们返回 `Self`。要了解为什么不允许这样做，请考虑拥有特征对象的代码；如果该代码调用（例如） `let y = x.clone()`，会发生什么？调用代码需要在堆栈上为 `y` 保留足够的空间，但它不知道 `y` 的大小，因为 `Self` 是任意类型。因此，返回提及 `Self` 的类型会导致不安全的特征。

第二个限制有一个例外。如果 `Self` 带有对编译时已知大小类型的明确限制，则返回某些 `Self` 相关类型的方法不会影响对象安全，由 `Sized` 标记特征指示为特征界限：

```rust
/// A `Stamp` can be copied and drawn multiple times.
trait Stamp: Draw {
    fn make_copy(&self) -> Self
    where
        Self: Sized;
}
```
```rust
let square = Square {
    top_left: Point { x: 1, y: 2 },
    size: 2,
};

// `Square` implements `Stamp`, so it can call `make_copy()`.
let copy = square.make_copy();

// Because the `Self`-returning method has a `Sized` trait bound,
// creating a `Stamp` trait object is possible.
let stamp: &dyn Stamp = &square;
```

这个特征边界意味着该方法无论如何都不能与特征对象一起使用，因为特征对象引用的是未知大小的东西（`dyn Trait`），所以该方法与对象安全无关：

```rust
// However, the method can't be invoked via a trait object.
let copy = stamp.make_copy();
```

```rust
error: the `make_copy` method cannot be invoked on a trait object
   --> src/main.rs:397:22
    |
353 |         Self: Sized;
    |               ----- this has a `Sized` requirement
...
397 |     let copy = stamp.make_copy();
    |                      ^^^^^^^^^
```

## 权衡
到目前为止，各种因素的平衡表明，你应该更喜欢泛型而不是特征对象，但在某些情况下，特征对象是完成这项工作的正确工具。

首先是实际考虑：如果生成的代码大小或编译时间是一个问题，那么特征对象将表现更好（如本条目前面所述）。

导致特征对象的一个​​更理论化的方面是，它们从根本上涉及类型擦除：在转换为特征对象时，有关具体类型的信息会丢失。这可能是一个缺点（参见条目 19），但它也很有用，因为它允许异构对象的集合——因为代码只依赖于特征的方法，它可以调用和组合具有不同具体类型的项目的方法。

渲染形状列表的传统 OO 示例就是一个例子：同一个 `render()` 方法可以用于同一个循环中的正方形、圆形、椭圆形和星形：

```rust
let shapes: Vec<&dyn Shape> = vec![&square, &circle];
for shape in shapes {
    shape.render()
}
```

特征对象还有一个更不为人知的潜在优势，即在编译时无法获知可用类型。如果在运行时动态加载新代码（例如通过 `dlopen(3)`），那么新代码中实现特征的项目只能通过特征对象来调用，因为没有源代码可以进行单态化。