# 3 - Netty Thread Model

Created by : Mr Dk.

2021 / 02 / 13 20:31

Longyou, Zhejiang, China

---

**线程模型** 指定了操作系统、编程语言、框架或者应用程序中的上下文管理的关键方面。如何及合适创建线程、调度线程将会对代码的执行产生显著的影响。由于具有多个 CPU 核心的计算机已经司空见惯，大多数现代应用程序都使用了复杂的多线程处理技术来有效利用系统资源。

在早期的 Java 中，多线程处理的具体方式是 **按需创建和启动新的 Thread** 来执行并发的任务单元，这种方式在高负载下工作得很差。Java 5 后引入了 `Executor` API，其中的线程池通过 **缓存和重用 `Thread`** 极大地提高了性能：

- 从线程池的空闲线程链表中选择一个 `Thread`，并指派其运行一个 `Runnable` 的任务
- 任务完成后，将 `Thread` 返回给列表

线程的池化相比于简单地创建和销毁线程来说是一种进步，但并不能消除上下文切换所带来的开销。上下文切换的开销随着线程数量的增加而加速增大。

## Interface Definition

事件循环是 Netty 的核心设计。从类结构上来说，`io.netty.channel.EventLoop` 派生自 `io.netty.concurrent` 包中的 `EventExecutor`，而 `EventExecutor` 又派生自 `java.util.concurrent` 中的 `ExecutorService`。也就是说，EventLoop 是基于 JUC 中的执行器框架扩展定义的。接口和类的继承关系图如下所示：

<img src="./img/7-2.png" alt="7-2" style="zoom:33%;" />

在该模型下，一个 EventLoop 将由一个永远都不会改变的 `Thread` 驱动。任务 (`Runnable` 或 `Callable`) 可以直接提交给 EventLoop 以立即执行或调度执行。根据机器配置的不同，可以创建多个 EventLoop 实例以充分利用资源。所有的 I/O 操作都由已经分配给 EventLoop 的那个 `Thread` 处理 - 通过在同一个线程中处理某个 EventLoop 的所有事件，可以避免一些本可能需要的同步操作。

### EventExecutorGroup

该接口的继承关系如下，负责维护多个 `EventExecutor` 的生命周期与它们的关闭。

```java
/**
 * The {@link EventExecutorGroup} is responsible for providing the {@link EventExecutor}'s to use
 * via its {@link #next()} method. Besides this, it is also responsible for handling their
 * life-cycle and allows shutting them down in a global fashion.
 *
 */
public interface EventExecutorGroup extends ScheduledExecutorService, Iterable<EventExecutor> {

}
```

接口内新定义的函数主要包含 **优雅地** 关闭执行器。当 `shutdownGracefully()` 被调用后，`isShuttingDown()` 将开始返回 `true`，表示执行器准备开始关闭自己。与 `ExecutorService` 提供地 `shutdown()` 不同，`shutdownGracefully()` 指定了一段 `quietPeriod` 时间：只有在这段时间以内没有任何任务被提交，执行器才开始关闭自己；如果这段时间内有新任务被提交，那么 `quietPeriod` 又将重新开始计算。

```java
/**
 * Returns {@code true} if and only if all {@link EventExecutor}s managed by this {@link EventExecutorGroup}
 * are being {@linkplain #shutdownGracefully() shut down gracefully} or was {@linkplain #isShutdown() shut down}.
 */
boolean isShuttingDown();

/**
 * Shortcut method for {@link #shutdownGracefully(long, long, TimeUnit)} with sensible default values.
 *
 * @return the {@link #terminationFuture()}
 */
Future<?> shutdownGracefully();

/**
 * Signals this executor that the caller wants the executor to be shut down.  Once this method is called,
 * {@link #isShuttingDown()} starts to return {@code true}, and the executor prepares to shut itself down.
 * Unlike {@link #shutdown()}, graceful shutdown ensures that no tasks are submitted for <i>'the quiet period'</i>
 * (usually a couple seconds) before it shuts itself down.  If a task is submitted during the quiet period,
 * it is guaranteed to be accepted and the quiet period will start over.
 *
 * @param quietPeriod the quiet period as described in the documentation
 * @param timeout     the maximum amount of time to wait until the executor is {@linkplain #shutdown()}
 *                    regardless if a task was submitted during the quiet period
 * @param unit        the unit of {@code quietPeriod} and {@code timeout}
 *
 * @return the {@link #terminationFuture()}
 */
Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);
```

接口还额外提供了在所有的 `EventExecutor` 都终结后的回调。

```java
/**
 * Returns the {@link Future} which is notified when all {@link EventExecutor}s managed by this
 * {@link EventExecutorGroup} have been terminated.
 */
Future<?> terminationFuture();
```

接口定义了函数，返回 `EventExecutorGroup` 内管理的 `EventExecutor` 中的一个。

```java
/**
 * Returns one of the {@link EventExecutor}s managed by this {@link EventExecutorGroup}.
 */
EventExecutor next();
```

### EventLoopGroup

该类继承自 `EventExecutorGroup` 接口：

```java
/**
 * Special {@link EventExecutorGroup} which allows registering {@link Channel}s that get
 * processed for later selection during the event loop.
 *
 */
public interface EventLoopGroup extends EventExecutorGroup {

}
```

该接口内额外定义了注册 `Channel` 的功能。多个 `Channel` 可以注册到 `EventLoopGroup` 上，并被其中的一个 `EventLoop` 选择：

```java
/**
 * Register a {@link Channel} with this {@link EventLoop}. The returned {@link ChannelFuture}
 * will get notified once the registration was complete.
 */
ChannelFuture register(Channel channel);

/**
 * Register a {@link Channel} with this {@link EventLoop} using a {@link ChannelFuture}. The passed
 * {@link ChannelFuture} will get notified once the registration was complete and also will get returned.
 */
ChannelFuture register(ChannelPromise promise);
```

### EventExecutor (EventLoop)

该接口是特殊的 `EventExecutorGroup` 实现，也是 `EventLoop` 的父接口。

```java
/**
 * The {@link EventExecutor} is a special {@link EventExecutorGroup} which comes
 * with some handy methods to see if a {@link Thread} is executed in a event loop.
 * Besides this, it also extends the {@link EventExecutorGroup} to allow for a generic
 * way to access methods.
 *
 */
public interface EventExecutor extends EventExecutorGroup {

}
```

接口定义了确定一个线程是否是 EventLoop 线程的函数。Netty 线程模型的卓越性能取决于对于当前正在执行的 `Thread` **身份** 的确定：

- 如果当前线程正是 EventLoop 线程，那么任务将被立刻执行
- 否则 EventLoop 将调度该任务稍后执行，保存在内部队列中

每个 EventLoop 都有自己的任务队列，独立于任何其它 EventLoop。因此，同一个 EventLoop 内部无需额外同步。另外，不能将一个执行时间较长的任务放入执行队列，否则将会阻塞同一个 EventLoop 线程将要执行的其它任务。

```java
/**
 * Calls {@link #inEventLoop(Thread)} with {@link Thread#currentThread()} as argument
 */
boolean inEventLoop();

/**
 * Return {@code true} if the given {@link Thread} is executed in the event loop,
 * {@code false} otherwise.
 */
boolean inEventLoop(Thread thread);
```

`EventExecutor` 能够通过 `parent()` 获得 `EventExecutorGroup` 的引用：

```java
/**
 * Return the {@link EventExecutorGroup} which is the parent of this {@link EventExecutor},
 */
EventExecutorGroup parent();
```

## EventLoop Thread Allocation

### Asynchronous Transportation

异步传输的实现只用了少量的 EventLoop 线程，每个 EventLoop 被多个 Channel 共享。通过尽可能少的 EventLoop 线程支撑大量的 Channel 能够减少内存开销与上下文切换开销。所有的 EventLoop 都由 EventLoopGroup 分配，每个 EventLoop 与一个 `Thread` 相关联。EventLoopGroup 会为每一个新创建的 Channel 分配一个 EventLoop 来处理，当前的默认实现是 _round-robin_。对于一个 Channel 来说，整个生命周期内的所有操作都由一个 EventLoop Thread 执行。

### Blocking Transportation

由于阻塞传输的特性，每个 Channel 都将被分配给一个 EventLoop (即一个 `Thread`)，得到的保证是每个 Channel 的 I/O 事件只会被一个 `Thread` 处理，这个 `Thread` 也将只处理这一个 Channel 的事件。这种设计在满足阻塞场景的同时，也保证了 Netty 编程模式的一致性。
