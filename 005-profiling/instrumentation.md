

*Instrumentation* 是一个过于复杂的概念，意味着将一下计时器和其他的跟踪代码 插入到程序中。一个最简单的例子是Unix上使用 `time` 程序来测量整个程序的执行时间。

更通用的，我们希望知道程序的哪个部分需要优化。编译器和 IDE 附带了一些工具，可以自动对指定的函数进行计时，但使用语言提供的可以在任何函数手动执行计时会更鲁棒：

```cpp
clock_t start = clock();
do_something();
float seconds = float(clock() - start) / CLOCKS_PER_SEC;
printf("do_something() took %.4f", seconds);
```

注意一个细节，此方法不能测量执行速度非常快的函数，因为`clock` 函数返回当前时间使用的是 毫秒 ($10^{-6}$)，而且它本身就要花费几百ns来完成。所有其他与时间相关的程序同样具有至少微秒级的粒度，这在低级别优化的世界中是永恒的。

为了实现更高的准确性，你可以在循环中重复执行函数，计时器只执行整体的一次，然后 总时间除以 迭代次数。

```cpp
#include <stdio.h>
#include <time.h>

const int N = 1e6;

int main() {
    clock_t start = clock();

    for (int i = 0; i < N; i++)
        clock(); // benchmarking the clock function itself

    float duration = float(clock() - start) / CLOCKS_PER_SEC;
    printf("%.2fns per iteration\n", 1e9 * duration / N);

    return 0;
}
```

你也需要保证没有被缓存、被编译器优化或者被其他类似的影响。本章节最后会有一个独立的、非常复杂的章节我们详细讨论这个话题。
## 事件采样

Instrumentation 也可以收集其他类型的信息，这些信息可以提供有关特定算法性能的有用见解。例如：

- 对于一个hash函数，我们对其输入的平均长度感兴趣。
- 对于一个二叉树，我们关注它的大小和高度
- 对于一个排序算法，我们想要知道它执行了多少次比较

类似的，我们可以在代码里插入计数器来计算这些算法相关的统计。

添加计数器的缺点是会引入额外开销，不过你可以通过仅对 随机一小部分调用执行操作开缓解它。

```c++
void query() {
    if (rand() % 100 == 0) {
        // update statistics
    }
    // main logic
}
```

如果采样率足够小，每次调用的剩余开销将是随机数生成和条件检查。有趣的是，我们可以通过一些统计魔法对其进行更多优化。


从数学上讲，我们在这里做的是从[伯努利分布](https://en.wikipedia.org/wiki/Bernoulli_distribution)(p 是采样率)中重复采样,知道成功。还有另一种分布告诉我们在第一个正值之前需要多少次伯努利采样迭代，称为[几何分布](https://en.wikipedia.org/wiki/Geometric_distribution)。我们可以做的是从中采样，并将该值用作递减计数器：

```c++
void query() {
    static next_sample = geometric_distribution(sample_rate);
    if (next_sample--) {
        next_sample = geometric_distribution(sample_rate);
        // ...
    }
    // ...
}
```


这样，我们可以消除在每次调用时对新随机数进行采样的需要，只需在计算统计数据时重置计数器

大型项目中的库算法开发人员经常使用此类技术来收集分析数据，且不会对最终程序的性能产生太大影响。