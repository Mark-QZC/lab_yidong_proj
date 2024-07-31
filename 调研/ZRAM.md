# 20210728
### ZRAM内存压缩机制
[一文读懂｜zRAM 内存压缩机制_linux zram-CSDN博客](https://blog.csdn.net/youzhangjing_/article/details/133685959)
zRAM 的原理是：将进程不常用的内存压缩存储，从而达到节省内存的使用。
zRAM 机制建立在 swap 机制之上，swap 机制是将进程不常用的内存交换到磁盘中，而 zRAM 机制是将进程不常用的内存压缩存储在内存某个区域。所以 zRAM 机制并不会发生 I/O 操作，从而避免因 I/O 操作导致的性能下降。
--zRAM使用开辟的少量内存空间和一些cpu时间来换取比与磁盘IO更少的时间开销。
由于 zRAM 机制是建立在 swap 机制之上，而 swap 机制需要配置 `文件系统` 或 `块设备` 来完成的。所以 zRAM 虚拟一个块设备，当系统内存不足时，swap 机制将内存写入到这个虚拟的块设备中。也就是说，zRAM 机制本质上只是一个虚拟块设备。

（启用zRAM，创建zRAM块设备，设置zRAM块设备的大小，压缩算法选择（lzo，lz4），将swap交换设备设置成zRAM

#### zRAM实现
**zRAM 块设备驱动的实现代码主要在 **`drivers/block/zram/zram_drv.c`** 文件中**
下仅分析 swap 机制在进行内存交换时，与 zRAM 块设备驱动的交互。
当 swap 机制将不常用的内存交换到 zRAM 块设备时，会调用 `zram_make_request()` 函数处理请求。而 `zram_make_request()` 最终会通过调用 `zram_bvec_write()` 函数来压缩内存，调用链如下：
```cpp
zram_make_request()
 -> __zram_make_request()
     -> zram_bvec_rw()
         -> zram_bvec_write()
    ///////////////////////////////
            ->zram_write_page()
                ->zcomp_compress()		//zcomp.c
                    ->crypto_comp_compress()
```
```cpp
static int
zram_bvec_write(struct zram *zram, struct bio_vec *bvec, u32 index, int  )
{
    ...
    // 1. 获取需要进行压缩的内存
    page = bvec->bv_page;
    ...
    user_mem = kmap_atomic(page);
    uncmem = user_mem;
 
    ...
    // 2. 对内存进行压缩
    ret = zcomp_compress(zram->comp, zstrm, uncmem, &clen);
    ...
 
    // 3. 获取压缩后的数据
    src = zstrm->buffer;
    ...
 8
    // 4. 申请一个内存块保存压缩后的数据
    handle = zs_malloc(meta->mem_pool, clen);
    ...
    cmem = zs_map_object(meta->mem_pool, handle, ZS_MM_WO);
 
    // 5. 将压缩后的数据保存到新申请的内存块中
    memcpy(cmem, src, clen);
    ...
 
    // 6. 将压缩后的数据登记到 zRAM 块设备的表格中
    meta->table[index].handle = handle;
    ...
    return ret;
}
```
为了简化分析过程，我们对代码进行精简。从上面的代码可以看出，zRAM 机制对内存进行压缩的步骤如下：

- 获取需要进行压缩的内存，需要进行压缩的内存由 swap 机制提供。
- 通过 zcomp_compress() 函数对内存进行压缩，src 指针指向压缩后的内存地址。
- 通过 zs_malloc() 和 zs_map_object() 函数申请一块新的内存块，大小为压缩后数据的大小。
- 将压缩后的数据复制到新申请的内存块中。
- 将压缩后的数据记录到 zRAM 块设备的表格中。

**压缩函数：**`ret = zcomp_compress(zram->comp, zstrm, uncmem, &clen);`
函数所用参数：zram->comp
							zstrm
							uncmem
							&clen



由于 zRAM 块设备是建立在内存中的虚拟块设备，所以其并没有真实块设备的特性。真实块设备会将存储空间划分成一个个块，而 zram_bvec_write() 函数的 index 参数就是数据块的编号。此参数有 swap 机制提供，所以 zRAM 块设备驱动通过 index 参数作为原始内存数据的编号。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429053845-0968b99f-be07-4bc8-8b71-5289a7fed17a.png#averageHue=%23fcfdf9&clientId=u3fa46190-4897-4&from=paste&height=431&id=Tl3E4&originHeight=646&originWidth=1153&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=133037&status=done&style=none&taskId=u6cea3a5e-6664-4c41-b380-19064a35c76&title=&width=768.6666666666666)
zRAM驱动有个数据块表，用来记录原始内存数据对应的压缩数据，此表的索引就是数据块的编号。swap 机制会维护此表格的使用情况，如哪个块是空闲的，哪个块被占用等。
当内存页被压缩后，swap 机制将会把原来的内存页释放掉，并且把所有映射到此内存页的进程解除映射，细节可以参考 swap 机制相关的资料。


### 2 内存管理基本概念
#### 2.1 内存管理区struct zone
内存管理区有三个，dma zone，normal zone，high zone。ARM架构下，normal zone也可以用dma操作，故dma zone为0.
#### 2.2 FPN
对可用内存进行编号。


### zRAM内存压缩技术分析及优化方向
#### 1.zRAM出现的背景
阿姆达尔定律告诉我们，任何计算机操作系统总是存在瓶颈。从历史上看，许多系统上工作负载的瓶颈是 CPU，因此系统设计人员致力于 CPU 运行更快、更高效，不断增加 CPU 内核的数量。 慢慢地，RAM 越来越成为瓶颈。当数据从磁盘加载到 RAM时，CPU会闲置地等待。增大 RAM 并不总是一个具有成本效益的选择，有时甚至根本不是一个选择。更快的 I/O 总线和固态磁盘SSD减少了瓶颈，但并没有消除它。于是工程师们想，假如可以增加存储在 RAM 中的有效数据量，那不是很好吗？而且，由于这些 CPU 无论如何都在等待，也许我们可以使用这些空闲的 CPU 周期来做一些事情实现这一目标？这是内核压缩的目标：我们在 RAM 中保留更多压缩数据，并使用空闲的 CPU 周期来执行压缩和解压缩算法。
zRAM于2014 年进入 Linux 3.14 内核主线，但由于 Linux 用途十分广泛，这一技术并非没有默认启用，只有 Android 移动终端设备和少部分的 Linux 桌面发行版如 Fedora 默认启用了这一技术，以保证多任务场景下内存的合理分层存储。
举个具体的例子，你在android手机上连续打开了10个应用，系统空闲内存持续下降，通过zRAM内存压缩会把最近最少使用的页面压缩起来， 如淘宝的页面已最近最少使用的，原来占用100M的匿名页会压缩起来存放，压缩后占用40M，通过这个压缩就节省了60M内存。从而过到内存压缩保留更多数据在RAM中的目的, 让用户体验更佳。
#### 2.zRAM软件架构
zRAM本质是一个块设备驱动，它使用**内存模拟block device**的做法。它把内存回收的策略交给内存管理，把压缩和解压缩交给压缩库，把自身内存分配交给zsmalloc， zRAM自身就是一个简单的驱动。
zRAM的软件架构主要包含3部分:
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429080520-7616859a-c5d4-4bcc-bbf3-fc356ddc50da.png#averageHue=%23a1d369&clientId=u3fa46190-4897-4&from=paste&height=473&id=ub61ab735&originHeight=709&originWidth=759&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=106212&status=done&style=none&taskId=ub9535ff0-95bd-4eb4-a0b4-32dc11aa93e&title=&width=506)
#### 3. zRAM实现分析
##### 3.1 zRAM驱动模块

- zram_init

zram_init注册了zram块设备驱动,流程如下:
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429103520-b2b7e1b0-dc73-4b5c-ac4e-9051bee19c59.png#averageHue=%23d2e8ba&clientId=u3fa46190-4897-4&from=paste&height=335&id=u0ca879fe&originHeight=502&originWidth=1062&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=229170&status=done&style=none&taskId=ud6ed6119-f660-475a-9f9f-c1849d6a177&title=&width=708)

- disksize_store

创建zram设备驱动后,通过**用户态节点**配置zram块设备大小， 对应disksize_store函数。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429138734-dc939511-0a3b-4d10-bbfd-7fce988fcd7d.png#averageHue=%23f0f0e9&clientId=u3fa46190-4897-4&from=paste&height=283&id=ud24690b6&originHeight=424&originWidth=1069&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=198424&status=done&style=none&taskId=ua0360010-38f3-404b-bf0b-22cd99ab151&title=&width=712.6666666666666)

- zram_make_request

所有zram的块设备请求都是通过zram_make_request进行的。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429159217-f29d6e9e-eeec-48af-81c3-a8d0afa68000.png#averageHue=%23f8f8f7&clientId=u3fa46190-4897-4&from=paste&height=192&id=u5957a95d&originHeight=288&originWidth=1156&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=102828&status=done&style=none&taskId=ub26a75a3-fb7e-4311-9ef2-997f39254dc&title=&width=770.6666666666666)
##### 3.2数据流模块
创建压缩数据流程如下：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429172018-c8ec3414-3914-41c4-8a80-8d540475170d.png#averageHue=%23f7f7f7&clientId=u3fa46190-4897-4&from=paste&height=168&id=u73c909f9&originHeight=252&originWidth=1117&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=85748&status=done&style=none&taskId=u97bc2d3e-afcd-41e7-b158-c23ba0fbb2a&title=&width=744.6666666666666)
之所以要针对每个在线CPU都创建一个压缩数据流，主要是优化系统中并发的情况。每个写压缩操作都独享一个压缩数据流，通过多个压缩数据流，使得系统允许多个并发压缩操作。
##### 3.3压缩算法模块
zRAM默认使用lzo压缩算法，还有很多压缩算法可以使用。包括有lz4压缩算法。
##### 3.4zRAM读写流程
写（压缩）流程：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429188635-f283b947-b4cf-4c97-bd3b-0519f3d7f779.png#averageHue=%23c9e4aa&clientId=u3fa46190-4897-4&from=paste&height=585&id=u1a6b10da&originHeight=877&originWidth=1120&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=452163&status=done&style=none&taskId=u25fcd7a8-0e5d-49a2-8938-090c77a61a2&title=&width=746.6666666666666)
读（解压缩）流程：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429197892-68ff625b-9173-41f3-beea-2edc4f26c726.png#averageHue=%23c5e1a5&clientId=u3fa46190-4897-4&from=paste&height=666&id=u553fe2cd&originHeight=999&originWidth=1129&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=474140&status=done&style=none&taskId=u20008765-f21a-4b59-8761-0498adf05d9&title=&width=752.6666666666666)
##### 3.5zRAM writeback功能
zRAM是将内存压缩后存放起来，仍然是在在RAM中。如果有大量一次性访问页面被压缩后很长时间没有被再次被访问， 虽然经过压缩但仍然占内存。zRAM支持设置外部存储分区作为zRAM的backing_dev，对不可压缩内存（ZRAM_HUGE）和长时间没有被访问过的内存（ZRAM_IDLE）回写到外部存储中。
以Android手机为例， 如果配置开启了zram writeback有idle回写功能，流程如下：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429209942-e559c22d-70f0-415d-a369-4dce8dd84948.png#averageHue=%23f5f0eb&clientId=u3fa46190-4897-4&from=paste&height=422&id=u6d255fe2&originHeight=633&originWidth=1144&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=264133&status=done&style=none&taskId=u42348005-3ea9-4fe4-b08b-4b4164f24aa&title=&width=762.6666666666666)
#### 4.zRAM性能调优
##### 4.1zram大小
【调优节点】： /sys/block/zram0/disksize
内核提供了/sys/block/zram0/disksize节点供配置zram大小。可以实际使用场景实测，一般有场景、重载的场景zram等典型场景观察内存相关指标。根据需求配置合理的zram大小。有人可能问：" 反正这个是最大值，不是设置越大越好吗？”
我们在上面disksize_store分析的时候提到过，zram大小是需要为其分配zram table的，是需要占用一定内存空间的。如果配置很大的zram空间但平时有用不着，这个就浪费内存的。
##### 4.2内存压缩算法
**【调优节点】：/sys/block/zram0/comp_algorithm**
在上面讲解压缩算法的时候，我们提到过各种压缩算法，这些压缩算法在不同平台，不同场景都有差异性的表现。可以根据需求选择最适用的。评估的考量主要是压缩/解压缩消耗的CPU cycle和压缩率的平衡。
##### 4.3zram的簇预读
【调优节点】：/proc/sys/vm/page-cluster
page-cluster的作用就是每次从交换设备读取数据时多读 2^n 页，一些块设备或文件系统有簇的概念，读取数据也是按簇读取，假设内存页大小为 4KiB，page-cluster为3的情况每次读取的一簇数据为 32KiB
这是swap时利用局部性原理，把附近的几个页“顺便”也读出来，减少频繁读取磁盘的次数。而其实使用zRAM时，并不会有磁盘相关操作，这里多读一些甚至是浪费内存的。
而且压缩后在zram中的数据也并不一定具有局部性原理。这里是可以通过实际测试来判断的。
```c
void __init swap_setup(void)
{
    unsigned long megs = totalram_pages >> (20 - PAGE_SHIFT);

    /* Use a smaller cluster for small-memory machines */
    if (megs < 16)
        page_cluster = 2;
    else
        page_cluster = 3;
    /*
     * Right now other parts of the system means that we
     * _really_ don't want to cluster much more
     */
}

```
##### 4.4zram的vma预读
【调优节点】：/sys/kernel/mm/swap/vma_ra_enabled
预读始终是swap机制的优化方向之一。上面提到的基于page-cluster的预读可能不一定适合zram。内核工程师们提出了基于VMA地址的预读， 对于zRAM可能更符合局部性原理。
内核支持可配置基于vma的预读还是page-cluster的预读。
```c
struct page *swapin_readahead(swp_entry_t entry, gfp_t gfp_mask,
                struct vm_fault *vmf)
{
    return swap_use_vma_readahead() ?
            swap_vma_readahead(entry, gfp_mask, vmf) :
            swap_cluster_readahead(entry, gfp_mask, vmf);

```
##### 4.5zram使用程度倾向
【调优节点】：/proc/sys/vm/swappiness
swappiness参数用于表征更倾向于回收匿名页还是文件页。Linux5.8以下版本swapiness默认值是60，最大值是100， 如果swappiness设置为100表示匿名页和文件将用同样的优先级进行回收。由于zram设备非外部存储设备，其本质还是对RAM的访问，可以相对更激进地使用zram，即把swappiness设置得比100还大，也就是回收匿名页比文件页激进。
linux5.8支持把swappiness设置得比100还大， 最大值从100修改为了200， 提交[[4]](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F565651762%23ref_4)已经进入5.8版本主线。如下：
```
mm: allow swappiness that prefers reclaiming anon over the file workingset
```
文档同时也更新了说明， 对于zram设备建议swappiness大于100。
```
For in-memory swap, like zram or zswap, [...] values beyond 100 can be considered. 
For example, if the random IO against the swap device is on average 2x faster than 
IO from the filesystem, swappiness should be 133 (x + 2x = 200, 2x = 133.33).
```
##### 4.6zRAM去重
【调优节点】：/sys/block/zram0/ dedup_enable
Android 是使用 zram 作为swap device的最大用户之一，节省zram内存使用量非常重要。有研究论文《MemScope: Analyzing Memory Duplication on Android Systems》[[5]](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F565651762%23ref_5)表明Android的内存内容的重复率是相当高的。
Android 应用进程和系统进程可能表现出不同的特点，因为 Android 应用程序运行 dalvik VM 但非应用程序进程不运行。应用进程的内存内容重复率平均约14%，系统进程的内存内容重复率平均约3%。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429287648-1a8c1da2-8921-4faf-9f76-3e8e9ac51ced.png#averageHue=%23f2f2f2&clientId=u3fa46190-4897-4&from=paste&height=485&id=uc850e2e7&originHeight=727&originWidth=1110&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=410057&status=done&style=none&taskId=ue1d948d4-7896-48aa-a580-085ce60a5aa&title=&width=740)
论文调查了哪些内存区域包含最多重复页面，结果是超过 80% 的重复页面来自来自匿名页、gralloc-buffer、dalvik-heap、dalvik-zygote和 libdvm.so 库的 BSS 段。如下图：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429299444-cc39a137-decf-42ec-855d-72b899eddbbd.png#averageHue=%23efefef&clientId=u3fa46190-4897-4&from=paste&height=501&id=u624bff41&originHeight=751&originWidth=1140&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=367975&status=done&style=none&taskId=uab72733f-2299-488d-8523-f329ef979d0&title=&width=760)
于是内核工程师们提出了zram支持去重功能以节省功能，提交集[[6]](https://link.juejin.cn/?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F565651762%23ref_6)如下：
```
zram: implement deduplication in zram
```
可以通过/sys/block/zram/io_stat/中zram->dedup->dedups查看重复的page量。
##### 4.7多种压缩算法组合压缩
社区在讨论一个zram多重压缩的优化。可以在压缩后再次变更压缩算法对idle和huge的page再次压缩。基本思想是使用一种更快但压缩率较低的默认算法率和可以使用更高压缩率的辅助算法。以较慢的压缩/解压缩为代价。替代压缩算法可以提供更好的压缩比，以减少 zsmalloc 内存使用。当然这个改动[[7]](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F565651762%23ref_7)还没有合入到主线中。
```
zram: Support multiple compression streams
```



#### linux/crypto/lzo.c & lz4.c的代码
### lzo.c:
```c
static int lzo_compress(struct crypto_tfm *tfm, const u8 *src,
            unsigned int slen, u8 *dst, unsigned int *dlen)
{
    struct lzo_ctx *ctx = crypto_tfm_ctx(tfm);

    return __lzo_compress(src, slen, dst, dlen, ctx->lzo_comp_mem);
}

static int __lzo_compress(const u8 *src, unsigned int slen,
              u8 *dst, unsigned int *dlen, void *ctx)
{
    size_t tmp_len = *dlen; /* size_t(ulong) <-> uint on 64 bit */
    int err;

    err = lzo1x_1_compress(src, slen, dst, &tmp_len, ctx);

    if (err != LZO_E_OK)
        return -EINVAL;

    *dlen = tmp_len;
    return 0;
}
```
lzo_compress：
#### 函数参数解释

1. `tfm`: (`struct crypto_tfm *`) 是一个指向`crypto_tfm`结构体的指针，该结构体代表了加密或压缩算法的上下文。在这个上下文中包含了压缩算法的所有必要信息。
2. `src`: (`const u8 *`) 是一个指向源数据缓冲区的指针，该缓冲区包含了待压缩的数据。
3. `slen`: (`unsigned int`) 是源数据缓冲区的长度。
4. `dst`: (`u8 *`) 是一个指向目标数据缓冲区的指针，压缩后的数据将被写入该缓冲区。
5. `dlen`: (`unsigned int *`) 是一个指向变量的指针，该变量用于传递目标数据缓冲区的最大长度，并且在压缩完成后返回实际压缩数据的长度。
#### 函数内部

1. `struct lzo_ctx *ctx = crypto_tfm_ctx(tfm);`
   - 这行代码从`tfm`中提取出`lzo_ctx`结构体指针。`lzo_ctx`结构体包含了LZO压缩算法所需的所有上下文信息，例如压缩内存。
2. `return __lzo_compress(src, slen, dst, dlen, ctx->lzo_comp_mem);`
   - 这行代码调用了`__lzo_compress`函数来进行实际的压缩操作。`__lzo_compress`函数接收**源数据、源数据长度、目标数据缓冲区、目标数据长度以及压缩所需的内存**等参数。
   - `ctx->lzo_comp_mem`是压缩算法所需的额外内存，用于临时存储压缩过程中的中间数据。
#### 函数返回值

- 返回值是
```
int
```
类型，通常表示压缩操作的结果：

   - 如果压缩成功，通常会返回0。
   - 如果发生错误，通常会返回一个负数，表示错误码。
#### 注意事项

- 在调用`lzo_compress`之前，需要确保`tfm`已经正确初始化，并且`lzo_ctx`中的资源已经被正确分配。
- `dlen`参数应该指向一个足够大的缓冲区，以容纳压缩后的数据。通常需要预留足够的空间，因为有些情况下压缩后的数据大小可能会大于原始数据大小。
- `__lzo_compress`函数应该是LZO压缩算法的内部实现，可能不对外公开，仅在LZO压缩算法的内部使用。

这段代码提供了一个简洁的接口来执行LZO压缩操作，而具体的压缩逻辑则由`__lzo_compress`函数负责。如果你想要深入了解LZO压缩算法的实现细节，可以进一步查看`__lzo_compress`函数的定义以及`lzo_ctx`结构体的定义。
### __lzo_compress：
#### 函数参数解释

1. `src`: (`const u8 *`) 是一个指向源数据缓冲区的指针，该缓冲区包含了待压缩的数据。
2. `slen`: (`unsigned int`) 是源数据缓冲区的长度。
3. `dst`: (`u8 *`) 是一个指向目标数据缓冲区的指针，压缩后的数据将被写入该缓冲区。
4. `dlen`: (`unsigned int *`) 是一个指向变量的指针，该变量用于传递目标数据缓冲区的最大长度，并且在压缩完成后返回实际压缩数据的长度。
5. `ctx`: (`void *`) 是一个指向LZO压缩算法上下文的指针，包含了压缩算法所需的额外信息。
#### 函数内部

1. `size_t tmp_len = *dlen;`
   - 这行代码将`dlen`中的值复制到`tmp_len`中。由于`tmp_len`是`size_t`类型，而`dlen`是`unsigned int`类型，在64位系统中，`size_t`通常比`unsigned int`大，因此需要进行类型转换。这样做的目的是确保在调用`lzo1x_1_compress`时，`tmp_len`可以正确地接收压缩后的数据长度。
2. `err = lzo1x_1_compress(src, slen, dst, &tmp_len, ctx);`
   - 这行代码调用了LZO压缩库中的`lzo1x_1_compress`函数来执行实际的压缩操作。
   - `lzo1x_1_compress`函数接收源数据、源数据长度、目标数据缓冲区、目标数据长度的指针以及压缩所需的上下文等参数。
3. `if (err != LZO_E_OK)`
   - 这行代码检查`lzo1x_1_compress`函数的返回值。如果返回值不是`LZO_E_OK`（表示成功），则表示发生了错误。
4. `return -EINVAL;`
   - 如果压缩失败，返回`-EINVAL`，这是一个错误码，通常表示无效的参数。
5. `*dlen = tmp_len;`
   - 如果压缩成功，则将`tmp_len`中的值赋给`dlen`所指向的变量，更新目标数据缓冲区的实际长度。
6. `return 0;`
   - 如果压缩成功，返回0表示操作成功。
#### 函数返回值

- 返回值是
```
int
```
类型，通常表示压缩操作的结果：

   - 如果压缩成功，通常会返回0。
   - 如果发生错误，通常会返回一个负数，表示错误码。
#### 注意事项

- 确保`dlen`指向的缓冲区足够大，以容纳压缩后的数据。在最坏的情况下，压缩后的数据可能会比原始数据更大。
- `lzo1x_1_compress`函数是LZO压缩库提供的函数，用于执行压缩操作。确保已经正确链接了LZO库，并且在编译时包含了必要的头文件。

这段代码提供了LZO压缩算法的实现逻辑，使用了LZO库中的函数来执行压缩操作。如果你想要进一步了解LZO库的内部实现细节，可以参考LZO库的文档或源代码。

### lz4.c:
```c
static int lz4_scompress(struct crypto_scomp *tfm, const u8 *src,
             unsigned int slen, u8 *dst, unsigned int *dlen,
             void *ctx)
{
    return __lz4_compress_crypto(src, slen, dst, dlen, ctx);
}

static int __lz4_compress_crypto(const u8 *src, unsigned int slen,
                 u8 *dst, unsigned int *dlen, void *ctx)
{
    int out_len = LZ4_compress_default(src, dst,
        slen, *dlen, ctx);

    if (!out_len)
        return -EINVAL;

    *dlen = out_len;
    return 0;
}
```
lz4_scompress：
#### 函数参数解释

1. `tfm`: (`struct crypto_scomp *`) 是一个指向`crypto_scomp`结构体的指针，该结构体代表了压缩算法的上下文。在这个上下文中包含了压缩算法的所有必要信息。
2. `src`: (`const u8 *`) 是一个指向源数据缓冲区的指针，该缓冲区包含了待压缩的数据。
3. `slen`: (`unsigned int`) 是源数据缓冲区的长度。
4. `dst`: (`u8 *`) 是一个指向目标数据缓冲区的指针，压缩后的数据将被写入该缓冲区。
5. `dlen`: (`unsigned int *`) 是一个指向变量的指针，该变量用于传递目标数据缓冲区的最大长度，并且在压缩完成后返回实际压缩数据的长度。
6. `ctx`: (`void *`) 是一个指向压缩算法上下文的指针，包含了压缩算法所需的额外信息。
#### 函数内部
```
return __lz4_compress_crypto(src, slen, dst, dlen, ctx);
```

   - 这行代码直接调用了`__lz4_compress_crypto`函数来进行实际的压缩操作。
   - `__lz4_compress_crypto`函数接收源数据、源数据长度、目标数据缓冲区、目标数据长度的指针以及压缩所需的上下文等参数。
#### 函数返回值

- 返回值是
```
int
```
类型，通常表示压缩操作的结果：

   - 如果压缩成功，通常会返回0。
   - 如果发生错误，通常会返回一个负数，表示错误码。
#### 注意事项

- 在调用`lz4_scompress`之前，需要确保`tfm`已经正确初始化，并且`ctx`中的资源已经被正确分配。
- `dlen`参数应该指向一个足够大的缓冲区，以容纳压缩后的数据。通常需要预留足够的空间，因为有些情况下压缩后的数据大小可能会大于原始数据大小。
### 内部函数`__lz4_compress_crypto`
`__lz4_compress_crypto`函数是LZ4压缩算法的实际压缩逻辑实现。它的具体实现不在这段代码中给出，但我们可以推测其功能类似于`__lzo_compress`函数，即执行实际的压缩操作。通常，这个函数会利用LZ4库中的压缩算法来压缩数据。
如果你想要深入了解LZ4压缩算法的实现细节，可以查找`__lz4_compress_crypto`函数的定义，它应该在LZ4相关的源代码文件中。你可以在`drivers/staging/lz4`或`crypto/lz4`目录下寻找相关的源代码文件。
这段代码提供了一个简洁的接口来执行LZ4压缩操作，而具体的压缩逻辑则由`__lz4_compress_crypto`函数负责。


#### 函数参数解释

1. `src`: (`const u8 *`) 是一个指向源数据缓冲区的指针，该缓冲区包含了待压缩的数据。
2. `slen`: (`unsigned int`) 是源数据缓冲区的长度。
3. `dst`: (`u8 *`) 是一个指向目标数据缓冲区的指针，压缩后的数据将被写入该缓冲区。
4. `dlen`: (`unsigned int *`) 是一个指向变量的指针，该变量用于传递目标数据缓冲区的最大长度，并且在压缩完成后返回实际压缩数据的长度。
5. `ctx`: (`void *`) 是一个指向LZ4压缩算法上下文的指针，包含了压缩算法所需的额外信息。
#### 函数内部

1. `int out_len = LZ4_compress_default(src, dst, slen, *dlen, ctx);`
   - 这行代码调用了`LZ4_compress_default`函数来执行实际的压缩操作。
   - `LZ4_compress_default`函数接收源数据、目标数据缓冲区、源数据长度、目标数据缓冲区的最大长度以及压缩所需的上下文等参数。
   - `out_len`是压缩后的数据长度。
2. `if (!out_len)`
   - 这行代码检查`out_len`是否为0。如果`out_len`为0，这意味着压缩失败。
3. `return -EINVAL;`
   - 如果压缩失败，返回`-EINVAL`，这是一个错误码，通常表示无效的参数。
4. `*dlen = out_len;`
   - 如果压缩成功，则将`out_len`中的值赋给`dlen`所指向的变量，更新目标数据缓冲区的实际长度。
5. `return 0;`
   - 如果压缩成功，返回0表示操作成功。
#### 函数返回值

- 返回值是
```
int
```
类型，通常表示压缩操作的结果：

   - 如果压缩成功，通常会返回0。
   - 如果发生错误，通常会返回一个负数，表示错误码。
#### 注意事项

- 确保`dlen`指向的缓冲区足够大，以容纳压缩后的数据。在最坏的情况下，压缩后的数据可能会比原始数据更大。
- `LZ4_compress_default`函数是LZ4库提供的函数，用于执行压缩操作。确保已经正确链接了LZ4库，并且在编译时包含了必要的头文件。
### LZ4库的压缩函数
`LZ4_compress_default`函数是LZ4库提供的默认压缩函数，它使用LZ4算法来压缩数据。在LZ4库中，还有其他的压缩函数可供选择，如`LZ4_compress_HC`，它提供了更高的压缩率但可能需要更长的时间来压缩数据。
这段代码提供了LZ4压缩算法的实现逻辑，使用了LZ4库中的函数来执行压缩操作。如果你想要进一步了解LZ4库的内部实现细节，可以参考LZ4库的文档或源代码。

#### LZ4_compress_default:
```
int LZ4_compress_default(const char *source, char *dest, int inputSize,
    int maxOutputSize, void *wrkmem)
{
    return LZ4_compress_fast(source, dest, inputSize,
        maxOutputSize, LZ4_ACCELERATION_DEFAULT, wrkmem);
}


```



# 下一步
## 0 zram结构体
### 0.1 zram
这段代码展示了`struct zram`的定义，它是zRAM模块中用于表示一个zRAM设备的核心数据结构。`struct zram`包含了管理zRAM设备所需的各种资源和状态信息。下面是对这个结构体的详细解析：
```c
struct zram {
    struct zram_table_entry *table;
    struct zs_pool *mem_pool;
    struct zcomp *comps[ZRAM_MAX_COMPS];
    struct gendisk *disk;
    /* Prevent concurrent execution of device init */
    struct rw_semaphore init_lock;
    /*
     * the number of pages zram can consume for storing compressed data
     */
    unsigned long limit_pages;

    struct zram_stats stats;
    /*
     * This is the limit on amount of *uncompressed* worth of data
     * we can store in a disk.
     */
    u64 disksize; /* bytes */
    const char *comp_algs[ZRAM_MAX_COMPS];
    s8 num_active_comps;
    /*
     * zram is claimed so open request will be failed
     */
    bool claim; /* Protected by disk->open_mutex */
#ifdef CONFIG_ZRAM_WRITEBACK
    struct file *backing_dev;
    spinlock_t wb_limit_lock;
    bool wb_limit_enable;
    u64 bd_wb_limit;
    struct block_device *bdev;
    unsigned long *bitmap;
    unsigned long nr_pages;
#endif
#ifdef CONFIG_ZRAM_MEMORY_TRACKING
    struct dentry *debugfs_dir;
#endif
};
```
#### 结构体成员解释

1. `table`: (`struct zram_table_entry *`) 指向一个链表，用于存储zRAM设备中的**页面映射表**。
2. `mem_pool`: (`struct zs_pool *`) 指向一个内存池，用于**管理zRAM设备中压缩数据的存储**。
3. `comps`: (`struct zcomp *[]`) 一个数组，用于**存储zRAM设备上可用的压缩算法实例**。
4. `disk`: (`struct gendisk *`) 指向`gendisk`结构体的指针，用于**将zRAM设备注册为块设备**。
5. `init_lock`: (`struct rw_semaphore`) **读写信号量**，用于同步zRAM设备的初始化过程，防止并发初始化。
6. `limit_pages`: (`unsigned long`) zRAM设备**可以消耗的用于存储压缩数据的页面数量上限**。
7. `stats`: (`struct zram_stats`) 统计信息结构体，用于**记录zRAM设备的使用情况**，如压缩和解压缩的统计数据。
8. `disksize`: (`u64`) **zRAM设备的容量**，以字节为单位，**表示可以存储的未压缩数据的最大量**。
9. `comp_algs`: (`const char *[]`) 一个数组，用于**存储可用压缩算法的名称**。
10. `num_active_comps`: (`s8`) 当前活动的**压缩算法的数量**。
11. `claim`: (`bool`) 标记z**RAM设备是否已被声明**，如果设备已被声明，新的打开请求将被拒绝。

---

_**回写功能相关：**_

12. `backing_dev`: (`struct file *`) 当配置了回写功能时，**指向zRAM设备的回写设备文件的指针**。
13. `wb_limit_lock`: (`spinlock_t`) 用于**同步回写限制操作的自旋锁**。
14. `wb_limit_enable`: (`bool`) **标记回写限制是否启用**。
15. `bd_wb_limit`: (`u64`) **回写设备的写回限制量**。
16. `bdev`: (`struct block_device *`) 当配置了回写功能时，指向回写设备的`block_device`结构体的指针。

---

17. `bitmap`: (`unsigned long *`) **位图**，用于跟踪zRAM设备中的页面状态。
18. `nr_pages`: (`unsigned long`) zRAM设备中的**总页面数**。
19. `debugfs_dir`: (`struct dentry *`) 当配置了内存跟踪功能时，指向debugfs目录的指针，用于**调试和监控zRAM设备的内存使用情况**。
#### 总结
`struct zram`是zRAM模块中一个重要的数据结构，它封装了zRAM设备的所有核心信息和资源。通过这个结构体，内核能够有效地管理zRAM设备的内存、压缩算法、统计信息、设备状态和配置选项，从而实现高效的数据压缩和内存优化。在zRAM模块的实现中，`struct zram`是管理和操作zRAM设备的基础。
## 1 调研lzo压缩算法上下文
```c
static int lzo_compress(struct crypto_tfm *tfm, const u8 *src,
            unsigned int slen, u8 *dst, unsigned int *dlen)
{
    struct lzo_ctx *ctx = crypto_tfm_ctx(tfm);

    //ctx->lzo_comp_mem指的是压缩时的工作内存
    return __lzo_compress(src, slen, dst, dlen, ctx->lzo_comp_mem);	
}
```
### 1.1 struct crypto_tfm *tfm：
```c
struct crypto_tfm {
    refcount_t refcnt;		//引用计数器

    u32 crt_flags;			//与算法相关的标志（无符号整数）

    int node;				//标识算法上下文在内核中的位置或编号（可能）
    
    void (*exit)(struct crypto_tfm *tfm);	//一个函数指针，指向一个清理函数
    
    struct crypto_alg *__crt_alg;			//算法的描述信息

    void *__crt_ctx[] CRYPTO_MINALIGN_ATTR;	//灵活数组成员，用于存储算法上下文所需的数据
};
```
#### 结构体成员解释

1. `refcnt`: (`refcount_t`) 是一个引用计数器，用于跟踪有多少引用指向这个`crypto_tfm`结构体实例。当引用计数变为0时，表示可以安全地释放与之关联的资源。
2. `crt_flags`: (`u32`) 是一个无符号整数，用于存储与算法相关的标志。这些标志可以表示算法的属性或状态。
3. `node`: (`int`) 可能用于标识算法上下文在内核中的位置或编号。
4. `exit`: (`void (*)(struct crypto_tfm *))` 是一个函数指针，指向一个清理函数。当不再需要该`crypto_tfm`实例时，这个函数会被调用来释放资源。
5. `__crt_alg`: (`struct crypto_alg *`) 是一个指向`crypto_alg`结构体的指针，该结构体包含了算法的描述信息。
6. `__crt_ctx[]`: (`void []`) 是一个灵活数组成员（Flexible Array Member, FAM），用于存储算法上下文所需的数据。`CRYPTO_MINALIGN_ATTR`是一个属性，确保数组元素的对齐满足最低要求。
#### 注意事项

- `crypto_tfm`结构体是内核中加密或压缩算法的上下文，包含了执行算法所需的全部信息。
- `refcnt`成员用于实现引用计数机制，确保当最后一个引用消失时可以安全地释放资源。
- `exit`成员指向的函数用于清理与`crypto_tfm`实例相关的资源。
- `__crt_ctx`成员是一个灵活数组成员，其大小在创建`crypto_tfm`实例时确定。这个数组用于存储特定于算法的数据，如密钥或其他状态信息。
#### 1.1.1 refcount_t：
```cpp
typedef struct refcount_struct {
    atomic_t refs;
} refcount_t;
```
`atomic_t refs`: 这是一个原子类型的整数，用于存储引用计数。`atomic_t`是一种特殊的整数类型，它保证了对整数值的读写操作是原子的，也就是说，这些操作不会被其他并发线程打断。这对于多线程环境非常重要，因为多个线程可能同时增加或减少引用计数。
##### 使用`refcount_t`
`refcount_t`类型的变量通常通过一组内核提供的原子操作函数来操作，以确保线程安全。这些函数包括但不限于：

- `refcount_read()`: 读取引用计数的值。
- `refcount_dec_and_test()`: 减少引用计数，并检查结果是否为零。如果为零，返回非零值，否则返回零。
- `refcount_inc()`: 增加引用计数。
- `refcount_set()`: 设置引用计数的值。
##### 示例代码
以下是如何使用`refcount_t`的示例：
```c
#include <linux/refcount.h>

refcount_t my_refcount;

// 初始化引用计数
INIT_REFCOUNT(&my_refcount);

// 增加引用计数
refcount_inc(&my_refcount);

// 减少引用计数并检查是否为零
if (refcount_dec_and_test(&my_refcount)) {
    // 引用计数为零，可以安全地释放资源
    // ...
}

// 读取引用计数
unsigned long count = refcount_read(&my_refcount);
```
#### 1.1.2 struct crypto_alg *__crt_alg;
//算法的描述信息
```c
struct crypto_alg {
    struct list_head cra_list;				//全局算法列表
    struct list_head cra_users;				//使用该算法的用户列表

    u32 cra_flags;							//算法的标志位
    unsigned int cra_blocksize;				//块大小
    unsigned int cra_ctxsize;				//算法上下文所需的内存大小
    unsigned int cra_alignmask;				//对数据对齐的要求

    int cra_priority;						//算法的优先级
    refcount_t cra_refcnt;					//引用计数

    char cra_name[CRYPTO_MAX_ALG_NAME];		//算法的名称
    char cra_driver_name[CRYPTO_MAX_ALG_NAME];	//驱动的名称

    const struct crypto_type *cra_type;		//算法的类型

    union {							//存储算法特定的数据结构
        struct cipher_alg cipher;
        struct compress_alg compress;
    } cra_u;

    int (*cra_init)(struct crypto_tfm *tfm);	//初始化函数
    void (*cra_exit)(struct crypto_tfm *tfm);	//清理函数
    void (*cra_destroy)(struct crypto_alg *alg);//销毁函数
    
    struct module *cra_module;		//指向加载此算法的模块
} CRYPTO_MINALIGN_ATTR;
```
##### 结构体成员解释

1. `cra_list`: (`struct list_head`) 用于将算法链接到全局算法列表中，便于管理和查找。
2. `cra_users`: (`struct list_head`) 用于维护使用该算法的用户列表。
3. `cra_flags`: (`u32`) 算法的标志位，例如是否支持硬件加速、是否是软件实现等。
4. `cra_blocksize`: (`unsigned int`) 算法的块大小，对于块加密算法尤其重要。
5. `cra_ctxsize`: (`unsigned int`) 算法上下文（context）所需的内存大小。
6. `cra_alignmask`: (`unsigned int`) 算法对数据对齐的要求，用于优化性能。
7. `cra_priority`: (`int`) 算法的优先级，影响算法的选择顺序。
8. `cra_refcnt`: (`refcount_t`) 引用计数，用于跟踪算法实例的使用情况。
9. `cra_name`: (`char [CRYPTO_MAX_ALG_NAME]`) 算法的名称。
10. `cra_driver_name`: (`char [CRYPTO_MAX_ALG_NAME]`) 算法驱动的名称，用于区分不同的实现。
11. `cra_type`: (`const struct crypto_type *`) 算法的类型，如加密、散列、压缩等。
12. `cra_u`: (`union`) 一个联合体，用于存储算法特定的数据结构，如`cipher_alg`或`compress_alg`。
13. `cra_init`: (`int (*)(struct crypto_tfm *))` 初始化函数，用于准备算法实例。
14. `cra_exit`: (`void (*)(struct crypto_tfm *))` 清理函数，用于释放算法实例的资源。
15. `cra_destroy`: (`void (*)(struct crypto_alg *))` 销毁函数，用于释放算法描述符的资源。
16. `cra_module`: (`struct module *`) 指向加载此算法的模块，用于模块卸载时清理算法。
### 1.2 struct lzo_ctx *ctx = crypto_tfm_ctx(tfm);
#### 1.2.1 struct lzo_ctx
```cpp
struct lzo_ctx {
    void *lzo_comp_mem;
};
```
`lzo_comp_mem`: (`void *`) 这是一个指针，指向LZO压缩算法在执行压缩操作时使用的**内部工作内存**。工作内存用于存储压缩过程中产生的中间数据，例如滑动窗口、哈希表等，这些数据有助于算法提高压缩效率和比率。
#### 1.2.2 crypto_tfm_ctx(tfm);
```cpp
static inline void *crypto_tfm_ctx(struct crypto_tfm *tfm)
{
	return tfm->__crt_ctx;
}
```
`tfm`: (`struct crypto_tfm *`) 这是一个指向`crypto_tfm`结构体的指针，代表了一个加密或压缩算法的实例。
##### 返回值
函数返回一个`void *`类型的指针，指向`tfm`结构体中的`__crt_ctx`成员。`__crt_ctx`成员是一个灵活数组成员，用于存储特定于算法的上下文数据，如密钥、状态信息或其他算法需要的信息。
## 2 如何选择不同的压缩算法
**内存压缩算法**
**【调优节点】：/sys/block/zram0/comp_algorithm**
```
linux-6.10/Documentation/ABI/testing/sysfs-block-zram:What:             /sys/block/zram<id>/comp_algorithm
linux-6.10/Documentation/ABI/testing/sysfs-block-zram:          The comp_algorithm file is read-write and lets to show
linux-6.10/Documentation/ABI/testing/sysfs-block-zram:What:             /sys/block/zram<id>/recomp_algorithm
linux-6.10/Documentation/ABI/testing/sysfs-block-zram:          The recomp_algorithm file is read-write and allows to set
linux-6.10/Documentation/admin-guide/blockdev/zram.rst:Using comp_algorithm device attribute one can see available and
linux-6.10/Documentation/admin-guide/blockdev/zram.rst: cat /sys/block/zram0/comp_algorithm
linux-6.10/Documentation/admin-guide/blockdev/zram.rst: echo lzo > /sys/block/zram0/comp_algorithm
linux-6.10/Documentation/admin-guide/blockdev/zram.rst:For the time being, the `comp_algorithm` content does not necessarily
linux-6.10/Documentation/admin-guide/blockdev/zram.rst:`comp_algorithm`. The thing is that, internally, ZRAM uses Crypto API
linux-6.10/Documentation/admin-guide/blockdev/zram.rst:comp_algorithm           RW show and change the compression algorithm
linux-6.10/Documentation/admin-guide/blockdev/zram.rst:using recomp_algorithm device attribute.
linux-6.10/Documentation/admin-guide/blockdev/zram.rst: cat /sys/block/zramX/recomp_algorithm
linux-6.10/Documentation/admin-guide/blockdev/zram.rst: echo "algo=zstd priority=1" > /sys/block/zramX/recomp_algorithm
linux-6.10/Documentation/admin-guide/blockdev/zram.rst: echo "algo=deflate priority=2" > /sys/block/zramX/recomp_algorithm
linux-6.10/drivers/block/zram/zram_drv.c:static void comp_algorithm_set(struct zram *zram, u32 prio, const char *alg)
linux-6.10/drivers/block/zram/zram_drv.c:static ssize_t __comp_algorithm_show(struct zram *zram, u32 prio, char *buf)
linux-6.10/drivers/block/zram/zram_drv.c:static int __comp_algorithm_store(struct zram *zram, u32 prio, const char *buf)
linux-6.10/drivers/block/zram/zram_drv.c:       comp_algorithm_set(zram, prio, compressor);
linux-6.10/drivers/block/zram/zram_drv.c:static ssize_t comp_algorithm_show(struct device *dev,
linux-6.10/drivers/block/zram/zram_drv.c:       return __comp_algorithm_show(zram, ZRAM_PRIMARY_COMP, buf);
linux-6.10/drivers/block/zram/zram_drv.c:static ssize_t comp_algorithm_store(struct device *dev,
linux-6.10/drivers/block/zram/zram_drv.c:       ret = __comp_algorithm_store(zram, ZRAM_PRIMARY_COMP, buf);
linux-6.10/drivers/block/zram/zram_drv.c:static ssize_t recomp_algorithm_show(struct device *dev,
linux-6.10/drivers/block/zram/zram_drv.c:               sz += __comp_algorithm_show(zram, prio, buf + sz);
linux-6.10/drivers/block/zram/zram_drv.c:static ssize_t recomp_algorithm_store(struct device *dev,
linux-6.10/drivers/block/zram/zram_drv.c:       ret = __comp_algorithm_store(zram, prio, alg);
linux-6.10/drivers/block/zram/zram_drv.c:       &dev_attr_recomp_algorithm.attr,
linux-6.10/drivers/block/zram/zram_drv.c:       comp_algorithm_set(zram, ZRAM_PRIMARY_COMP, default_compressor);
linux-6.10/tools/testing/selftests/zram/zram_lib.sh:    local algs=$(cat /sys/block/zram${i}/comp_algorithm)
linux-6.10/tools/testing/selftests/zram/zram_lib.sh:            local sys_path="/sys/block/zram${i}/comp_algorithm"
```

---

### 2.1 设置压缩算法函数
```c
static void comp_algorithm_set(struct zram *zram, u32 prio, const char *alg)
{
    /* Do not free statically defined compression algorithms */
    if (zram->comp_algs[prio] != default_compressor)
        kfree(zram->comp_algs[prio]);

    zram->comp_algs[prio] = alg;
}
```
#### 函数参数解释

1. `zram`: (`struct zram *`) 是一个指向`zram`结构体的指针，代表了一个zRAM设备的实例。
2. `prio`: (`u32`) 是一个无符号整数，表示压缩算法的优先级。在zRAM中，压缩算法通常按照优先级顺序选择，`prio`值越小，优先级越高。
3. `alg`: (`const char *`) 是一个指向字符串的指针，该字符串表示要设置的压缩算法的名称。
#### 函数内部

1. `if (zram->comp_algs[prio] != default_compressor)`
   - 这行代码检查`zram`设备当前在优先级`prio`处是否使用的是默认压缩算法。如果不是默认算法，说明之前设置了其他算法，需要释放旧的算法资源。
1. `kfree(zram->comp_algs[prio]);`
   - 如果当前算法不是默认算法，调用`kfree`函数来释放之前设置的压缩算法资源。
1. `zram->comp_algs[prio] = alg;`
   - 将新的压缩算法名称`alg`设置到`zram`设备的`comp_algs`数组中对应优先级的位置。
#### 注意事项

- `default_compressor`通常是一个静态定义的压缩算法，不应该被释放。这是因为静态定义的算法通常在编译时就已经确定，且在整个运行周期内保持不变。
- `comp_algs`是`zram`结构体中的一个数组，用于存储不同优先级的压缩算法。每个`comp_algs[prio]`元素都可能是一个指向压缩算法上下文的指针。
#### 总结
`comp_algorithm_set`函数用于在zRAM设备中设置压缩算法。它首先检查并释放任何非默认的旧算法资源，然后设置新的压缩算法。这个函数是zRAM模块中用于动态配置压缩策略的重要组成部分，使得用户可以根据需要选择不同的压缩算法，以优化内存使用和系统性能。
需要注意的是，实际的代码中可能还会有更多的错误检查和同步机制，以确保多线程环境下的正确性和资源的安全释放。此外，`comp_algs`数组中的元素类型在实际代码中可能不是简单的字符串指针，而是指向更复杂的算法上下文结构体的指针。
### 2.2 显示压缩算法函数
这段代码展示了`__comp_algorithm_show`函数，它是zRAM模块中用于显示压缩算法信息的函数。下面是对这段代码的详细解析：
```c
static ssize_t __comp_algorithm_show(struct zram *zram, u32 prio, char *buf)
{
    ssize_t sz;

    down_read(&zram->init_lock);
    sz = zcomp_available_show(zram->comp_algs[prio], buf);
    up_read(&zram->init_lock);

    return sz;
}
```
#### 函数参数解释

1. `zram`: (`struct zram *`) 是一个指向`zram`结构体的指针，代表了一个zRAM设备的实例。
2. `prio`: (`u32`) 是一个无符号整数，表示压缩算法的优先级。在zRAM中，压缩算法通常按照优先级顺序选择，`prio`值越小，优先级越高。
3. `buf`: (`char *`) 是一个指向字符数组的指针，用于接收压缩算法的信息字符串。
#### 函数内部

1. `down_read(&zram->init_lock);`
   - 这行代码获取`zram`设备上的`init_lock`读锁。读锁允许多个读操作同时进行，但阻止写操作，以确保在读取压缩算法信息时不会发生数据竞争。
1. `sz = zcomp_available_show(zram->comp_algs[prio], buf);`
   - 调用`zcomp_available_show`函数来获取优先级`prio`处的压缩算法信息，并将其写入到`buf`中。`sz`返回写入的字节数。
1. `up_read(&zram->init_lock);`
   - 释放`zram`设备上的`init_lock`读锁，允许其他读写操作继续。
#### 函数返回值

- 返回值是`ssize_t`类型，表示写入`buf`中的字节数。如果函数执行失败，通常会返回负数。
#### 注意事项

- `zcomp_available_show`函数是用于显示压缩算法信息的辅助函数，它通常会根据压缩算法上下文构建并返回一个描述压缩算法的字符串。
- `init_lock`是一个读写锁，用于保护`zram`设备的初始化状态和关键数据结构。读锁用于读取操作，而写锁用于修改`zram`设备的状态，如设置压缩算法或初始化设备。
#### 总结
`__comp_algorithm_show`函数用于在zRAM设备中显示指定优先级的压缩算法信息。它使用读写锁来确保在多线程环境中读取压缩算法信息的一致性和安全性。通过调用`zcomp_available_show`函数，它能够获取并返回压缩算法的描述信息，这对于调试和监控zRAM设备的压缩策略非常有用。
在实际的zRAM模块中，`__comp_algorithm_show`函数可能还会包含更多的错误检查和资源管理代码，以确保函数的健壮性和稳定性。
### 2.3 存储压缩算法函数
这段代码展示了`__comp_algorithm_store`函数，它是zRAM模块中用于存储（即设置）压缩算法的函数。下面是对这段代码的详细解析：
```c
static int __comp_algorithm_store(struct zram *zram, u32 prio, const char *buf)
{
    char *compressor;
    size_t sz;

    sz = strlen(buf);
    if (sz >= CRYPTO_MAX_ALG_NAME)
        return -E2BIG;

    compressor = kstrdup(buf, GFP_KERNEL);
    if (!compressor)
        return -ENOMEM;

    /* ignore trailing newline */
    if (sz > 0 && compressor[sz - 1] == '\n')
        compressor[sz - 1] = 0x00;

    if (!zcomp_available_algorithm(compressor)) {
        kfree(compressor);
        return -EINVAL;
    }

    down_write(&zram->init_lock);
    if (init_done(zram)) {
        up_write(&zram->init_lock);
        kfree(compressor);
        pr_info("Can't change algorithm for initialized device\n");
        return -EBUSY;
    }

    comp_algorithm_set(zram, prio, compressor);
    up_write(&zram->init_lock);
    return 0;
}
```
### 函数参数解释

1. `zram`: (`struct zram *`) 是一个指向`zram`结构体的指针，代表了一个zRAM设备的实例。
2. `prio`: (`u32`) 是一个无符号整数，表示压缩算法的优先级。
3. `buf`: (`const char *`) 是一个指向字符数组的指针，包含用户输入的压缩算法名称。
### 函数内部

1. `sz = strlen(buf);`
   - 获取用户输入的压缩算法名称的长度。
1. `if (sz >= CRYPTO_MAX_ALG_NAME)`
   - 检查压缩算法名称的长度是否超过最大允许长度，如果是，则返回`-E2BIG`错误。
1. `compressor = kstrdup(buf, GFP_KERNEL);`
   - 使用`kstrdup`函数复制用户输入的压缩算法名称到内核堆栈，以避免后续对`buf`的修改影响到用户空间。
1. `if (sz > 0 && compressor[sz - 1] == '\n')`
   - 移除压缩算法名称末尾的换行符，如果存在的话。
1. `if (!zcomp_available_algorithm(compressor))`
   - 检查输入的压缩算法是否可用，如果不可用，则释放`compressor`并返回`-EINVAL`错误。
1. `down_write(&zram->init_lock);`
   - 获取`zram`设备上的`init_lock`写锁，阻止其他写操作，确保设置压缩算法时的一致性。
1. `if (init_done(zram))`
   - 检查zRAM设备是否已经被初始化，如果是，则释放锁、释放`compressor`并返回`-EBUSY`错误，表示不能在设备初始化后更改压缩算法。
1. `comp_algorithm_set(zram, prio, compressor);`
   - 调用`comp_algorithm_set`函数来设置压缩算法。
1. `up_write(&zram->init_lock);`
   - 释放`zram`设备上的`init_lock`写锁。
### 函数返回值

- 返回值是`int`类型，表示设置压缩算法的结果。如果成功，返回0；如果失败，返回相应的错误码。
### 注意事项

- `kstrdup`函数用于在内核堆栈中复制字符串，使用`GFP_KERNEL`标志来请求内核内存。
- `init_lock`是一个读写锁，用于保护`zram`设备的初始化状态。写锁用于设置压缩算法这样的写操作，防止在设置算法时发生数据竞争。
- `init_done(zram)`是一个用于检查zRAM设备初始化状态的函数，如果设备已经初始化，不允许再更改压缩算法。
### 总结
`__comp_algorithm_store`函数用于设置zRAM设备的压缩算法。它首先验证输入的算法名称，然后检查算法是否可用，最后在确保设备未初始化的情况下设置压缩算法。通过使用读写锁，它保证了在多线程环境中设置压缩算法的一致性和安全性。如果设备已经初始化，该函数将拒绝设置压缩算法，以避免潜在的数据丢失或设备状态混乱。
## 3 创建一个zRAM压缩任务，从上到下调用关系
##### 加载zRAM模块
首先，确保zRAM模块已经加载到内核中。这可以通过手动加载模块完成：
```c
modprobe zram
```
##### 创建zRAM设备
接下来，创建一个zRAM设备。这通常通过写入到/sys/block/zram0目录下的某些文件来完成，但实际上，底层会调用到的内核函数包括：
```c
zram_create_device(): 	//在内核中创建一个新的zRAM设备实例。
```
##### 配置压缩算法
配置zRAM设备所使用的压缩算法。这可以通过修改/sys/block/zram0/comp_algorithm文件来完成。在内核层面，这涉及到调用：
```c
zram_set_compressor(): //设置zRAM设备的压缩算法。
```
##### 初始化zRAM设备
初始化zRAM设备，使其准备好接受写入操作。这包括：
```c
zram_init_device(): //初始化设备，准备压缩算法和其他相关资源。
```
##### 注册zRAM设备
将新创建的zRAM设备注册为块设备，使其在系统中可见。这涉及到：
```c
register_blkdev(): //将zRAM设备注册为块设备。
```
##### 配置和启用zRAM设备
最后，配置和启用zRAM设备，使其开始接受和压缩数据。这包括：
```c
zram_set_params(): //设置zRAM设备的参数，如压缩级别等。
zram_enable_device(): //启用zRAM设备，使其开始工作。
```
##### 具体函数调用流程
当你通过modprobe zram加载模块时，内核会调用zram_init()函数，该函数会初始化zRAM模块。
zram_init()函数会调用zram_create_device()来创建一个或多个zRAM设备。
每个设备创建后，zram_init_device()会被调用来初始化设备。
接着，zram_set_compressor()会被调用来根据comp_algorithm设置压缩算法。
最后，register_blkdev()将设备注册为块设备，使其对系统可见。
```c
void create_zram_device(void)
{
    struct zram *zram_dev;
    // 创建zRAM设备
    zram_dev = zram_create_device();

    if (zram_dev) {
        // 初始化设备
        zram_init_device(zram_dev);

        // 设置压缩算法
        zram_set_compressor(zram_dev, COMP_LZ4); // 例如使用LZ4压缩

        // 配置设备参数
        zram_set_params(zram_dev, /* parameters */);

        // 注册设备为块设备
        register_blkdev(zram_dev->blk_dev, "zram0");

        // 启用设备
        zram_enable_device(zram_dev);
    }
}
```
请注意，以上代码是简化的示例，实际的内核代码会更复杂，涉及到更多的错误检查、同步机制和资源管理。此外，很多操作都是通过系统调用或内核空间的函数调用来间接完成的，而不是直接调用上述函数。例如，modprobe命令最终会触发zram_init()函数的执行，而/sys/block/zram0下的文件操作会调用对应的内核文件操作函数。
##### zRAM读写流程
写（压缩）流程：
__zram_bvec_write:![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429188635-f283b947-b4cf-4c97-bd3b-0519f3d7f779.png#averageHue=%23c9e4aa&clientId=u3fa46190-4897-4&from=paste&height=585&id=d4m0D&originHeight=877&originWidth=1120&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=452163&status=done&style=none&taskId=u25fcd7a8-0e5d-49a2-8938-090c77a61a2&title=&width=746.6666666666666)
读（解压缩）流程：
__zram_bvec_read:![image.png](https://cdn.nlark.com/yuque/0/2024/png/46686475/1722429197892-68ff625b-9173-41f3-beea-2edc4f26c726.png#averageHue=%23c5e1a5&clientId=u3fa46190-4897-4&from=paste&height=666&id=qfyLV&originHeight=999&originWidth=1129&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=474140&status=done&style=none&taskId=u20008765-f21a-4b59-8761-0498adf05d9&title=&width=752.6666666666666)
## 4 怎么从swap到zram
## 5 整一个压缩流程的接口都需要搞清楚
#### 每个函数调用的什么东西，需要哪些变量，这些变量存了什么东西，具体是什么，干了什么
## 6 数据流大小多少
zRAM模块在内核层面确实有一些与数据块大小相关的限制和考量，这些限制主要由以下几个方面构成：

1. **块设备限制**：zRAM作为一个块设备，遵循Linux内核中块设备的一般规则。块设备通常处理固定大小的数据块，zRAM也不例外。zRAM的块大小（即一次读写操作的数据量）通常与系统中的其他块设备相匹配，以确保数据的一致性和性能。这个块大小通常为4KB、8KB或更大的倍数，但并非一个严格的“最大数据块”的概念，而是数据交换的基本单元大小。
2. **压缩算法限制**：zRAM使用的压缩算法（如LZ4、ZSTD等）可能有自己的内部限制，比如滑动窗口大小、缓冲区大小等，这些会影响可以一次性处理的数据量。然而，这些限制通常是算法设计的细节，对于zRAM的使用者来说通常是透明的。
3. **内存页大小**：在Linux中，内存页的大小通常为4KB，这是内核管理内存的基本单位。zRAM在压缩和解压缩数据时，通常以页为单位进行操作。因此，从某种意义上说，zRAM的“最大数据块”可以认为是内存页的大小，但实际操作中，它可以处理任意长度的连续数据流，只是内部会按页进行分割和处理。
4. **系统资源限制**：zRAM模块本身以及压缩算法的运行需要消耗系统资源，包括CPU时间和内存。如果系统资源有限，如CPU速度较慢或可用内存不足，zRAM处理数据的能力也会受到影响，这可能会表现为处理数据流时的性能瓶颈。
5. **zRAM设备大小限制**：如前所述，zRAM设备的大小可以通过`mem_limit`参数来设置，这实际上限制了zRAM可以使用的物理内存总量。这间接地影响了zRAM可以处理的数据量，因为一旦达到这个限制，zRAM将不能再接受新的数据进行压缩。

综上所述，虽然zRAM在内核层面没有一个严格意义上的“最大数据块”限制，但在实际操作中，它受到块设备规则、压缩算法特性、系统资源可用性以及设备大小限制等因素的影响。在设计和优化系统时，需要考虑到这些限制，以确保zRAM模块可以高效地运行在特定的硬件和软件环境中。
## 7 虚拟机实验
