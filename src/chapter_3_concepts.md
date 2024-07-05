## 概念

本书的前两章介绍了 Rust 的类型和traits，这有助于提供处理编写 Rust 代码所涉及的一些概念所需的词汇——本章的主题。

借用检查器和生命周期检查是 Rust 核心的独特之处；它们也是 Rust 新手常见的绊脚石，因此也是本章前两个条款的重点。

本章中的其他项目涵盖了更容易掌握的概念，但与用其他语言编写代码略有不同。其中包括：

* 关于 Rust 不安全模式的建议以及如何避免它（[条款16](/chapter_3_concepts/16_unsafe.md)）
* 关于用 Rust 编写多线程代码的好消息和坏消息（[条款17](/chapter_3_concepts/17_deadlock.md)）
* 关于避免运行时中止的建议（[条款18](/chapter_3_concepts/18_panic.md)）
* 关于 Rust 反射方法的信息（[项目19](/chapter_3_concepts/19_reflection.md)）
* 关于平衡优化和可维护性的建议（[项目20](/chapter_3_concepts/20_optimize.md)）

尝试将你的代码与这些概念的后果保持一致是一个好主意。可以在 Rust 中重新创建（部分） C/C++ 的行为，但如果这样做，为什么还要使用 Rust 呢？
