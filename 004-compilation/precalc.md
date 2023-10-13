
> 不如看这篇文章 https://zhuanlan.zhihu.com/p/350429113



当编译器识别到一个变量不依赖任何用户提供的数据，它们可以在编译时对其进行计算，然后转换为编码到机器码中的常量值。


这个优化对性能有很大帮助，但它不是C++标准的一部分（c++20之后 consteval），所以编译器不一定非要这样做。当编译时计算实在难以实现或者非常耗时时，编译器可能会跳过它。

## 常量表达式

For a more reliable solution, in modern C++ you can mark a function as `constexpr`; if it is called by passing constants its value is guaranteed to be computed during compile time:


```c++
constexpr int fibonacci(int n) {
    if (n <= 2)
        return 1;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

static_assert(fibonacci(10) == 55);
```

These functions have some restrictions like that they only call other `constexpr` functions and can't do memory allocation, but otherwise, they are executed "as is."

Note that while `constexpr` functions don't cost anything during run time, they still increase compilation time, so at least remotely care about their efficiency and don't put something NP-complete in them:

```c++
constexpr int fibonacci(int n) {
    int a = 1, b = 1;
    while (n--) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

There used to be many more limitations in earlier C++ standards, like you could not use any sort of state inside them and had to rely on recursion, so the whole process felt more like Haskell programming rather than C++. Since C++17, you can even compute static arrays using the imperative style, which is useful for precomputing lookup tables:

```c++
struct Precalc {
    int isqrt[1000];

    constexpr Precalc() : isqrt{} {
        for (int i = 0; i < 1000; i++)
            isqrt[i] = int(sqrt(i));
    }
};

constexpr Precalc P;

static_assert(P.isqrt[42] == 6);
```

Note that when you call `constexpr` functions while passing non-constants, the compiler may or may not compute them during compile time:

```c++
for (int i = 0; i < 100; i++)
    cout << fibonacci(i) << endl;
```

In this example, even though technically we perform a constant number of iterations and call `fibonacci` with parameters known at compile time, they are technically not compile-time constants. It's up to the compiler whether to optimize this loop or not — and for heavy computations, it often chooses not to.

