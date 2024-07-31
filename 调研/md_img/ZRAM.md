- [ ] **zRAM 块设备驱动的实现代码主要在 `drivers/block/zram/zram_drv.c` 文件中**
- [ ] **压缩函数：`ret = zcomp_compress(zram->comp, zstrm, uncmem, &clen);`**



### ZRAM内存压缩机制

[一文读懂｜zRAM 内存压缩机制_linux zram-CSDN博客](https://blog.csdn.net/youzhangjing_/article/details/133685959)

zRAM 的原理是：将进程不常用的内存压缩存储，从而达到节省内存的使用。

zRAM 机制建立在 swap 机制之上，swap 机制是将进程不常用的内存交换到磁盘中，而 zRAM 机制是将进程不常用的内存压缩存储在内存某个区域。所以 zRAM 机制并不会发生 I/O 操作，从而避免因 I/O 操作导致的性能下降。

--zRAM使用开辟的少量内存空间和一些cpu时间来换取比与磁盘IO更少的时间开销。

由于 zRAM 机制是建立在 swap 机制之上，而 swap 机制需要配置 `文件系统` 或 `块设备` 来完成的。所以 zRAM 虚拟一个块设备，当系统内存不足时，swap 机制将内存写入到这个虚拟的块设备中。也就是说，zRAM 机制本质上只是一个虚拟块设备。



（启用zRAM，创建zRAM块设备，设置zRAM块设备的大小，压缩算法选择（lzo，lz4），将swap交换设备设置成zRAM



#### zRAM实现

**zRAM 块设备驱动的实现代码主要在 `drivers/block/zram/zram_drv.c` 文件中**

下仅分析 swap 机制在进行内存交换时，与 zRAM 块设备驱动的交互。

当 swap 机制将不常用的内存交换到 zRAM 块设备时，会调用 `zram_make_request()` 函数处理请求。而 `zram_make_request()` 最终会通过调用 `zram_bvec_write()` 函数来压缩内存，调用链如下：

```c++
zram_make_request()
 -> __zram_make_request()
     -> zram_bvec_rw()
         -> zram_bvec_write()
    ///////////////////////////////
    		->zram_write_page()
    			->zcomp_compress()		//zcomp.c
    				->crypto_comp_compress()
```

```c++
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

**压缩函数：`ret = zcomp_compress(zram->comp, zstrm, uncmem, &clen);`**

函数所用参数：zram->comp

​							zstrm

​							uncmem

​							&clen







由于 zRAM 块设备是建立在内存中的虚拟块设备，所以其并没有真实块设备的特性。真实块设备会将存储空间划分成一个个块，而 zram_bvec_write() 函数的 index 参数就是数据块的编号。此参数有 swap 机制提供，所以 zRAM 块设备驱动通过 index 参数作为原始内存数据的编号。

![img](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\format,png)

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

![image-20240725122516387](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725122516387.png)

#### 3. zRAM实现分析

##### 3.1 zRAM驱动模块

- zram_init

zram_init注册了zram块设备驱动,流程如下:

![image-20240725122634054](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725122634054.png)

- disksize_store

创建zram设备驱动后,通过**用户态节点**配置zram块设备大小， 对应disksize_store函数。

![image-20240725122843657](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725122843657.png)

- zram_make_request

所有zram的块设备请求都是通过zram_make_request进行的。

![image-20240725135855098](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725135855098.png)

##### 3.2数据流模块

创建压缩数据流程如下：

![image-20240725140018928](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725140018928.png)

之所以要针对每个在线CPU都创建一个压缩数据流，主要是优化系统中并发的情况。每个写压缩操作都独享一个压缩数据流，通过多个压缩数据流，使得系统允许多个并发压缩操作。

##### 3.3压缩算法模块

zRAM默认使用lzo压缩算法，还有很多压缩算法可以使用。包括有lz4压缩算法。

##### 3.4zRAM读写流程

写（压缩）流程：

![image-20240725140745554](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725140745554.png)

读（解压缩）流程：

![image-20240725140804180](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725140804180.png)

##### 3.5zRAM writeback功能

zRAM是将内存压缩后存放起来，仍然是在在RAM中。如果有大量一次性访问页面被压缩后很长时间没有被再次被访问， 虽然经过压缩但仍然占内存。zRAM支持设置外部存储分区作为zRAM的backing_dev，对不可压缩内存（ZRAM_HUGE）和长时间没有被访问过的内存（ZRAM_IDLE）回写到外部存储中。

以Android手机为例， 如果配置开启了zram writeback有idle回写功能，流程如下：

![image-20240725141052056](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725141052056.png)

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

##### 4.4<u>zram的vma预读</u>

【调优节点】：/sys/kernel/mm/swap/vma_ra_enabled

预读始终是swap机制的优化方向之一。上面提到的基于page-cluster的预读可能不一定适合zram。内核工程师们提出了基于<u>VMA</u>地址的预读， 对于zRAM可能更符合局部性原理。

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

linux5.8支持把swappiness设置得比100还大， 最大值从100修改为了200， 提交[[4\]](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F565651762%23ref_4)已经进入5.8版本主线。如下：

```
mm: allow swappiness that prefers reclaiming anon over the file workingset
```

文档同时也更新了说明， 对于zram设备建议swappiness大于100。

```
For in-memory swap, like zram or zswap, [...] values beyond 100 can be considered. For example, if the random IO against the swap device is on average 2x faster than IO from the filesystem, swappiness should be 133 (x + 2x = 200, 2x = 133.33).

```

##### 4.6zRAM去重

【调优节点】：/sys/block/zram0/ dedup_enable

Android 是使用 zram 作为swap device的最大用户之一，节省zram内存使用量非常重要。有研究论文《MemScope: Analyzing Memory Duplication on Android Systems》[[5\]](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F565651762%23ref_5)表明Android的内存内容的重复率是相当高的。

Android 应用进程和系统进程可能表现出不同的特点，因为 Android 应用程序运行 dalvik VM 但非应用程序进程不运行。应用进程的内存内容重复率平均约14%，系统进程的内存内容重复率平均约3%。

![image-20240725143555501](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725143555501.png)

论文调查了哪些内存区域包含最多重复页面，结果是超过 80% 的重复页面来自来自匿名页、gralloc-buffer、dalvik-heap、dalvik-zygote和 libdvm.so 库的 BSS 段。如下图：

![image-20240725143620721](D:\Study\R&D\Project\20240718YiDongProj\调研\md_img\image-20240725143620721.png)

于是内核工程师们提出了zram支持去重功能以节省功能，提交集[[6\]](https://link.juejin.cn/?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F565651762%23ref_6)如下：

```
zram: implement deduplication in zram
```

可以通过/sys/block/zram/io_stat/中zram->dedup->dedups查看重复的page量。

##### 4.7多种压缩算法组合压缩

社区在讨论一个zram多重压缩的优化。可以在压缩后再次变更压缩算法对idle和huge的page再次压缩。基本思想是使用一种更快但压缩率较低的默认算法率和可以使用更高压缩率的辅助算法。以较慢的压缩/解压缩为代价。替代压缩算法可以提供更好的压缩比，以减少 zsmalloc 内存使用。当然这个改动[[7\]](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F565651762%23ref_7)还没有合入到主线中。

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

<u>lzo_compress</u>：

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

<u>__lzo_compress：</u>

### 函数参数解释

1. `src`: (`const u8 *`) 是一个指向源数据缓冲区的指针，该缓冲区包含了待压缩的数据。
2. `slen`: (`unsigned int`) 是源数据缓冲区的长度。
3. `dst`: (`u8 *`) 是一个指向目标数据缓冲区的指针，压缩后的数据将被写入该缓冲区。
4. `dlen`: (`unsigned int *`) 是一个指向变量的指针，该变量用于传递目标数据缓冲区的最大长度，并且在压缩完成后返回实际压缩数据的长度。
5. `ctx`: (`void *`) 是一个指向LZO压缩算法上下文的指针，包含了压缩算法所需的额外信息。

### 函数内部

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

### 函数返回值

- 返回值是

  ```
  int
  ```

  类型，通常表示压缩操作的结果：

  - 如果压缩成功，通常会返回0。
  - 如果发生错误，通常会返回一个负数，表示错误码。

### 注意事项

- 确保`dlen`指向的缓冲区足够大，以容纳压缩后的数据。在最坏的情况下，压缩后的数据可能会比原始数据更大。
- `lzo1x_1_compress`函数是LZO压缩库提供的函数，用于执行压缩操作。确保已经正确链接了LZO库，并且在编译时包含了必要的头文件。

这段代码提供了LZO压缩算法的实现逻辑，使用了LZO库中的函数来执行压缩操作。如果你想要进一步了解LZO库的内部实现细节，可以参考LZO库的文档或源代码。



lz4.c:

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

<u>lz4_scompress：</u>

#### 函数参数解释

1. `tfm`: (`struct crypto_scomp *`) 是一个指向`crypto_scomp`结构体的指针，该结构体代表了压缩算法的上下文。在这个上下文中包含了压缩算法的所有必要信息。
2. `src`: (`const u8 *`) 是一个指向源数据缓冲区的指针，该缓冲区包含了待压缩的数据。
3. `slen`: (`unsigned int`) 是源数据缓冲区的长度。
4. `dst`: (`u8 *`) 是一个指向目标数据缓冲区的指针，压缩后的数据将被写入该缓冲区。
5. `dlen`: (`unsigned int *`) 是一个指向变量的指针，该变量用于传递目标数据缓冲区的最大长度，并且在压缩完成后返回实际压缩数据的长度。
6. `ctx`: (`void *`) 是一个指向压缩算法上下文的指针，包含了压缩算法所需的额外信息。

#### 函数内部

1. ```
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

#### 内部函数`__lz4_compress_crypto`

`__lz4_compress_crypto`函数是LZ4压缩算法的实际压缩逻辑实现。它的具体实现不在这段代码中给出，但我们可以推测其功能类似于`__lzo_compress`函数，即执行实际的压缩操作。通常，这个函数会利用LZ4库中的压缩算法来压缩数据。

如果你想要深入了解LZ4压缩算法的实现细节，可以查找`__lz4_compress_crypto`函数的定义，它应该在LZ4相关的源代码文件中。你可以在`drivers/staging/lz4`或`crypto/lz4`目录下寻找相关的源代码文件。

这段代码提供了一个简洁的接口来执行LZ4压缩操作，而具体的压缩逻辑则由`__lz4_compress_crypto`函数负责。





### 函数参数解释

1. `src`: (`const u8 *`) 是一个指向源数据缓冲区的指针，该缓冲区包含了待压缩的数据。
2. `slen`: (`unsigned int`) 是源数据缓冲区的长度。
3. `dst`: (`u8 *`) 是一个指向目标数据缓冲区的指针，压缩后的数据将被写入该缓冲区。
4. `dlen`: (`unsigned int *`) 是一个指向变量的指针，该变量用于传递目标数据缓冲区的最大长度，并且在压缩完成后返回实际压缩数据的长度。
5. `ctx`: (`void *`) 是一个指向LZ4压缩算法上下文的指针，包含了压缩算法所需的额外信息。

### 函数内部

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

### 函数返回值

- 返回值是

  ```
  int
  ```

  类型，通常表示压缩操作的结果：

  - 如果压缩成功，通常会返回0。
  - 如果发生错误，通常会返回一个负数，表示错误码。

### 注意事项

- 确保`dlen`指向的缓冲区足够大，以容纳压缩后的数据。在最坏的情况下，压缩后的数据可能会比原始数据更大。
- `LZ4_compress_default`函数是LZ4库提供的函数，用于执行压缩操作。确保已经正确链接了LZ4库，并且在编译时包含了必要的头文件。

### LZ4库的压缩函数

`LZ4_compress_default`函数是LZ4库提供的默认压缩函数，它使用LZ4算法来压缩数据。在LZ4库中，还有其他的压缩函数可供选择，如`LZ4_compress_HC`，它提供了更高的压缩率但可能需要更长的时间来压缩数据。

这段代码提供了LZ4压缩算法的实现逻辑，使用了LZ4库中的函数来执行压缩操作。如果你想要进一步了解LZ4库的内部实现细节，可以参考LZ4库的文档或源代码。



<u>LZ4_compress_default</u>:

```
int LZ4_compress_default(const char *source, char *dest, int inputSize,
	int maxOutputSize, void *wrkmem)
{
	return LZ4_compress_fast(source, dest, inputSize,
		maxOutputSize, LZ4_ACCELERATION_DEFAULT, wrkmem);
}


```







# 下一步

### 调研lzo压缩算法上下文

```c
static int lzo_compress(struct crypto_tfm *tfm, const u8 *src,
			unsigned int slen, u8 *dst, unsigned int *dlen)
{
	struct lzo_ctx *ctx = crypto_tfm_ctx(tfm);

	return __lzo_compress(src, slen, dst, dlen, ctx->lzo_comp_mem);
}
```

struct crypto_tfm *tfm：

```c
struct crypto_tfm {
	refcount_t refcnt;

	u32 crt_flags;

	int node;
	
	void (*exit)(struct crypto_tfm *tfm);
	
	struct crypto_alg *__crt_alg;

	void *__crt_ctx[] CRYPTO_MINALIGN_ATTR;
};
```

### 结构体成员解释

1. `refcnt`: (`refcount_t`) 是一个引用计数器，用于跟踪有多少引用指向这个`crypto_tfm`结构体实例。当引用计数变为0时，表示可以安全地释放与之关联的资源。
2. `crt_flags`: (`u32`) 是一个无符号整数，用于存储与算法相关的标志。这些标志可以表示算法的属性或状态。
3. `node`: (`int`) 可能用于标识算法上下文在内核中的位置或编号。
4. `exit`: (`void (*)(struct crypto_tfm *))` 是一个函数指针，指向一个清理函数。当不再需要该`crypto_tfm`实例时，这个函数会被调用来释放资源。
5. `__crt_alg`: (`struct crypto_alg *`) 是一个指向`crypto_alg`结构体的指针，该结构体包含了算法的描述信息。
6. `__crt_ctx[]`: (`void []`) 是一个灵活数组成员（Flexible Array Member, FAM），用于存储算法上下文所需的数据。`CRYPTO_MINALIGN_ATTR`是一个属性，确保数组元素的对齐满足最低要求。



### 如何选择不同的压缩算法

### 以及从上到下的调用还是未明晰

### 怎么从swap到zram

### 整一个压缩流程的接口都需要搞清楚

#### 	每个函数调用的什么东西，需要哪些变量，这些变量存了什么东西，具体是什么，干了什么

### 数据流大小多少
