# 实例分析

从还是从spdk_io_device_register开始, 分析其io_device对应的io_channel做的额外操作.

标题2: 文件.

标题3: 函数.

正文: 此函数内必定有个spdk_io_device_register调用, 然后展开分析.

有些结构体, 这里不会列出其内容, 自行查看代码.

## lib/bdev/bdev.c

### spdk_bdev_initialize

```c
spdk_io_device_register(&g_bdev_mgr, bdev_mgmt_channel_create, bdev_mgmt_channel_destroy, sizeof(struct spdk_bdev_mgmt_channel), "bdev_mgr");
```

---

io_device为g_bdev_mgr, 类型为struct spdk_bdev_mgr.

```c
struct spdk_bdev_mgr {
    spdk_mempool *bdev_io_pool;
};
```

主要是管理一些mempool的分配.

---

额外空间为struct spdk_bdev_mgmt_channel.

```c
struct spdk_bdev_mgmt_channel {
    bdev_io_stailq_t per_thread_cache;
    TAILQ_HEAD(, spdk_bdev_shared_resource) shared_resources;
    TAILQ_HEAD(, spdk_bdev_io_wait_entry) io_wait_queue;
};
```

---

函数bdev_mgmt_channel_create.

```c
int bdev_mgmt_channel_create(void *io_device, void *ctx_buf)
{
    struct spdk_bdev_mgmt_channel *ch = ctx_buf;
    struct spdk_bdev_io *bdev_io;

    STAILQ_INIT(&ch->per_thread_cache);
    for (i = 0; i < ch->bdev_io_cache_size; i++) {
        bdev_io = spdk_mempool_get(g_bdev_mgr.bdev_io_pool);
        STAILQ_INSERT_HEAD(&ch->per_thread_cache, bdev_io, internal.buf_link);
    }
    TAILQ_INIT(&ch->shared_resources);
    TAILQ_INIT(&ch->io_wait_queue);
}
```

把g_bdev_mgr的bdev_io_pool按照配置分配到spdk_bdev_mgmt_channel的per_thread_cache里.

### bdev_init

```c
spdk_io_device_register(__bdev_to_io_dev(bdev), bdev_channel_create, bdev_channel_destroy, sizeof(struct spdk_bdev_channel), bdev_name);
```

---

io\_device为\_\_bdev_to_io_dev(bdev).

可以单纯把其看作是bdev, \_\_bdev_to_io_dev可以暂时无视. 类型为spdk_bdev.

```c
struct spdk_bdev {
    void *ctxt;
};
```

---

额外空间为struct spdk_bdev_channel.

```c
struct spdk_bdev_channel {
    struct spdk_bdev *bdev;
    struct spdk_io_channel *channel;
};
```

其中的channel是子channel.

---

函数bdev_channel_create.

```c
int bdev_channel_create(void *io_device, void *ctx_buf)
{
    struct spdk_bdev        *bdev = __bdev_from_io_dev(io_device);
    struct spdk_bdev_channel    *ch = ctx_buf;
    struct spdk_io_channel›     *mgmt_io_ch;
    struct spdk_bdev_mgmt_channel›  *mgmt_ch;

    ch->bdev = bdev;
    ch->channel = bdev->fn_table->get_io_channel(bdev->ctxt);
    mgmt_io_ch = spdk_get_io_channel(&g_bdev_mgr);
    mgmt_ch = spdk_io_channel_get_ctx(mgmt_io_ch);
}
```

绑定了spdk_bdev_channel->bdev为spdk_io_device_register相关的bdev.  
获取了子channel.  
获取了g_bdev_mgr->get_io_channel, 得到struct spdk_bdev_mgmt_channel.

## module/bdev/nvme/bdev_nvme.c

### bdev_nvme_library_init

```c
spdk_io_device_register(&g_nvme_bdev_ctrlrs, bdev_nvme_poll_group_create_cb, bdev_nvme_poll_group_destroy_cb, sizeof(struct nvme_bdev_poll_group), "bdev_nvme_poll_groups");
```

---

io_device为g_nvme_bdev_ctrlrs, 类型为struct nvme_bdev_ctrlrs(其实是一个struct nvme_bdev_ctrlr的链表).

```c
struct nvme_bdev_ctrlrs {
};
```

这里不关心其结构, 因为之后用不到.

---

额外空间为struct nvme_bdev_poll_group

```c
struct nvme_bdev_poll_group {
    struct spdk_nvme_poll_group *group;
    struct spdk_poller *poller;
};
```

---

函数bdev_nvme_poll_group_create_cb.

```c
int bdev_nvme_poll_group_create_cb(void *io_device, void *ctx_buf)
{
    struct nvme_bdev_poll_group *group = ctx_buf;

    group->group = spdk_nvme_poll_group_create(group);
    group->poller = spdk_poller_register(bdev_nvme_poll, group, g_opts.nvme_ioq_poll_period_us);
}
```

创建了group(类型为struct spdk_nvme_poll_group)  
创建了poller, 函数为bdev_nvme_poll.

### create_ctrlr

```c
spdk_io_device_register(nvme_bdev_ctrlr, bdev_nvme_create_cb, bdev_nvme_destroy_cb, sizeof(struct nvme_io_channel), name);
```

---

io_device为nvme_bdev_ctrlr, 类型为struct nvme_bdev_ctrlr.

```c
struct nvme_bdev_ctrlr {
    struct spdk_nvme_ctrlr›     *ctrlr;
    struct spdk_nvme_transport_id›  trid;
};
```

可以看到这里的结构体比较接近硬件信息了.

---

额外空间为struct nvme_io_channel.

```c
struct nvme_io_channel {
    struct spdk_nvme_qpair›     *qpair;
    struct nvme_bdev_poll_group›*group;
};
```

---

函数bdev_nvme_create_cb.

```c
int bdev_nvme_create_cb(void *io_device, void *ctx_buf)
{
    struct nvme_bdev_ctrlr *nvme_bdev_ctrlr = io_device;
    struct nvme_io_channel *ch = ctx_buf;
    struct spdk_io_channel *pg_ch = NULL;

    ch->qpair = spdk_nvme_ctrlr_alloc_io_qpair(nvme_bdev_ctrlr->ctrlr, &opts, sizeof(opts));
    pg_ch = spdk_get_io_channel(&g_nvme_bdev_ctrlrs);
    ch->group = spdk_io_channel_get_ctx(pg_ch);
    spdk_nvme_poll_group_add(ch->group->group, ch->qpair);
    spdk_nvme_ctrlr_connect_io_qpair(nvme_bdev_ctrlr->ctrlr, ch->qpair);
}
```

分配了io_qpair.  
获取了g_nvme_bdev_ctrlrs->get_io_channel, 得到了struct nvme_bdev_poll_group.

## module/bdev/zone_block/vbdev_zone_block.c

### zone_block_register

```c
spdk_io_device_register(bdev_node, _zone_block_ch_create_cb, _zone_block_ch_destroy_cb, sizeof(struct zone_block_io_channel), name->vbdev_name);
```

---

io_device为bdev_node, 类型为struct bdev_zone_block.

```c
struct bdev_zone_block {
    struct spdk_bdev bdev;
    struct spdk_bdev_desc *base_desc;
};
```

base_desc即为zone模块所依赖的nvme模块对应的bdev的描述符.

---

额外空间为struct zone_block_io_channel.

```c
struct zone_block_io_channel {
    struct spdk_io_channel *base_ch;
};
```

类似的, base_ch也是zone模块所依赖的nvme模块对应的设备的io_channel. 即子channel.

---

函数\_zone_block_ch_create_cb.

```c
int _zone_block_ch_create_cb(void *io_device, void *ctx_buf)
{
    struct zone_block_io_channel *bdev_ch = ctx_buf;
    struct bdev_zone_block *bdev_node = io_device;
    bdev_ch->base_ch = spdk_bdev_get_io_channel(bdev_node->base_desc);
}
```

获取了子channel.
