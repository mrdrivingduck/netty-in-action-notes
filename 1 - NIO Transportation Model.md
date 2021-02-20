# 1 - NIO Transportation Model

Created by : Mr Dk.

2021 / 02 / 13 0:54

Ningbo, Zhejiang, China

---

## 阻塞 I/O (BIO / OIO)

早期的 Java API (`java.net`) 只支持本地 OS 的 socket 库提供的阻塞函数。由于函数的阻塞，如果想要同时管理多个并发连接的客户端，需要为每个客户端建立一个线程来服务。常见的编程范式：

1. 实例化一个 `ServerSocket` 并绑定端口
2. 在死循环中调用 `accept()` 阻塞
3. 为新连接的 `Socket` 对象创建线程服务

```java
ServerSocket serverSocket = new ServerSocket(8080);
try {
    while (true) {
        Socket clientSocket = serverSocket.accept();
        new Thread(() -> {
            OutputStream out = null;
            try {
                out = clientSocket.getOutputStream();
                out.write("hello".getBytes());
                out.flush();
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    if (out != null) {
                        out.close();
                    }
                    if (clientSocket != null) {
                        clientSocket.close();
                    }
                } catch (IOException e) {

                }
            }
        }).start();
    }
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (serverSocket != null) {
        try {
            serverSocket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

传统阻塞 I/O 无法支撑大量的并发连接，原因如下：

* 在任何时候都会有大量线程处于休眠状态
* 每个线程的调用栈都被分配了内存
* 上下文切换的开销很大

## 非阻塞 I/O (NIO)

如今的 OS 本地 socket 库提供了非阻塞的调用：

* 可以对 socket 进行配置，使读写调用在没有数据时 **立刻返回**
* OS 提供事件通知 API (**I/O 多路复用**) 注册一组非阻塞的 socket，确定它们中是否包含任何已有数据可供读写的 socket

Java 在 2002 年的 JDK 1.4 中引入了 `java.nio`，提供了对非阻塞 I/O 的支持。其中，`java.nio.channels.Selector` 是非阻塞 I/O 的关键。它使用了 I/O 多路复用 API 向 OS 随时查询多个读写操作的完成状态。因此，单一的线程就可以处理多个并发的连接。多路复用器背后充当的功能是一个 **注册表**：对注册表的一次请求，能够获知已注册的所有 channel **是否发生了状态变化**。可能的状态有：

* `OP_ACCEPT` - 新的 channel 已被接受并就绪
* `OP_CONNECT` - Channel 连接已被建立
* `OP_READ` - Channel 中已有就绪可被读取的数据
* `OP_WRITE` - Channel 已就绪被用于写数据

与阻塞 I/O 相比，非阻塞 I/O 模型提供了更好的资源管理：

* 使用较少的线程就可以处理许多连接，减少了内存和上下文切换的开销
* 没有 I/O 操作需要处理时，线程也可被用于其它任务

编程范式主要包含几个部分：

1. 绑定 `ServerSocket` 到端口，并设置为非阻塞模式，注册 `ACCEPT` 事件
2. 在死循环中调用 `select()` 轮询注册后的所有事件
3. 如果出现 `ACCEPT` 事件，那么立刻调用 `accept()`，并将获取到的 socket 也设置为非阻塞，并注册该 socket 的 `READ` 和 `WRITE` 事件到 selector 上
4. 如果出现 `READ` 或 `WRITE` 事件，那么获取到相应的 buffer 并读取或写入

```java
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
ServerSocket serverSocket = serverChannel.socket();
serverSocket.bind(new InetSocketAddress(8080));
Selector selector = Selector.open();
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

ByteBuffer msg = ByteBuffer.wrap("Hello".getBytes());

while (true) {
    try {
        selector.select(); // blocked
    } catch (IOException e) {
        e.printStackTrace();
        break;
    }

    Set<SelectionKey> readyKeys = selector.selectedKeys();
    Iterator<SelectionKey> iter = readyKeys.iterator();
    while (iter.hasNext()) {
        SelectionKey key = iter.next();
        iter.remove();

        try {
            if (key.isAcceptable()) {
                ServerSocketChannel server = (ServerSocketChannel) key.channel();
                SocketChannel client = server.accept();
                client.configureBlocking(false);
                client.register(selector, SelectionKey.OP_WRITE | SelectionKey.OP_READ, msg.duplicate());
            }
            if (key.isWritable()) {
                SocketChannel client = (SocketChannel) key.channel();
                ByteBuffer buffer = (ByteBuffer) key.attachment();
                while (buffer.hasRemaining()) {
                    if (client.write(buffer) == 0) {
                        break;
                    }
                }
                client.close();
            }
        } catch (IOException e) {
            key.cancel();
            try {
                key.channel().close();
            } catch (IOException ee) {
                ee.printStackTrace();
            }
        }
    }
}
```

## Netty 编程模型

1. 初始化一个 `EventLoopGroup`
2. 将服务器引导到 `EventLoopGroup` 上，绑定本地地址，绑定处理函数

```java
ByteBuf buf = Unpooled.copiedBuffer("Hello", Charset.forName("utf-8"));

EventLoopGroup group = new NioEventLoopGroup(); // --> OioEventLoopGroup
try {
    ServerBootstrap b = new ServerBootstrap();
    b
        .group(group)
        .channel(NioServerSocketChannel.class) // --> OioServerSocketChannel.class
        .localAddress(new InetSocketAddress(8080))
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                    @Override
                    public void channelActive(ChannelHandlerContext ctx) throws Exception {
                        ctx
                            .writeAndFlush(buf.duplicate())
                            .addListener(ChannelFutureListener.CLOSE);
                    }
                });
            }
        });
    ChannelFuture future = b.bind().sync();
    future.channel().closeFuture().sync();
} finally {
    group.shutdownGracefully().sync();
}
```

Netty 同时支持阻塞版本与非阻塞版本的函数，并且两种版本的代码几乎完全相同，因为 Netty 为每种传输实现都暴露了相同的 API，业务代码几乎不受影响，在上面代码的注释位置稍作修改即可。Netty 内置的开箱即用传输包含：

| 名称     | 包                            | 描述                                              |
| -------- | ----------------------------- | ------------------------------------------------- |
| NIO      | `io.netty.channel.socket.nio` | 由 JDK 中的 NIO API 实现，保证了平台通用性        |
| OIO      | `io.netty.channel.socket.oio` | 由 JDK 中的 `java.net` 阻塞 API 实现              |
| EPOLL    | `io.netty.channel.epoll`      | 由 Linux JDK NIO API 实现，只能在 Linux 上使用    |
| Local    | `io.netty.channel.local`      | 同一个 JVM 内部运行的客户端与服务器之间的异步通信 |
| Embedded | `io.netty.channel.embedded`   | 不基于真正网络的传输，用于 handler 的单元测试     |

---

