## 条款11：实现RAII模式的Drop trait

> “永远不要派人去做机器的工作。”——Agent Smith

RAII 代表“资源获取即初始化”，这是一种编程模式，其中值的生命周期与某些附加资源的生命周期完全相关。RAII 模式由 C++ 编程语言推广，是 C++ 对编程的最大贡献之一。

值的生命周期与资源的生命周期之间的相关性编码在 RAII 类型中：

* 类型的构造函数获取对某些资源的访问权限
* 类型的析构函数释放对该资源的访问权限

结果是 RAII 类型具有不变量：当且仅当项存在时，才可以访问底层资源。由于编译器确保在范围退出时销毁局部变量，这反过来意味着在范围退出时也会释放底层资源。

这对于可维护性特别有用：如果对代码的后续更改改变了控制流，则项和资源的生命周期仍然正确。要了解这一点，请考虑一些手动锁定和解锁互斥体的代码，而不使用 RAII 模式；该代码是用 C++ 编写的，因为 Rust 的 Mutex 不允许这种容易出错的使用！

```cpp
// C++ code
class ThreadSafeInt {
 public:
  ThreadSafeInt(int v) : value_(v) {}

  void add(int delta) {
    mu_.lock();
    // ... more code here
    value_ += delta;
    // ... more code here
    mu_.unlock();
  }
```

通过提前退出来捕获错误情况的修改会使互斥锁保持锁定状态：

```cpp
// C++ code
void add_with_modification(int delta) {
  mu_.lock();
  // ... more code here
  value_ += delta;
  // Check for overflow.
  if (value_ > MAX_INT) {
    // Oops, forgot to unlock() before exit
    return;
  }
  // ... more code here
  mu_.unlock();
}
```

但是，将锁定行为封装到 RAII 类中：

```cpp
// C++ code (real code should use std::lock_guard or similar)
class MutexLock {
 public:
  MutexLock(Mutex* mu) : mu_(mu) { mu_->lock(); }
  ~MutexLock()                   { mu_->unlock(); }
 private:
  Mutex* mu_;
};
```

意味着等效代码对于这种修改是安全的：

```cpp
// C++ code
void add_with_modification(int delta) {
  MutexLock with_lock(&mu_);
  // ... more code here
  value_ += delta;
  // Check for overflow.
  if (value_ > MAX_INT) {
    return; // Safe, with_lock unlocks on the way out
  }
  // ... more code here
}
```

在 C++ 中，RAII 模式最初通常用于内存管理，以确保手动分配（`new`、`malloc()`）和释放（`delete`、`free()`）操作保持同步。此内存管理的通用版本已添加到 C++11 中的 C++ 标准库中：`std::unique_ptr<T>` 类型确保单个位置具有内存的“所有权”，但允许“借用”指向内存的指针以供临时使用（`ptr.get()`）。

在 Rust 中，内存指针的这种行为内置于语言中（第 15 条），但 RAII 的一般原则对于其他类型的资源仍然有用。为任何包含必须释放的资源的类型实现 `Drop`，例如：

* 访问操作系统资源。对于 Unix 派生的系统，这通常意味着包含文件描述符的东西；如果无法正确释放这些资源，则会占用系统资源（并且最终还会导致程序达到每个进程的文件描述符限制）。
* 访问同步资源。标准库已经包含内存同步原语，但其他资源（例如文件锁、数据库锁等）可能需要类似的封装。
* 对于处理低级内存管理的不安全类型（例如，用于外部函数接口 [FFI] 功能），可以访问原始内存。

Rust 标准库中最明显的 RAII 实例是 `Mutex::lock()` 操作返回的 [`MutexGuard`](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html) 项，它往往广泛用于使用第 17 项中讨论的共享状态并行性的程序。这大致类似于前面显示的最后一个 C++ 示例，但在 Rust 中，`MutexGuard` 项除了是持有锁的 RAII 项之外，还充当互斥保护数据的代理：

```rust
use std::sync::Mutex;

struct ThreadSafeInt {
    value: Mutex<i32>,
}

impl ThreadSafeInt {
    fn new(val: i32) -> Self {
        Self {
            value: Mutex::new(val),
        }
    }
    fn add(&self, delta: i32) {
        let mut v = self.value.lock().unwrap();
        *v += delta;
    }
}
```

第 17 条建议不要对大段代码保持锁定；为确保这一点，请使用块来限制 RAII 项的范围。这会导致略微奇怪的缩进，但为了增加安全性和生命周期精度，这是值得的：

```rust
impl ThreadSafeInt {
    fn add_with_extras(&self, delta: i32) {
        // ... more code here that doesn't need the lock
        {
            let mut v = self.value.lock().unwrap();
            *v += delta;
        }
        // ... more code here that doesn't need the lock
    }
}
```

在介绍了 RAII 模式的用途之后，有必要解释一下如何实现它。 `Drop` trait允许您将用户定义的行为添加到项目的销毁中。 此特性只有一个方法 `drop`，编译器会在释放保存该项目的内存之前运行该方法：

```rust
#[derive(Debug)]
struct MyStruct(i32);

impl Drop for MyStruct {
    fn drop(&mut self) {
        println!("Dropping {self:?}");
        // Code to release resources owned by the item would go here.
    }
}
```

drop 方法是专门为编译器保留的，不能手动调用：

```rust
x.drop();
```
```rust
error[E0040]: explicit use of destructor method
  --> src/main.rs:70:7
   |
70 |     x.drop();
   |     --^^^^--
   |     | |
   |     | explicit destructor calls not allowed
   |     help: consider using `drop` function: `drop(x)`
```

值得了解一下这里的技术细节。请注意，`Drop::drop` 方法的签名是 `drop(&mut self)`，而不是 `drop(self)`：它获取对项目的可变引用，而不是将项目移入方法中。如果 `Drop::drop` 像普通方法一样运行，则意味着该项目之后仍可供使用 - 即使其所有内部状态都已整理好并且资源已释放！

```rust
{
    // If calling `drop` were allowed...
    x.drop(); // (does not compile)

    // `x` would still be available afterwards.
    x.0 += 1;
}
// Also, what would happen when `x` goes out of scope?
```

编译器建议了一种简单的替代方案，即调用 `drop()` 函数手动删除一个项目。此函数确实接受一个移动的参数，而 `drop(_item: T)` 的实现只是一个空的主体 `{ }` — 因此当到达该范围的右括号时，移动的项目将被删除。

还请注意，`drop(&mut self)` 方法的签名没有返回类型，这意味着它无法发出失败信号。如果尝试释放资源可能会失败，那么您可能应该有一个单独的 `release` 方法返回结果，这样用户就可以检测到此失败。

无论技术细节如何，`drop` 方法仍然是实现 RAII 模式的关键位置；它的实现是释放与项目相关的资源的理想位置。

