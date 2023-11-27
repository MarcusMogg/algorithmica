
现代计算机内存是高度分层的，包含多个不同大小、速度的缓存层级，其中较高级别通常存储来自最常访问的低级别数据以减少延迟：下一个级别通常快一个数量级，但也更小和/或更昂贵。

![](../img/hierarchy.png)

抽象地说，各种存储设备可以描述为具有一定存储容量 M 的模块，并且可以以 块 B （不是单个字节！）的形式读取或写入数据，需要固定的时间完成。

从这个角度来看，每种类型的内存都有几个重要特征：

- *total size* 总大小 $M$;
- *block size* 块大小 $B$; 
- *latency*, 延迟，即获取一个字节所需的时间;;
- *bandwidth*, 带宽，可能高于块大小乘以延迟，这意味着 I/O 操作可以“重叠”"overlap";
- *cost* 摊销意义上的成本，包括芯片的价格、能源需求、维护等.

以下是 2021 年商用硬件的大致比较表：

| Type | $M$      | $B$ | Latency | Bandwidth | $/GB/mo[^pricing] |
|:-----|:---------|-----|---------|-----------|:------------------|
| L1   | 10K      | 64B | 2ns     | 80G/s     | -                 |
| L2   | 100K     | 64B | 5ns     | 40G/s     | -                 |
| L3   | 1M/core  | 64B | 20ns    | 20G/s     | -                 |
| RAM  | GBs      | 64B | 100ns   | 10G/s     | 1.5               |
| SSD  | TBs      | 4K  | 0.1ms   | 5G/s      | 0.17              |
| HDD  | TBs      | -   | 10ms    | 1G/s      | 0.04              |
| S3   | $\infty$ | -   | 150ms   | $\infty$  | 0.02[^S3]         |

实际上，每种类型的内存都有许多细节，我们现在将介绍这些细节。

## Volatile Memory 易失性内存


RAM级别的所有内容被成为 易失性内存，因为在电量不足或者其他灾难情况下不会保存数据。ta很快，因此用来在计算机通电的情况下存储临时数据。

由快到慢:

- **CPU registers**, CPU用于存储立即数的 零成本访问数据单元，也可以认为是一种内存. 有数量限制 (e.g.,只有 16个 "通用寄存器" ), 在某些情况下，处于性能原因，你可能希望使用所有的寄存器.
- **CPU caches.** 现代CPU通常有多级缓存 (L1, L2, often L3, and rarely even L4). 最低级别在核心之间共享，通常根据核心数量变化 (e.g., 10核 CPU应该有10M  L3 cache).
- **Random access memory,** 第一种可拓展的内存类型:如今你可以在公有云上租赁TB级别 RAM 机器.这是应该存储大多数工作数据的地方。

CPU 缓存系统有一个重要的概念叫做 缓存行*cache line*， 它是CPU和RAM之间的级别数据传输单元。 缓存行的大小通常是 64byte，这意味着所有主内存都被划分为 64 字节的块，每当请求（读取或写入）单个字节时，也会获取其所有 63 个缓存行邻居，无论你是否愿意。

CPU 级别的缓存会根据缓存行的上次访问时间自动进行。访问时，缓存行的内容将放置在最低的缓存层上，然后逐渐逐出，除非及时再次访问。程序员无法明确地控制这个过程，但值得详细研究它是如何工作的，我们将在下一章中介绍。
## 非易失性存储器

虽然 CPU 缓存中的数据单元和 RAM 中的数据单元只存储几个电子（这些电子会定期泄漏并需要定期刷新），但非易失性存储器类型的数据单元存储数百个电子。这使得数据可以在没有电源的情况下长时间保留，但代价是性能和耐用性——因为当你有更多的电子时，你也有更多的机会让它们与硅原子碰撞。


有很多方法可以以持久的方式存储数据，但从程序员的角度来看，这些是主要的：

- **Solid state drives.** 固态硬盘 These have relatively low latency on the order of 0.1ms ($10^5$ ns), but they also have a high cost, amplified by the fact that they have limited lifespans as each cell can only be written to a limited number of times. This is what mobile devices and most laptops use because they are compact and have no moving parts.
- **Hard disk drives** 硬盘驱动器 are unusual because they are actually [rotating physical disks](https://www.youtube.com/watch?v=3owqvmMf6No&feature=emb_title) with a read/write head attached to them. To read a memory location, you need to wait until the disk rotates to the right position and then very precisely move the head to it. This results in some very weird access patterns where reading one byte randomly may take the same time as reading the next 1MB of data — which is usually on the order of milliseconds. Since this is the only part of a computer, except for the cooling system, that has mechanically moving parts, hard disks break quite often (with the average lifespan of ~3 years for a data center HDD).
- **Network-attached storage**,网络连接存储 which is the practice of using other networked devices to store data on them. There are two distinctive types. The first one is the Network File System (NFS), which is a protocol for mounting the file system of another computer over the network. The other is API-based distributed storage systems, most famously [Amazon S3](https://aws.amazon.com/s3/), that are backed by a fleet of storage-optimized machines of a public cloud, typically using cheap HDDs or some [more exotic](https://aws.amazon.com/storagegateway/vtl/) storage types internally. While NFS can sometimes work even faster than HDD if it is located in the same data center, object storage in the public cloud usually has latencies of 50-100ms. They are typically highly distributed and replicated for better availability.

由于 SDD/HDD 明显比 RAM 慢，因此在此级别或以下的所有内容通常称为外部存储器。

与 CPU 缓存不同，外部存储器可以显式控制。这在许多情况下都很有用，但大多数程序员只想从中抽象出来，并将其用作主内存的扩展，而操作系统可以通过虚拟内存来做到这一点。