
有两种库实现算法的情况并不少见，每种实现都维护自己的基准测试代码，并且每种都声称比另一种更快。这让所有相关人员感到困惑，尤其是用户，他们必须以某种方式在两者之间进行选择。

此类情况通常不是由其作者的欺诈行为引起的; 他们只是对“更快”的含义有不同的定义。事实上，只定义和使用一个性能指标通常是非常有问题的。

## 测量正确的事

有很多事情会在基准测试中引入偏差。


**不同的数据集**。有许多算法的性能在某种程度上取决于数据集分布。例如，为了定义最快的排序、最短路径或二进制搜索算法，您必须指定运行算法的数据集。

这有时甚至适用于处理单个输入的算法。例如，向 GCD 实现提供顺序数字不是一个好主意，因为它使分支非常可预测：

```c++
// don't do this
int checksum = 0;

for (int a = 0; a < 1000; a++)
    for (int b = 0; b < 1000; b++)
        checksum ^= gcd(a, b);
```

但是，如果我们随机抽样这些相同的数字，分支预测会变得更加困难，并且基准测试需要更长的时间。尽管处理相同的输入，但顺序会改变:

```c++
int a[1000], b[1000];

for (int i = 0; i < 1000; i++)
    a[i] = rand() % 1000, b[i] = rand() % 1000;

int checksum = 0;

for (int t = 0; t < 1000; t++)
    for (int i = 0; i < 1000; i++)
        checksum += gcd(a[i], b[i]);
```


尽管在大多数情况下，最合乎逻辑的选择是随机均匀地对数据进行采样，但许多实际应用程序的分布远非均匀，因此不能只选择一个分布。一般来说，一个好的基准测试应该是特定于应用程序的，并使用尽可能能代表您的真实用例的数据集。

**多个目标.** 一些算法设计问题具有多个关键目标。比如哈希表，除了高度依赖键的分布外，还需要仔细平衡：

- 内存使用,
- 添加元素的延迟,
- latency of positive membership query,  在/不在 表中的元素查询延迟？
- latency of negative membership query.

在哈希表实现之间进行选择的唯一方法是尝试将多个变体放入应用程序中。

**延迟与吞吐量.** 人们经常忽略的另一个方面是，执行时间可以用多种方式定义，即使是单个查询也是如此。

当你编写这样的代码时：

```c++
for (int i = 0; i < N; i++)
    q[i] = rand();

int checksum = 0;

for (int i = 0; i < N; i++)
    checksum ^= lower_bound(q[i]);
```


然后对整个程序进行计时并将其除以迭代次数，您实际上是在测量查询的吞吐量 - 每单位时间可以处理多少操作。由于交叉存取，这通常少于单独处理一个操作的实际时间。

要测量真实的*latency*延迟，需要在调用之间引入依赖关系：

```c++
for (int i = 0; i < N; i++)
    checksum ^= lower_bound(checksum ^ q[i]);
```

通常可能存在管道停滞问题的算法中会产生最大的差异，例如，在比较分支和无分支算法时。

**冷缓存.** 另一个偏差来源是冷缓存效应，内存初始读取需要更长的时间，因为所需的数据尚未在缓存中。

这可以通过在开始测量之前进行预热运行来解决：

```c++
// warm-up run

volatile checksum = 0;

for (int i = 0; i < N; i++)
    checksum ^= lower_bound(q[i]);


// actual run

clock_t start = clock();
checksum = 0;

for (int i = 0; i < N; i++)
    checksum ^= lower_bound(q[i]);
```


**过度优化.** 。有时基准测试是完全错误的，因为编译器只优化了基准测试的代码。为了防止编译器偷工减料，您需要添加校验和并将它们打印在某处或添加 `volatile` 限定符，这也可以防止循环迭代的任何类型的交叉存取。

对于仅写入数据的算法，可以使用 `__sync_synchronize()` 内部函数添加内存围栏并防止编译器累积更新。

## 减少噪声


我们描述的问题在测量中产生偏差：它们始终使一种算法优于另一种算法。基准测试还存在其他类型的问题，这些问题会导致不可预测的偏差或完全随机的噪声，从而增加方差。


这些类型的问题是由副作用和某种外部噪声引起的，主要是由于嘈杂的邻居和 CPU 频率缩放：

- 如果对计算密集型算法进行基准测试，使用`perf stat`以周期为单位测量其性能： 这样它将与时钟频率无关，时钟频率的波动通常是噪声的主要来源。
- 否则，请将核心频率设置为您期望的频率，并确保没有任何干扰。在Linux上，你可以使用 `cpupower` (e.g., `sudo cpupower frequency-set -g powersave` 将其设为最低 `sudo cpupower frequency-set -g ondemand` 启用涡轮增压).我使用了一个方便的 [GNOME shell 扩展](https://extensions.gnome.org/extension/1082/cpufreq/)，它有一个单独的按钮来做到这一点。
- 如果适用，请关闭超线程并将作业附加到特定内核。确保系统上没有其他作业正在运行，关闭网络并尽量不要摆弄鼠标

您无法完全消除噪音和偏见。甚至程序的名称也会影响其速度：可执行文件的名称最终出现在环境变量中，环境变量最终出现在调用堆栈中，因此名称的长度会影响堆栈对齐，这可能导致数据访问速度减慢由于跨越缓存行或内存页边界。

在指导优化时，尤其是在向其他人报告结果时，考虑噪声非常重要。除非您期望获得 2 倍的改进，否则请像对待 A/B 测试一样对待所有微基准测试。

当您在笔记本电脑上运行程序不到一秒钟时，性能波动±5%是完全正常的。因此，如果要决定是还原还是保持潜在的 +1% 改进，请运行它直到达到统计显著性，您可以通过计算方差和 p 值来确定统计显著性。

## 拓展阅读

Interested readers can explore this comprehensive [list of experimental computer science resources](https://www.cs.huji.ac.il/w~feit/exp/related.html) by Dror Feitelson, perhaps starting with "[Producing Wrong Data Without Doing Anything Obviously Wrong](http://eecs.northwestern.edu/~robby/courses/322-2013-spring/mytkowicz-wrong-data.pdf)" by Todd Mytkowicz et al.

You can also watch [this great talk](https://www.youtube.com/watch?v=r-TLSBdHe1A) by Emery Berger on how to do statistically sound performance evaluation.
