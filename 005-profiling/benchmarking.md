
大多数好的软件工程实践都以某种方式解决了使开发周期更快的问题：您希望更快地编译软件（构建系统），尽快捕获错误（静态分析，持续集成），在新版本准备就绪后立即发布（持续部署），毫不拖延地对用户反馈做出反应（敏捷开发）。

性能工程也不例外。如果你做得正确，它也应该类似于一个循环：

1. 执行程序并收集指标.
2. 找到瓶颈.
3. 移除瓶颈然后回到步骤1.

在本节中，我们将讨论基准测试，并讨论一些实用技术，这个循环更短，并帮助你更快地迭代。大部分建议来自本书的工作，因此您可以在本书的[代码存储库](https://github.com/sslotin/ahm-code)中找到许多描述设置的真实示例。

## C++ 中的基准测试

编写基准测试的方法有很多。也许最流行的方法是在一个文件中包含要比较的几个相同语言的实现，在 `main` 函数中单独调用它们，并在同一源文件中计算所需的所有指标。

这种方法的缺点是你需要编写大量的样板代码，并为每个实现复制它，但可以通过元编程部分减少。例如，当您对多个 gcd 实现进行基准测试时，您可以使用以下高阶函数大大减少基准测试代码：

```c++
const int N = 1e6, T = 1e9 / N;
int a[N], b[N];

void timeit(int (*f)(int, int)) {
    clock_t start = clock();

    int checksum = 0;

    for (int t = 0; t < T; t++)
        for (int i = 0; i < n; i++)
            checksum ^= f(a[i], b[i]);
    
    float seconds = float(clock() - start) / CLOCKS_PER_SEC;

    printf("checksum: %d\n", checksum);
    printf("%.2f ns per call\n", 1e9 * seconds / N / T);
}

int main() {
    for (int i = 0; i < N; i++)
        a[i] = rand(), b[i] = rand();
    
    timeit(std::gcd);
    timeit(my_gcd);
    timeit(my_another_gcd);
    // ...

    return 0;
}
```


这是一种开销非常低的方法，可让您运行更多实验并从中获得准确的结果。您仍然需要执行一些重复的操作，但它们可以通过框架在很大程度上实现自动化，[Google benchmark library](https://github.com/google/benchmark) 是C++最受欢迎的选择。一些编程语言也有方便的内置工具进行基准测试：这里特别提到 [Python's timeit function](https://docs.python.org/3/library/timeit.html)和[Julia's @benchmark macro](https://github.com/JuliaCI/BenchmarkTools.jl).。

尽管在执行速度方面效率很高，但 C 和 C++ 并不是最高效的语言，尤其是在分析方面。当您的算法依赖于某些参数（例如输入大小）并且您需要从每个实现中收集多个数据点时，您确实希望将基准测试代码与外部环境集成, 并使用其他程序分析结果。

## Splitting Up Implementations

提高模块化和可重用性的一种方法是 将所有测试和分析代码与算法的实际实现分开，并使不同的文件中实现不同的版本，但具有相同的接口。

In C/C++, you can do this by creating a single header file (e.g., `gcd.hh`) with a function interface and all its benchmarking code in `main`:

在 C/C++ 中，您可以通过创建一个带有函数接口的单个头文件（例如， `gcd.hh` ），`main`里面有所有的测试代码 来做到这一点：

```c++
int gcd(int a, int b); // to be implemented

// for data structures, you also need to create a setup function
// (unless the same preprocessing step for all versions would suffice)

int main() {
    const int N = 1e6, T = 1e9 / N;
    int a[N], b[N];
    // careful: local arrays are allocated on the stack and may cause stack overflow
    // for large arrays, allocate with "new" or create a global array

    for (int i = 0; i < N; i++)
        a[i] = rand(), b[i] = rand();

    int checksum = 0;

    clock_t start = clock();

    for (int t = 0; t < T; t++)
        for (int i = 0; i < n; i++)
            checksum += gcd(a[i], b[i]);
    
    float seconds = float(clock() - start) / CLOCKS_PER_SEC;

    printf("%d\n", checksum);
    printf("%.2f ns per call\n", 1e9 * seconds / N / T);
    
    return 0;
}
```


然后，为每个算法版本创建许多实现文件（例如：`v2.cc` ，`v1.cc` 或一些有意义的名称），这些文件都包含该同一个头文件

```c++
#include "gcd.hh"

int gcd(int a, int b) {
    if (b == 0)
        return a;
    else
        return gcd(b, a % b);
}
```

这样做的全部目的是能够从命令行对特定算法版本进行基准测试，而无需接触任何源代码文件。为此，您可能还希望公开它可能具有的任何参数，例如，通过从命令行参数解析它们：

```c++
int main(int argc, char* argv[]) {
    int N = (argc > 1 ? atoi(argv[1]) : 1e6);
    const int T = 1e9 / N;

    // ...
}
```

另一种方法是使用 C 风格的全局定义，然后在编译期间将它们与 `-D N=...` 标志一起传递：

```c++
#ifndef N
#define N 1000000
#endif

const int T = 1e9 / N;
```

通过这种方式，您可以使用编译时常量，这可能对某些算法的性能非常有益，但代价是每次要更改参数时都必须重新构建程序，这大大增加了跨一系列参数值收集指标所需的时间。

## Makefile


通过拆分源文件，可以使用缓存生成系统（如 Make）加快编译速度。

我通常在我的项目中携带这个 Makefile 的一个版本：

```c++
compile = g++ -std=c++17 -O3 -march=native -Wall

%: %.cc gcd.hh
	$(compile) $< -o $@ 

%.s: %.cc gcd.hh
	$(compile) -S -fverbose-asm $< -o $@

%.run: %
	@./$<

.PHONY: %.run
```

现在，您可以使用`make example`进行编译，并使用`make example.run` 自动运行 。

您还可以在生成文件中添加用于计算统计信息的脚本，或将其与`perf stat` 调用合并 以使分析自动进行。
## Jupyter Notebooks


为了加快分析速度，您可以创建一个 Jupyter 笔记本，可以在其中放置所有脚本并执行所有绘图。

添加用于对实现进行基准测试的包装器很方便，它只返回标量结果：

```python
def bench(source, n=2**20):
    !make -s {source}
    if _exit_code != 0:
        raise Exception("Compilation failed")
    res = !./{source} {n} {q}
    duration = float(res[0].split()[0])
    return duration
```

然后，您可以使用它来编写干净的分析代码:

```python
ns = list(int(1.17**k) for k in range(30, 60))
baseline = [bench('std_lower_bound', n=n) for n in ns]
results = [bench('my_binary_search', n=n) for n in ns]

# plotting relative speedup for different array sizes
import matplotlib.pyplot as plt

plt.plot(ns, [x / y for x, y in zip(baseline, results)])
plt.show()
```

此工作流建立后将使您更快地迭代并专注于优化算法本身。
