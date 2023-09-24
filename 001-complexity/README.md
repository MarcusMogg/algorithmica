
许多计算机科学的书籍都会在一开始引入**算法复杂度** 的概念，简单来说，就是计算过程中所有基本操作（加、减、乘、除……）的总和，有的时候也会按操作的成本进行加权。

算法复杂度是一个古老的概念，在1960年代早期系统制定，在那之后广泛应用于算法设计的代价函数。这个模型被广泛采用的原因是它很好的近似了当时计算机的工作方式。
# 经典复杂度理论

CPU的基本操作被称为**指令（instructions）**，它们的成本被称为**延迟（latencies）**。 指令被存储在内存中，并被处理器一条一条的处理，处理器有一系列**寄存器（register）** 存储内部状态。其中一个存储器被称为指令指针（instruction pointer，IP），用于指示下一条需要读取和执行的指令。每条指令会改变处理器的状态，可能会修改主存，，并且需要不同的CPU周期来完成，然后才能启动下一条指令。


为了估计程序的真实执行时间，你需要计算它所有执行指令的延迟的总和，并除以 时钟频率，即一个特定型号CPU每秒执行的周期数。

![](img/cpu.png)


时钟频率是一个不稳定且通常未知的变量，取决于 CPU 型号、操作系统设置、当前芯片温度、其他组件的功耗以及许多其他因素。相比之下，指令延时是固定的，用时钟周期表示，在不同的CPU之间甚至有些一致，因此对它们进行计数对于分析目的更有用。

例如，按定义矩阵乘法算法需要  $n^2 \cdot (n + n - 1)$ 次算术运算：具体来说，$n^3$ 次乘法和  $n^2 \cdot (n - 1)$ 次加法。如果我们查找这些指令的延迟（在称为指令表的特殊文档中，例如 [this one](https://www.agner.org/optimize/instruction_tables.pdf)），我们可以发现，乘法需要 3 个周期，而加法需要 1 个周期，因此整个计算需要$3 \cdot n^3 + n^2 \cdot (n - 1) = 4 \cdot n^3 - n^2$ 时钟周期。

类似于 将指令延迟的总和用作总执行时间，计算复杂度可用于量化抽象算法的内在时间要求，而无需依赖特定计算机的选择。

# 渐进复杂度

The idea to express execution time as a function of input size seems obvious now, but it wasn't so in the 1960s. Back then, [typical computers](https://en.wikipedia.org/wiki/CDC_1604) cost millions of dollars, were so large that they required a separate room, and had clock rates measured in kilohertz. They were used for practical tasks at hand, like predicting the weather, sending rockets into space, or figuring out how far a Soviet nuclear missile can fly from the coast of Cuba — all of which are finite-length problems. Engineers of that era were mainly concerned with how to multiply $3 \times 3$ matrices rather than $n \times n$ ones.

将执行时间

What caused the shift was the acquired confidence among computer scientists that computers will continue to become faster — and indeed they have. Over time, people stopped counting execution time, then stopped counting cycles, and then even stopped counting operations exactly, replacing it with an *estimate* that, on sufficiently large inputs, is only off by no more than a constant factor. With *asymptotic complexity*, verbose "$4 \cdot n^3 - n^2$ operations" turns into plain "$\Theta(n^3)$," hiding the initial costs of individual operations in the "Big O," along with all the other intricacies of the hardware.

![](img/complexity.jpg)

The reason we use asymptotic complexity is that it provides simplicity while still being just precise enough to yield useful results about relative algorithm performance on large datasets. Under the promise that computers will eventually become fast enough to handle any *sufficiently large* input in a reasonable amount of time, asymptotically faster algorithms will always be faster in real-time too, regardless of the hidden constant.

But this promise turned out to be not true — at least not in terms of clock speeds and instruction latencies — and in this chapter, we will try to explain why and how to deal with it.
