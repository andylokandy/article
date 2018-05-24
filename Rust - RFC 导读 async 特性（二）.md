# Rust RFC 导读:  `async/await` 特性 (二)

[上一篇](https://zhuanlan.zhihu.com/p/37209852)我们讲到了编译器对 `async/await` 的特性支持，相应的，我们要让编译器知道 `Future` 是什么，编译器才能生成匿名的 `Future` 类型。为此，我们就有了第二份 RFC —— 将 `Future` 加入标准库（准确来说是 `libcore`）。

[RFC 2418 companion libs](https://github.com/aturon/rfcs/blob/async-trait/text/0000-async.md)

*注意这篇 RFC 还在讨论阶段并未被 merge，因此最终方案可能会改变，目前已经可以看见讨论区提出了相较于 `arbitrary self type`（`Pin<self>`） 更好的解决方案，这部分是最有可能发生变化的。*

## Summary

这篇文章将会深入 `async/await` 的实现原理，如果读者只是希望使用现成的库和 `async/await` 语法，仅看上一篇文章就已经足够了。而如果你希望开发使用异步实现的库，或者天生有着强烈的好奇心，那么这篇文章就是专门为你定做的。

这篇 RFC 并不打算将整个 `futures-rs` 移入标准库，相反极力精简，仅仅加入必要的最基础的构件，把剩余的功能留给社区的库来实现。就算如此，我们也将迎来帮数量众多的新朋友: `core::task::{Context, Poll, Wake, Waker, UnsafeWake, Executor, TaskObj, SpawnErrorKind, SpawnObjError}` 和 `core::ops::Async`。

如果加上另一篇 [RFC: 2349-pin](https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md)，那就还有 `core::marker::Unpin` 和 `core::mem::Pin`。

我们可以看到，新加入的东西真心不少，但是别担心，它们的存在都是必要而且符合直觉的，接下来我会逐个解释这些东西到底用来干什么的。

另外，读者可能注意到，这篇 RFC 中并没有出现 `Future`，这是因为这篇 RFC 写在上一篇 RFC 之后，在这期间社区讨论决定了并不引入 `Future` 全部的功能(比如 `map()`, `and_then()`)，而是只定义其中关键一部分 `poll()`，剩余的功能依旧留给 `futures-rs` 库来提供，所以这个被精简的 `trait` 不能叫做 `Future`，那就改名叫作了 `Async`。也就是说，async 函数的返回值应该是 `impl Async<Output = T>`。

## `Async`

让我们先从关键的 `Async trait` 入手，下面是它的定义： 

```rust
pub trait Async {
    type Output;
    /// Attempt to resolve the computation to a final value, registering
    /// the current task for wakeup if the value is not yet available.
    fn poll(self: Pin<Self>, cx: &mut task::Context) -> Poll<Self::Output>;
}
```

`Async` 只定义了一个函数 `poll()`，作用是尝试获取异步操作的结果， 熟悉使用 `futures-rs` 的读者想必已经对它十分了解了。如果不是也没关系，我会从头解释一遍。我们先忽略传入参数，只看返回值，它返回的是 `Poll` 类型：

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

`Poll` 有两个 variance: `Ready(T)` 表示异步操作已经完成并获得返回值, `Pendding` 表示异步操作还在进行中，你过会儿再过来看看。`Poll` 的概念很简单，就不多说了。

这就带来了一个问题，我们该什么时候调用 `poll()` 呢？它仅仅在完成之后才能给出有用的结果，万一如果返回的是 `Pending`, 我们怎么知道什么时候需要再次 `poll()` 呢？又由谁来负责 `poll()` 呢？

答案是我们需要一个 `event loop`（比如线程池）来帮助我们完成这个调度(scheduling)工作。标准库并不会提供一个现成的 `event loop`，但标准库提供了定义它们的 `trait`。

## `Executor`

我们来看一个具体的例子，比如说我们正在打算从 `socket` 接收数据，我们得到了一个 `Async`，这时候 `socket` 可能还没准备好，所以第一次 `poll()` 返回了 `Poll::Pending`，这时我们必须把这个 `Async` 交给一个 `event loop` 保管（就比如说一个由 `futures-rs` 库提供的线程池 `ThreadPool`），以备过会 `socket` 准备好了再回来 `poll()`。

在这个例子里 `ThreadPool` 实现了 `Executor`。我们来看看这个东西的定义：

```rust
pub trait Executor {
    fn spawn_obj(&mut self, task: TaskObj) -> Result<(), SpawnObjError>;
}
```

这里的 `TaskObj` 实际上是一个 `Box<Async<Output = ())>>`。

`spawn_obj()` 用于给 `Executor` 安排工作，让 `Executor` 负责稍后的 `poll()` 工作。

这里说个题外话，因为 `TaskObj` 使用了动态派发 （`trait object`），所以目前这里的堆分配是必须的。然而我们还有另一套用于 `no_std` 的 `unsafe` 的解决方案可以避免堆分配，有兴趣的读者可以在原文拉到最后了解一下。在不久的将来，在 Rust 支持在栈上存储 `dynamic size type(DST)` 的时候 [[Merged] RFC 1909:unsized-rvalues](https://github.com/rust-lang/rfcs/blob/master/text/1909-unsized-rvalues.md)，这里的 `TaskObj` 也会取消，取而代之的是栈上的 `Task`，这也是现在取个这么难听名字的原因。

## `Wake`

解决了 who 的问题，那还有 when 问题。我们让 `ThreadPool` 来负责重新 `poll()` ，那到底什么时候 `poll()` 呢？只有 `socket` 知道自己什么时候准备好，所以 `socket` 需要在准备好的时候 “通知” `ThreadPool` 重新 `poll()`。这个操作叫做 `wake` （唤醒），这意味着，我们要给 `socket` 提供通知 `ThreadPool` 的方法 —— 让 `socket` 拿着 `Arc<ThreadPool>`。完美。

为了让 `socket` 能够唤醒 `Executor`，标准库提供了 `Wake`。`socket` 只需握着 `Wake`，就能在准备好的时候调用 `wake()`，通知 `Executor` 该 `poll()`了。 下面是 `Wake` 的定义：

```rust
pub trait Wake: Send + Sync {
    fn wake(&self);
}
```

但是实现 `Wake` 的可以是 `Executor` 自己吗？不可以。因为一个 `Executor` 上可能托管着成百上千的 `TaskObj`，直接 `wake()` 并不能告诉 `Executor` 究竟是哪个 `TaskObj` 需要 `poll()`。因此 `Executor` 会给每个 `TaskObj` 提供一个唯一标识，然后把这个标识和自己的引用计数指针打包装在一起，弄一个类似 `WakeHandler` 的玩意。

```rust
struct WakeHandler {
    exec: Arc<ThreadPool>,
    id: u64,
}

impl Wake for WakeHandler { .. }
```

所以这里 `Async` 在被第一次 `poll()` 的时候 `socket` 就会记下这个 `WakeHandler`（因为这里 `Async` 的实际类型是 `socket` 提供的），准备好后  `socket`  调用 `wake()`，`Excutor` 根据唯一标识找到对应的 `Async` 然后 `poll()`, `poll()` 就会从 `socket` 拿回完成的 `Poll::Ready(T)`。完美。

## `Context`

明白了 `Excutor` 和 `Wake` 之后，让我们回过头看看 `Async` 的函数签名：

```rust
fn poll(self: Pin<Self>, cx: &mut task::Context) -> Poll<Self::Output>;
```

我们看 `poll()` 的第二个参数: `task::Context`:

```rust
pub struct Context<'a> { .. }

impl<'a> Context<'a> {
    pub fn new(waker: &'a Waker, executor: &'a mut Executor) -> Context<'a>;
    /// Get the `Waker` associated with the current task.
    pub fn waker(&self) -> &Waker;
    /// Run an asynchronous computation to completion on the default executor.
    pub fn spawn(&mut self, f: impl Async<Output = ()> + 'static + Send);
    /// Get the default executor associated with this task.
    pub fn executor(&mut self) -> &mut BoxExecutor;
}
```

`Context` 的主要工作十分明了：
-   提供 `Wake` 
-   提供默认 `Excutor`。

`spawn()` 是 `executor().spawn()` 的捷径（顺带一些错误处理）。

这里出现了 `Waker` 不是 `Wake`, `Waker` 又是一个这次标准库新加入的类型，里面装着 `Wake` trait object:

```rust
pub struct Waker {
    wake: &Wake
}

impl Waker {
    pub fn wake(&self) {
        self.wake.wake();
    }
}
```

## `Pin`

`Pin` 类型不在本篇 RFC 范围之内，而且展开了说篇幅会非常的长，有兴趣的话可以前去 RFC 阅读，或者看看 `@withoutboat` 关于 *`borrow across yield point`* 的长篇系列，那里详细地解释了设计上遇到了什么难点，以及为什么一定要引入 `Pin` 概念。

[RFC: 2349-pin](https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md)

[Async/Await I: Self-Referential Structs](https://boats.gitlab.io/blog/post/2018-01-25-async-i-self-referential-structs/)

目前在这篇 RFC 中，我们只需要知道 Pin 是什么就足够了。简单来说，一般的类型自动实现 `Unpin`，而有些特殊的的类型会反向实现 `!Unpin`，这样特殊的类型在被装进 `Pin<T>` 之后就不能再移动了(immovable type)。异步函数返回类型也正是这种特殊类型。

`Pin<T>` 是指针类型，提供比 `&mut T` 更严格的规则：

- `Pin<T: Unpin>` 将于普通 `&mut T` 完全一样。
- `Pin<T: !Unpin>` 只能提供 `Deref` 成 `&T`，以防止 T 被移动 (mem::replace())。

我们来明确一下:

- `Future`: `Unpin`
- async fn 返回的 `impl Async`: `!Unpin`
- `Box<Async>`: `Unpin`

## Conclusion

这篇文章介绍了标准库中加入的 `Async` 和任务类型，通过这些类型的加入，我们可以方便地定义自己的事件循环或者使用现成的库来驱动异步任务。文章并没有涵盖用于 `no_std` 环境的类型和解决方案，有兴趣的读者需要自己去阅读一下 RFC。

下一篇文章可能会讲 `Pin` 类型和关于 Rust `immovable type` 的故事，但是这玩意讲起来比裹脚布还长，到时懒起来分分钟就跳票了。