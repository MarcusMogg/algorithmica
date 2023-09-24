
如果你正在阅读本书，那么在你的计算机科学之旅中，存在某个时刻，你第一次开始关心代码的效率。

我是在高中时，意识到制作网站和做有用的编程不会让你进入大学，并进入了令人兴奋的算法竞赛世界。我是一个不错的程序员，特别是对于一个高中生来说，但我从来没有真正想知道我的代码需要多少时间才能执行。但突然之间，它开始变得重要：现在每个问题都有严格的时间限制。我开始计算我的操作。一秒钟能做多少？

我对计算机体系结构了解不多，无法回答这个问题。但我也不需要一个正确的答案——只需要一个经验法则。我的想法是：“2-3GHz 意味着每秒执行 2-30 亿条指令，在对数组元素执行某些操作的简单循环中，我还需要增加循环计数器，检查循环结束条件，做数组索引之类的东西，所以让我们为每个有用的指令增加 3-5 条指令的空间”，最后使用 $5 \cdot 10^8$ 作为估计。这些陈述都不是真的，但是计算我的算法需要多少操作并将其除以这个数字对于我的用例来说是一个很好的经验法则。



当然，真实的答案要复杂的多，并且高度取决于你想要什么操作。像  *pointer chasing* 可以慢到 s $10^7$  ，而 SIMD 加速过的线性代数可以快到   $10^{11}$ 。为了证明这些显着的差异，我们将使用以不同语言实现的矩阵乘法的案例研究，并更深入地研究计算机如何执行它们。

## 语言类型


在最低级别，计算机执行二进制编码指令的*机器码*，用于控制 CPU。它们非常难以使用，所以人们创造了计算机之后的第一件是就是创造 *编程语言*。它抽象出计算机操作的一些细节，以简化编程过程。

编程语言本质上只是一个接口，编写的任何程序总要在某个时候转换为 CPU执行的机器码，有多种不同的方式来做到这一点：


- 从程序员视角，有两种类型的语言：编译型，在执行之前预处理； 解释型，运行时转换
- 从计算机视角，也有两种类型：native, 直接执行机器码；托管，依赖于某种运行时来执行


由于在解释器中运行机器代码没有意义，因此总共有三种类型的语言：:

- 解释型语言 Python, JavaScript, or Ruby.
- 带运行时的编译型语言,  Java, C#
- 编译为native的语言 C, C++， Rust.

执行计算机程序没有“正确”的方法：每种方法都有自己的优点和缺点。解释器和虚拟机提供了灵活性，并支持一些不错的高级编程功能，例如动态类型、运行时代码更改和自动内存管理，但这些都带来了一些不可避免的性能权衡，我们现在将讨论这些功能。

## 解释型语言

一个用python写的 $1024 \times 1024$ 矩阵乘法:

```python
import time
import random

n = 1024

a = [[random.random()
      for row in range(n)]
      for col in range(n)]

b = [[random.random()
      for row in range(n)]
      for col in range(n)]

c = [[0
      for row in range(n)]
      for col in range(n)]

start = time.time()

for i in range(n):
    for j in range(n):
        for k in range(n):
            c[i][j] += a[i][k] * b[k][j]

duration = time.time() - start
print(duration)
```

这份代码需要运行 630 s.  超过10min了！

让我们试着正确看待这个数字。运行它的CPU的时钟频率为1.4GHz，这意味着它每秒执行 $1.4 \cdot 10^9$  周期， 整个计算大概 $10^{15}$ 次操作，也就是说最里面的循环中每次乘法大约880个周期。

如果你知道 Python做了哪些事来理解程序员的意图，你就不会奇怪：
- 解析表达式 `c[i][j] += a[i][k] * b[k][j]`;
- 尝试找出 `a`, `b`, and `c` ，并在带有类型信息的特殊哈希表中查找它们的名称;
- 理解`a` 是一个数组, 获取 `[]` 运算符, 检索`a[i]`, 理解他也是一个数组,再次获取 `[]` 运算符, 获取 `a[i][k]`的指针, 然后再获取你本身
- 查找其类型 `float`, 然后获取 `*` 运算符;
- 在 `b`  `c` 上执行相同操作，最后把结果写到 `c[i][j]`.

当然，广泛使用的语言（如Python）的解释器得到了很好的优化，他们可以在重复执行相同的代码时跳过其中一些步骤。但是，由于语言设计，一些相当大的开销仍然是不可避免的。如果我们摆脱所有这些类型的检查和pointer chasing，也许我们可以得到接近 1 的乘法比率的周期数，或者本机乘法的“成本”

## 托管语言


相同的矩阵乘法过程，但是使用java

```java
import java.util.Random;

public class Matmul {
    static int n = 1024;
    static double[][] a = new double[n][n];
    static double[][] b = new double[n][n];
    static double[][] c = new double[n][n];

    public static void main(String[] args) {
        Random rand = new Random();

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                a[i][j] = rand.nextDouble();
                b[i][j] = rand.nextDouble();
                c[i][j] = 0;
            }
        }

        long start = System.nanoTime();

        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                for (int k = 0; k < n; k++)
                    c[i][j] += a[i][k] * b[k][j];
                
        double diff = (System.nanoTime() - start) * 1e-9;
        System.out.println(diff);
    }
}
```


现在只需要执行10s，每个乘法大概只需要13个CPU 周期，比Python快63倍。

Java是一种 编译的，但不是native语言。程序首先编译为字节码，然后由虚拟机（JVM）解释。为了实现更高的性能，经常执行的代码部分（如最 里面的循环）在运行时编译到机器代码中，然后在几乎没有额外开销的情况下执行。此技术称为实时编译（jit）。

JIT compilation is not a feature of the language itself, but of its implementation. There is also a JIT-compiled version of Python called [PyPy](https://www.pypy.org/), which needs about 12 seconds to execute the code above without any changes to it.


### Compiled Languages

Now it's turn for C:

```cpp
#include <stdlib.h>
#include <stdio.h>
#include <time.h>

#define n 1024
double a[n][n], b[n][n], c[n][n];

int main() {
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            a[i][j] = (double) rand() / RAND_MAX;
            b[i][j] = (double) rand() / RAND_MAX;
        }
    }

    clock_t start = clock();

    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            for (int k = 0; k < n; k++)
                c[i][j] += a[i][k] * b[k][j];

    float seconds = (float) (clock() - start) / CLOCKS_PER_SEC;
    printf("%.4f\n", seconds);
    
    return 0;
}
```

It takes 9 seconds when you compile it with `gcc -O3`.

It doesn't seem like a huge improvement — the 1-3 second advantage over Java and PyPy can be attributed to the additional time of JIT-compilation — but we haven't yet taken advantage of a far better C compiler ecosystem. If we add `-march=native` and `-ffast-math` flags, time suddenly goes down to 0.6 seconds!

What happened here is we [communicated to the compiler](/hpc/compilation/flags/) the exact model of the CPU we are running (`-march=native`) and gave it the freedom to rearrange [floating-point computations](/hpc/arithmetic/float) (`-ffast-math`), and so it took advantage of it and used [vectorization](/hpc/simd) to achieve this speedup.

It's not like it is impossible to tune the JIT-compilers of PyPy and Java to achieve the same performance without significant changes to the source code, but it is certainly easier for languages that compile directly to native code.

### BLAS

Finally, let's take a look at what an expert-optimized implementation is capable of. We will test a widely-used optimized linear algebra library called [OpenBLAS](https://www.openblas.net/). The easiest way to use it is to go back to Python and just call it from `numpy`:

```python
import time
import numpy as np

n = 1024

a = np.random.rand(n, n)
b = np.random.rand(n, n)

start = time.time()

c = np.dot(a, b)

duration = time.time() - start
print(duration)
```

Now it takes ~0.12 seconds: a ~5x speedup over the auto-vectorized C version and ~5250x speedup over our initial Python implementation!

You don't typically see such dramatic improvements. For now, we are not ready to tell you exactly how this is achieved. Implementations of dense matrix multiplication in OpenBLAS are typically [5000 lines of handwritten assembly](https://github.com/xianyi/OpenBLAS/blob/develop/kernel/x86_64/dgemm_kernel_16x2_haswell.S) tailored separately for *each* architecture. In later chapters, we will explain all the relevant techniques one by one, and then [return](/hpc/algorithms/matmul) to this example and develop our own BLAS-level implementation using just under 40 lines of C.

### Takeaway

The key lesson here is that using a native, low-level language doesn't necessarily give you performance; but it does give you *control* over performance.

Complementary to the "N operations per second" simplification, many programmers also have a misconception that using different programming languages has some sort of multiplier on that number. Thinking this way and [comparing languages](https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html) in terms of performance doesn't make much sense: programming languages are fundamentally just tools that take away *some* control over performance in exchange for convenient abstractions.  Regardless of the execution environment, it is still largely a programmer's job to use the opportunities that the hardware provides.
