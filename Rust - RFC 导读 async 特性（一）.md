# Rust RFC 导读:  `async/await` 特性 (一)

在 2018 年五月份 Rust 迎来了 1.26 版本，一并带来不少让人眼前一亮的特性，比如 `impl trait`，`match` 自动绑定等等，这是 Rust Core Team 和社区在有条不紊地履行着 2018 Roadmap 里的承诺，给 2018 epoch 打下基础，根据 Roadmap，今年9月份前我们将迎来一个更重磅的特性，`async/await`。Core team如期给出了两份 RFC，给 `async/await` 特性画下个大饼。这篇文章旨在提供对这两份 RFC 的导读，让读者掌握 `async` 特性的基本概念。

[RFC 2394-async_await](https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md)

[RFC 2418 companion libs](https://github.com/aturon/rfcs/blob/async-trait/text/0000-async.md)

## Summary

在高性能网络服务领域，人们常用异步 IO 而不是阻塞 IO，这是因为异步 IO 更容易扩展从而获得巨大的并发能力，而 Rust 正在逐步涉足网络服务领域，因此能够提供简洁而强大的异步开发能力格外重要。

为此，Rust 社区已经进行了长时间的实验和反馈，尝试了众多的技术方法，社区最终采用 `async/await` ，因为它能够提供强大的抽象能力，而且简洁，容易学习，是目前的最优方案。



由于这次特性更新影响重大，涉及方面较广，文章将随 RFC 分为两部分，分别为语言特性和标准库两方面。

## 语言特性

本篇 RFC 的重点是为编译器增加四种新的类型：`async function`（异步函数），`async closure`（异步闭包）， `async block`（异步代码块）和 `await!` 内建 macro。

### 异步函数

函数开头加上 `async` 关键词就成为了异步函数。

```rust
async fn function(argument: &str) -> usize {
     // ...
}
```

异步函数的行为和普通函数不同，当异步函数被调用时，内部的代码逻辑不会立即执行，相反，异步函数会返回一个匿名的 `Future` 类型，之后当我们 `poll` 这个 `Future` 的时候，函数内部的代码才会被执行并且执行到 `await` 处停止（如果异步函数内部有的话），直到异步函数结尾。

异步函数其实是某种 delayed computation （延迟运算）—— 在手动 `poll` future 之前，异步函数内部一行代码也不会执行。

看下面这个例子

```rust
async fn print_async() {
     println!("Hello from print_async")
}

fn main() {
     let future = print_async();
     println!("Hello from main");
     futures::block_on(future);
}
```

Print:
```
Hello from main
Hello from print_async
```
`"Hello from main"` 会在 `"Hello from print_async"` 之前 print 出来。

异步函数的类型签名也与普通函数不同

`async fn foo(args..) -> T` 的实际类型签名是 `fn(args..) -> impl Future<Output = T>`，其中的返回的是由编译器生成的匿名类型。


### 异步闭包

与函数类似，闭包也可以声明为异步闭包，只需在闭包前加上 `async` 关键字。

异步闭包的返回类型是 `impl Future<Output = T>`，调用异步闭包时，内部代码不会被执行而是返回一个 `Future`，与异步函数一模一样。

```rust
fn main() {
    let closure = async || {
         println("Hello from async closure.");
    };
    println!("Hello from main");
    let future = closure();
    println!("Hello from main again");
    futures::block_on(future);
}
```

Print:
```
Hello from main
Hello from main again
Hello from async closure
```

### 异步块

通过异步块可以便捷地创建一个 `Future`:

```rust
let my_future = async {
    println!("Hello from an async block");
};
```

### `await!`

`await!` 是一个编译器内建的 macro ，用来暂停（pause） `Future` 的执行流程，并且把执行流程交回给调用方 (`yield`)。

`await!` 只能传入 `IntoFuture` 类型。

```rust
// future: impl Future<Output = usize>
let n = await!(future);
```

`await!` 展开的逻辑是这样的：
1. `poll` 传入的 `future`。
2. 如果 `poll` 得到 `Poll::Pending` 就将执行权交回给调用方。
3. 如果 `poll` 得到 `Poll::Ready(T)`，得到的值会被作为 `await!` 表达式的值，从而继续执行 `future` 剩下的逻辑代码。

`await!` 只能用于异步函数，异步闭包或者异步块中，否则将会是编译错误。

## Conclusion
`async/await` 为 Rust 提供了强大的异步抽象，它不止可以助力网络并发，它在文件IO，多线程运算方面也大有作为。`async` 所涉及的 `Generator` 还可用于简化 `Iterator` 代码，写法更加接近于 Python 等脚本语言，同时保持 Rust 引以为豪的 Zero-Cost-Abstration。 

下一篇将着力介绍为了迎接 `async/await`，标准库要加入的新朋友 `Executor`, `Pin` 和 `Async`。