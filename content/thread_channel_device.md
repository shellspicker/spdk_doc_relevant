# spdk设备模型

包含3个组件: thread, io_channel, io_device.

一句话解释就是: thread通过io_channel去操作io_device.

此篇文章的主角不是thread, 而是io_channel和io_device.

其中thread在spdk线程模型里面也会见到, 可以认为thread和线程模型关系大一点, 这里暂时不管线程模型也可以.

io_device, 即设备的抽象.

io_channel, 即设备行为的抽象.

先来看这3个组件的联系, 上图:

![看图](image/thread_channel_device.png)

这个图恰好是3层, 以thread--io_channel--io_device的层次表示.

细心一看, 1个thread到1个io_device的路线, 有且仅有1个io_channel.
所以这里一共2个thread, 2个io_device, 从上到下一共有4条路线.

考虑一下这种结构的意义, 可以使得不同的线程访问相同的设备, 行为上是独立的, 以便达到并行的效果.

来看一下3个组件的细节.

## thread

### struct spdk_thread

```c
struct spdk_thread {
    TAILQ_HEAD(, spdk_io_channel) io_channels;
};
```

thread到io_channel为1对多的关系.

## io_channel

### struct spdk_io_channel

```c
struct spdk_io_channel {
   struct spdk_thread *thread;
    struct io_device *dev;
    uint32_t ref;
};
```

io_channel到thread为1对1的关系.

io_channel到io_device为1对1的关系.

前面提到了, io_channel是抽象的行为, 但是此结构体内并没有void \*, 可以直接到源码里看, 这个结构体后面有一段注释, 这个结构体创建出来会分配一个**额外的空间**, 引用到具体的硬件相关或者子设备的spdk_io_channel上.

> Modules will allocate extra memory off the end of this structure to store references to hardware-specific references (i.e. NVMe queue pairs, or references to child device spdk_io_channels (i.e. virtual bdevs).

记住这个额外的空间, 后面会多次提到.

## io_device

### struct io_device

```c
struct io_device {
    void *io_device;
    spdk_io_channel_create_cb create_cb;
    uint32_t ctx_size;
};
```

由于io_device是个抽象的概念, 所以, 真实的设备其实是struct io_device里面的void \*io_device.

注意这里的ctx_size, 即spdk_io_channel里面的额外空间大小. 而create_cb, 则是和这个额外空间相关的一个函数.

现在可以暂时理解为, io_device会使用spdk_io_channel的额外空间来做一些额外的事情.

## helper function

### spdk_io_device_register

注册一个设备.

```c
void spdk_io_device_register(
    void *io_device,
    spdk_io_channel_create_cb create_cb,
    spdk_io_channel_destroy_cb destroy_cb,
    uint32_t ctx_size,
    const char *name)
{
    struct io_device *dev;
    dev = calloc(1, sizeof(struct io_device));
    dev->io_device = io_device;
    dev->create_cb = create_cb;
    dev->ctx_size = ctx_size;
    /*
    ......
    最后会把dev放到一个全局设备链表里.
    */
}
```

参数1, 真实的io_device.

参数2, spdk_io_channel对应的额外空间的**使用方法**.

参数4, spdk_io_channel中的额外空间**大小**.

主要是初始化一些成员.

### spdk_io_channel_create_cb

spdk_io_channel对应的额外空间的使用方法.

```c
typedef int(* spdk_io_channel_create_cb) (void *io_device, void *ctx_buf);
```

参数1, 真实的io_device.

参数2, spdk_io_channel中的额外空间的**地址**.

注意这里ctx_buf, 表示一个地址, 就是说此时已经把额外空间分配出来了, 这里相当于是把额外空间和具体的device关联起来.

至于什么时候创建的额外空间, 先不用关心.

这是一个函数指针, 这里以函数称呼之.

这个函数具体做了什么事情, 值得看一下, 因为这里算是首次使用这个额外空间.

不过此篇文章并不会讲实例, 所以想知道这个函数做了什么事情, 在具体分析实例的文章里会讲到.

### spdk_get_io_channel

创建/获取spdk_io_channel的方法.

```c
struct spdk_io_channel *(void *io_device)
{
    struct spdk_io_channel *ch;
    struct spdk_thread *thread;
    struct io_device *dev;
    /*
    ......
    在此之前肯定已经对void *io_device注册过了, 这里找到了其对应的struct io_device, 存为dev.
    thread直接从当前线程获取.
    */
    TAILQ_FOREACH(ch, &thread->io_channels, tailq) {
        if (ch->dev == dev) {
            ch->ref++;
            return ch;
        }
    }
    ch = calloc(1, sizeof(*ch) + dev->ctx_size);
    ch->dev = dev;
    ch->thread = thread;
    dev->create_cb(io_device, (uint8_t *)ch + sizeof(*ch));
    return ch;
}
```

注意开头3个结构体, 此函数把3个组件都关联在一起了, 可以说是核心函数了.

thread->io_channels的遍历, 我们可以看出来这个函数能推导出文章最初的层次图的含义:

1个thread对应1个io_device, 用的只有1个io_channel. 同一个thread多次访问io_device, 只会让同一个io_channel的引用计数增加而已.

struct spdk_io_channel分配出来即包含了额外空间的大小, 之后马上调用了[create_cb](#spdk_io_channel_create_cb)

这个过程, 有点类似设计模式里面的单例模式, 获取实例和初始化实例的过程都包含了.

### spdk_io_channel_get_ctx

获得spdk_io_channel之后额外空间的地址.

```c
void *spdk_io_channel_get_ctx(struct spdk_io_channel *ch)
{
    return (uint8_t *)ch + sizeof(*ch);
}
```

## 总结

主角2位: io_device和io_channel.

io_device对应结构体struct io_device.

io_channel对应结构体struct spdk_io_channel.

---

主要流程:

- 注册设备, 指定额外空间大小和想用额外空间来做事的函数.
- 首次得到spdk_io_channel的时候, 创建额外空间, 且执行额外空间对应的处理函数(类似单例模式的初始化).

---

如果忘了额外空间的意义, 回到[io_channel](#io_channel).

spdk_io_channel的额外空间可以指向另外的spdk_io_channel, 有点递归的意思, 当一个spdk_io_channel的额外空间指向了具体的硬件, 那么说明不会再有下一层spdk_io_channel了.

因为这里没有实例分析, 所以自行想象一下这个spdk_io_channel层层嵌套的情景.

## 疑问

为何同样是抽象, struct io_device是用void \*实现, struct spdk_io_channel是用额外空间实现?

struct spdk_io_channel能否把额外空间声明成一个void \*?
