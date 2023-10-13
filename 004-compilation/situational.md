
`-O2` and `-O3` 中启用的大部分优化 保证可以提升、或者至少不损害性能。 `-O3`  中不包含的选项 可能是不严格满足编译标准，或者是高度情景化的，需要程序员的额外输入来保证使用是有益的。

让我们讨论一下最常用的那些。
## Loop Unrolling 循环展开

循环展开默认不启用，除非循环迭代次数是一个小的编译期常数—— 这种情况下会被完全替换为一个完全无跳转 的重复序列。可以使用`-funroll-loops`  标志全局启用，该标志将展开所有循环，如果其迭代次数可以在编译时或进入循环时确定。

你也可以使用 pragma 标记一个指定的循环：

```c++
#pragma GCC unroll 4
for (int i = 0; i < n; i++) {
    // ...
}
```


循环展开会使二进制变大，而且可能并不会使允许变快，所以不要狂热的使用它。

## Function Inlining

inline 最好让编译器来决定，但你可以使用 `inline` 关键词来影响它

```c++
inline int square(int x) {
    return x * x;
}
```


这个提示可能被编译器忽略，如果它认为潜在的性能收益不知道。你可以使用`always_inline` 属性来强制使用内联

```c++
#define FORCE_INLINE inline __attribute__((always_inline))
```

也有一个`-finline-limit=n`选项 可以让你指定内联函数大小（指令数量）的门槛。Clang等价选项`-inline-threshold`.

## Likeliness of Branches 分支可能性


分支（if switch）可能性可以使用`[[likely]]` and `[[unlikely]]` 属性进行提示：

```c++
int factorial(int n) {
    if (n > 1) [[likely]]
        return n * factorial(n - 1);
    else [[unlikely]]
        return 1;
}
```

这是C++20中的新特性。在这之前，有编译器特定的intrinsics包装条件表达式。GCC例子：

```c++
int factorial(int n) {
    if (__builtin_expect(n > 1, 1))
        return n * factorial(n - 1);
    else
        return 1;
}
```


当你需要告诉编译器正确的方向时，有很多像这样的其他例子，我们将在稍后相关的时候介绍

## Profile-Guided Optimization

添加这些元信息到源代码中是无聊的，人们已经厌烦写C++代码.

而且这样的优化是否有益往往不是很明显。为了决定是否执行分支重排、循环展开，我们需要回答这些问题：

- 分支使用的频率?
- 函数调用的频率？
- 循环迭代次数的平均值?


幸运的是，有方式自动提供真实世界信息

*Profile-guided optimization* (PGOe) 是使用 profile数据改善性能的技术，而不仅仅是静态分析所能实现的。简单来说，它将计时器和计数器添加到程序中的感兴趣的点，在真实数据上编译和运行它，然后再次编译它，但这次额外提供了来自测试运行的额外信息。


整个流程对现代编译器是自动化的。例如：`-fprofile-generate` 让 GCC使用分析代码来检查程序

```
g++ -fprofile-generate [other flags] source.cc -o binary
```


在我们运行程序之后——最好是在尽可能代表真实用例的输入上——它将创建一堆包含测试运行的日志数据的`*.gcda`文件，之后我们可以重建程序，并添加标志 `-fprofile-use` ：

```
g++ -fprofile-use [other flags] source.cc -o binary
```

对于大型代码库，它通常会将性能提高 10-20%，因此，它通常用在性能关键型项目的构建过程中。这是投资可靠的基准测试代码的更多理由。

