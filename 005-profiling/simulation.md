
最后一个分析的方式，不是真实运行程序收集数据，而是通过专门的工具 模拟 来分析会发生什么。

这种分析器有很多种类，取决于模拟计算的方向。在本文中，我们主要关注 缓存和分支预测，使用[Cachegrind](https://valgrind.org/docs/manual/cg-manual.html)，这是  [Valgrind](https://valgrind.org/) 面向分析的部分，Valgrind 是用于内存泄漏检测和内存调试的成熟工具。

## 使用Cachegrind分析


Cachegrind 本质上是检查二进制中的“有趣”指令—— 执行内存读/写 和 条件/直接跳转—— 然后将这些指令替换为 使用软件数据结构 模拟相关的硬件操作 的代码。 因此它不需要访问源代码，可以直接在已经编译的程序上运行。可以使用下面的命令执行：

```bash
valgrind --tool=cachegrind --branch-sim=yes ./run
#       also simulate branch prediction ^   ^ any command, not necessarily one process
```

它检测所有涉及的二进制文件，运行它们，并输出类似于 perf stat 的摘要：

```
I   refs:      483,664,426
I1  misses:          1,858
LLi misses:          1,788
I1  miss rate:        0.00%
LLi miss rate:        0.00%

D   refs:      115,204,359  (88,016,970 rd   + 27,187,389 wr)
D1  misses:      9,722,664  ( 9,656,463 rd   +     66,201 wr)
LLd misses:         72,587  (     8,496 rd   +     64,091 wr)
D1  miss rate:         8.4% (      11.0%     +        0.2%  )
LLd miss rate:         0.1% (       0.0%     +        0.2%  )

LL refs:         9,724,522  ( 9,658,321 rd   +     66,201 wr)
LL misses:          74,375  (    10,284 rd   +     64,091 wr)
LL miss rate:          0.0% (       0.0%     +        0.2%  )

Branches:       90,575,071  (88,569,738 cond +  2,005,333 ind)
Mispredicts:    19,922,564  (19,921,919 cond +        645 ind)
Mispred rate:         22.0% (      22.5%     +        0.0%   )
```


将上一章节的相同代码喂给 Cachegrind，Cachegrind 输出几乎和perf 相同的数字，只是 perf测量的 内存读取和 分支 稍多，因为分支预测：在硬件确实发生了，因此增加了硬件计数器，但被丢弃并且不会影响实际性能，因此在模拟中被忽略。

Cachegrind 仅对缓存的第一级（ `D1` 数据、指令`I1`）  和最后一级 （ `LL`，unified）进行建模，其特征是从系统中推断出来的。它不会以任何方式限制您，因为您也可以从命令行设置它们，例如，对 L2 缓存进行建模： `--LL=<size>,<associativity>,<line size>` .

目前位置，看起来它只减慢了我们的程序 ，而且并没有提供  `perf stat`  不能提供的信息。为了获取除了摘要信息之外的信息，我们可以检查一个带分析数据的特殊的文件，默认情况下它会将其转储到 同一目录中名为 `cachegrind.out.<pid>` 的文件。它是人类可读的，但应通过`cg_annotate`命令  读取：

```bash
cg_annotate cachegrind.out.4159404 --show=Dr,D1mr,DLmr,Bc,Bcm
#                                    ^ we are only interested in data reads and branches
```

首先它显示了运行期间 使用的参数，包括缓存系统的特征：

```
I1 cache:         32768 B, 64 B, 8-way associative
D1 cache:         32768 B, 64 B, 8-way associative
LL cache:         8388608 B, 64 B, direct-mapped
```


它没有完全正确获得 L3 缓存：它不是统一的（总共 8M，但单个内核只能看到 4M）和 16 路关联，但我们暂时忽略它。

接下来，它输出类似于 `perf report` 的函数摘要 ：

```
Dr         D1mr      DLmr Bc         Bcm         file:function
--------------------------------------------------------------------------------
19,951,476 8,985,458    3 41,902,938 11,005,530  ???:query()
24,832,125   585,982   65 24,712,356  7,689,480  ???:void std::__introsort_loop<...>
16,000,000        60    3  9,935,484    129,044  ???:random_r
18,000,000         2    1  6,000,000          1  ???:random
 4,690,248    61,999   17  5,690,241  1,081,230  ???:setup()
 2,000,000         0    0          0          0  ???:rand
```


可以看到在排序阶段有很多分支错误预测，在二分搜索过程中也有很多 L1 缓存未命中和分支错误预测。我们无法用 perf 获取这些信息——它只会告诉整个程序的使用计数

Cachegrind的另一个重要功能是源代码的逐行关联。为此，需要编译程序时带上调试信息（ `-g` ），并明确告知 `cg_annotate` 要关联哪些源文件，或者只是传递 `--auto=yes` 该选项，以便告诉它关联可以访问的所有内容（包括标准库源代码）。

因此 整个 源代码到分析的过程是：

```bash
g++ -O3 -g sort-and-search.cc -o run
valgrind --tool=cachegrind --branch-sim=yes --cachegrind-out-file=cachegrind.out ./run
cg_annotate cachegrind.out --auto=yes --show=Dr,D1mr,DLmr,Bc,Bcm
```

由于 glibc 实现不易读的，出于说明目的，我们用  我们自己的二进制搜索替换`lower_bound`，它将像这样注释：

```c++
Dr         D1mr      DLmr Bc         Bcm       
         .         .    .          .         .  int binary_search(int x) {
         0         0    0          0         0      int l = 0, r = n - 1;
         0         0    0 20,951,468 1,031,609      while (l < r) {
         0         0    0          0         0          int m = (l + r) / 2;
19,951,468 8,991,917   63 19,951,468 9,973,904          if (a[m] >= x)
         .         .    .          .         .              r = m;
         .         .    .          .         .          else
         0         0    0          0         0              l = m + 1;
         .         .    .          .         .      }
         .         .    .          .         .      return l;
         .         .    .          .         .  }
```


不幸的是，Cachegrind只跟踪内存访问和分支。当瓶颈是由其他原因引起的时，我们需要其他模拟工具。