# 2 - ByteBuf

Created by : Mr Dk.

2021 / 02 / 13 20:31

Ningbo, Zhejiang, China

---

网络数据的基本单位是字节。Java NIO 提供了 `ByteBuffer` 作为字节容器，但是使用起来较为繁琐。Netty 实现了 `ByteBuf`，为开发者提供了更好的 API。

## Access

根据 `ByteBuf` 源代码中的注释，可以获知其工作原理。`ByteBuf` 是随机顺序访问的字节序列。

`ByteBuf` 在底层支持顺序 / 随机访问。对于顺序访问，其内部维护的索引如注释中所示：

* 读索引之前的字节是被使用丢弃后的字节
* 读索引与写索引之间是已被写入可以被读取的字节，在调用 `read` / `skip` 操作后读索引向前推进
* 写索引与容量之间是可以被写入的字节空间，在调用 `write` 操作后写索引向前推进

```
 +-------------------+------------------+------------------+
 | discardable bytes |  readable bytes  |  writable bytes  |
 |                   |     (CONTENT)    |                  |
 +-------------------+------------------+------------------+
 |                   |                  |                  |
 0      <=      readerIndex   <=   writerIndex    <=    capacity
```

对于随机访问 (`get` / `set` 操作)，API 可以操作 buffer 中 `[0, capacity)` 范围内的任意字节，并且 **不会对读写索引产生影响**。

对于 buffer 最前面的可丢弃字节，可以进行重新回收利用，以最大化可写字节数。但是这个操作将会导致内存复制：

```
BEFORE discardReadBytes()

    +-------------------+------------------+------------------+
    | discardable bytes |  readable bytes  |  writable bytes  |
    +-------------------+------------------+------------------+
    |                   |                  |                  |
    0      <=      readerIndex   <=   writerIndex    <=    capacity

AFTER discardReadBytes()

    +------------------+--------------------------------------+
    |  readable bytes  |    writable bytes (got more space)   |
    +------------------+--------------------------------------+
    |                  |                                      |
readerIndex (0) <= writerIndex (decreased)        <=        capacity
```

通过将读写索引清零，可以清空 buffer：

```
BEFORE clear()

    +-------------------+------------------+------------------+
    | discardable bytes |  readable bytes  |  writable bytes  |
    +-------------------+------------------+------------------+
    |                   |                  |                  |
    0      <=      readerIndex   <=   writerIndex    <=    capacity

AFTER clear()

    +---------------------------------------------------------+
    |             writable bytes (got more space)             |
    +---------------------------------------------------------+
    |                                                         |
    0 = readerIndex = writerIndex            <=            capacity
```

通过一个已有的 buffer 获取视图的方式包含两种：

* 派生缓冲区 - 返回新的 `ByteBuf` 实例，里面有独立的读写索引，但是底层数组与原对象共享
* 复制缓冲区 - 返回现有 buffer 的真实副本 (通过 `copy()`)

## References Count

Netty 提供了显式提升或降低一个对象的被引用计数的能力。`io.netty.util.ReferenceCounted` 接口内维护了一个对象的引用计数，通常从 `1` 开始。当调用 `retain()` 函数时，对象的引用计数增加；当调用 `release()` 时，对象的引用计数减少。当对象的引用计数为 `0` 时，对象将被显式释放，不可再被访问。

`ByteBuf` 也实现了 `ReferenceCounted` 接口。

```java
/**
 * A reference-counted object that requires explicit deallocation.
 * <p>
 * When a new {@link ReferenceCounted} is instantiated, it starts with the reference count of {@code 1}.
 * {@link #retain()} increases the reference count, and {@link #release()} decreases the reference count.
 * If the reference count is decreased to {@code 0}, the object will be deallocated explicitly, and accessing
 * the deallocated object will usually result in an access violation.
 * </p>
 * <p>
 * If an object that implements {@link ReferenceCounted} is a container of other objects that implement
 * {@link ReferenceCounted}, the contained objects will also be released via {@link #release()} when the container's
 * reference count becomes 0.
 * </p>
 */
public interface ReferenceCounted {
    /**
     * Returns the reference count of this object.  If {@code 0}, it means this object has been deallocated.
     */
    int refCnt();

    /**
     * Increases the reference count by {@code 1}.
     */
    ReferenceCounted retain();

    /**
     * Increases the reference count by the specified {@code increment}.
     */
    ReferenceCounted retain(int increment);

    /**
     * Records the current access location of this object for debugging purposes.
     * If this object is determined to be leaked, the information recorded by this operation will be provided to you
     * via {@link ResourceLeakDetector}.  This method is a shortcut to {@link #touch(Object) touch(null)}.
     */
    ReferenceCounted touch();

    /**
     * Records the current access location of this object with an additional arbitrary information for debugging
     * purposes.  If this object is determined to be leaked, the information recorded by this operation will be
     * provided to you via {@link ResourceLeakDetector}.
     */
    ReferenceCounted touch(Object hint);

    /**
     * Decreases the reference count by {@code 1} and deallocates this object if the reference count reaches at
     * {@code 0}.
     *
     * @return {@code true} if and only if the reference count became {@code 0} and this object has been deallocated
     */
    boolean release();

    /**
     * Decreases the reference count by the specified {@code decrement} and deallocates this object if the reference
     * count reaches at {@code 0}.
     *
     * @return {@code true} if and only if the reference count became {@code 0} and this object has been deallocated
     */
    boolean release(int decrement);
}
```

## Pooling

`ByteBuf` 的创建支持 **池化** / **非池化**。Netty 通过 `io.netty.buffer.ByteBufAllocator` 接口实现 `ByteBuf` 的池化。`ByteBuf` 的类型包含以下几种：

* 堆上 buffer - 底层将数据维护在 JVM 堆空间中的一个字节数组里
* 直接缓冲区 - 驻留在会被 GC 的常规堆内存外，分配和释放代价昂贵
* 组合缓冲区 - 为多个 `ByteBuf` 提供一个聚合视图，是将多个缓冲区表示为单个合并缓冲区的虚拟表示，消除了没必要的复制

```java
/**
 * Implementations are responsible to allocate buffers. Implementations of this interface are expected to be
 * thread-safe.
 */
public interface ByteBufAllocator {

    ByteBufAllocator DEFAULT = ByteBufUtil.DEFAULT_ALLOCATOR;

    /**
     * Allocate a {@link ByteBuf}. If it is a direct or heap buffer
     * depends on the actual implementation.
     */
    ByteBuf buffer();
    ByteBuf buffer(int initialCapacity);
    ByteBuf buffer(int initialCapacity, int maxCapacity);

    /**
     * Allocate a {@link ByteBuf}, preferably a direct buffer which is suitable for I/O.
     */
    ByteBuf ioBuffer();
    ByteBuf ioBuffer(int initialCapacity);
    ByteBuf ioBuffer(int initialCapacity, int maxCapacity);

    /**
     * Allocate a heap {@link ByteBuf}.
     */
    ByteBuf heapBuffer();
    ByteBuf heapBuffer(int initialCapacity);
    ByteBuf heapBuffer(int initialCapacity, int maxCapacity);

    /**
     * Allocate a direct {@link ByteBuf}.
     */
    ByteBuf directBuffer();
    ByteBuf directBuffer(int initialCapacity);
    ByteBuf directBuffer(int initialCapacity, int maxCapacity);

    /**
     * Allocate a {@link CompositeByteBuf}.
     * If it is a direct or heap buffer depends on the actual implementation.
     */
    CompositeByteBuf compositeBuffer();
    CompositeByteBuf compositeBuffer(int maxNumComponents);
    CompositeByteBuf compositeHeapBuffer();
    CompositeByteBuf compositeHeapBuffer(int maxNumComponents);
    CompositeByteBuf compositeDirectBuffer();
    CompositeByteBuf compositeDirectBuffer(int maxNumComponents);

    /**
     * Returns {@code true} if direct {@link ByteBuf}'s are pooled
     */
    boolean isDirectBufferPooled();

    /**
     * Calculate the new capacity of a {@link ByteBuf} that is used when a {@link ByteBuf} needs to expand by the
     * {@code minNewCapacity} with {@code maxCapacity} as upper-bound.
     */
    int calculateNewCapacity(int minNewCapacity, int maxCapacity);
}
```

在具体的实现上，`ByteBufAllocator` 包含：

* `PooledByteBufAllocator` 使用 *jemalloc* 的方法来高效地分配内存，最大程度地减少内存碎片
* `UnpooledByteBufAllocator` 的实现不池化 `ByteBuf` 实例，每次调用都返回一个新的 `ByteBuf` 实例

在无法获得到一个 `ByteBufAllocator` 的引用时，通过 `io.netty.buffer.Unpooled` 工具类，内部使用一个 `UnpooledByteBufAllocator` 的实例，可以轻松分配非池化的不同类型的 `ByteBuf`：

```java
public final class Unpooled {

    private static final ByteBufAllocator ALLOC = UnpooledByteBufAllocator.DEFAULT;
    
    // ...
    
    /**
     * Creates a new big-endian Java heap buffer with reasonably small initial capacity, which
     * expands its capacity boundlessly on demand.
     */
    public static ByteBuf buffer() {
        return ALLOC.heapBuffer();
    }
    
    // ...
    
    /**
     * Creates a new big-endian direct buffer with reasonably small initial capacity, which
     * expands its capacity boundlessly on demand.
     */
    public static ByteBuf directBuffer() {
        return ALLOC.directBuffer();
    }
    
    // ...
    
    /**
     * Creates a new big-endian buffer which wraps the specified {@code array}.
     * A modification on the specified array's content will be visible to the
     * returned buffer.
     */
    public static ByteBuf wrappedBuffer(byte[] array) {
        if (array.length == 0) {
            return EMPTY_BUFFER;
        }
        return new UnpooledHeapByteBuf(ALLOC, array, array.length);
    }
    
    // ...
    
    /**
     * Creates a new big-endian buffer whose content is a copy of the
     * specified {@code array}.  The new buffer's {@code readerIndex} and
     * {@code writerIndex} are {@code 0} and {@code array.length} respectively.
     */
    public static ByteBuf copiedBuffer(byte[] array) {
        if (array.length == 0) {
            return EMPTY_BUFFER;
        }
        return wrappedBuffer(array.clone());
    }
    
    // ...
}
```

---

