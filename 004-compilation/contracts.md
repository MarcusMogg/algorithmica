
在Java和rust 这种"安全"语言中，通常你的所有可能操作和所有可能输入 都有良好定义的行为有一些内容未明确定义，比如hash table key的顺序，vector 的增长因子，但这些通常是一些次要细节，留给实现，以便将来提高性能。

相比之下，C/C++ 将未定义行为的概念提升到另一个层级。某些操作不会造成 编译期或者运行时错误，只是不允许——从程序员和编译器契约的意义上说，在未定义行为的情况下，编译器在法律上被允许做任何事情，包括炸毁显示器或格式化硬盘驱动器。但是编译器工程师对这样做不感兴趣。相反，未定义的行为用于保证缺少极端情况并帮助优化。

## 为什么存在未定义行为

主要有两种操作会导致未定义行为：

- 几乎肯定是错误的操作，比如除0、解引用空指针、从未初始化内存读取。你希望在测试的时候尽快发现这些问题，所以崩溃或者具有某些非确定性行为比让它们始终执行固定的回退操作（例如返回零）要好。 您可以使用*sanitizers* 编译并运行程序，以便及早捕获未定义的行为。在 GCC 和 Clang 中，您可以使用该 `-fsanitize=undefined` 标志，一些因UB 导致的臭名昭著的操作将被检测 以防运行时出现。

- 不同平台上有不同行为的操作。比如，左移int超过31位是 ub，因为此操作在Arm和x86CPU上的实现不同。如果将一个定义为标准行为，那么在另一个平台编译的程序不得不花费一下周期来检测边界行为，因此最好不定义它。

  有时，有些特定平台行为存在合法用例，可以标记为实现定义*implementation-defined* 而不是未定义行为。例如，负数右移结果是平台相关的：最高位补0或者1（e.g.,`11010110 = -42` 右移 一位可以是`01101011 = 107` 或 `11101011 = -21`）

Designating something as undefined instead of implementation-defined behavior also helps compilers in optimization. Consider the case of signed integer overflow. On almost all architectures, [signed integers](/hpc/arithmetic/integer) overflow the same way as unsigned ones, with `INT_MAX + 1 == INT_MIN`, and yet, this is undefined behavior according to the C++ standard. This is very much intentional: if you disallow signed integer overflow, then `(x + 1) > x` is guaranteed to be always true for `int`, but not for `unsigned int`, because `(x + 1)` may overflow. For signed types, this lets compilers optimize such checks away.

As a more naturally occurring example, consider the case of a loop with an integer control variable. Modern C++ and languages like Rust encourage programmers to use an unsigned integer (`size_t` / `usize`), while C programmers stubbornly keep using `int`. To understand why, consider the following `for` loop:

```cpp
for (unsigned int i = 0; i < n; i++) {
    // ...
}
```

How many times does this loop execute? There are technically two valid answers: $n$ and infinity, the second being the case if $n$ exceeds $2^{32}$ so that $i$ keeps resetting to zero every $2^{32}$ iterations. While the former is probably the one assumed by the programmer, to comply with the language spec, the compiler still has to insert additional runtime checks and consider the two cases, which should be optimized differently. Meanwhile, the `int` version would make exactly $n$ iterations because the very possibility of a signed overflow is defined out of existence.

## 删除边界情况

“安全”编程风格通常设计许多运行时检查，但它们不必以牺牲性能为代价。

例如 C++ STL 里`vector` 和 `array` 有一个”不安全“的 `[]`操作符 和 一个 "安全" 的`.at()`方法，如下：

```cpp
T at(size_t k) {
    if (k >= size())
        throw std::out_of_range("Array index exceeds its size");
    return _memory[k];
}
```


有趣的是，这些检查在runtime很少真的执行，因为编译器通常而已在编译期 证明 每次访问在边界内。例如，当 `for`循环从 1 迭代到数组大小并在每一步索引 i -th 元素时，不会发生任何非法事件，因此可以安全地优化边界检查。

## Assumptions 假设

当编译器不能证明边界情况，但你可以时，你可以使用未定义行为机制提供附加信息。

Clang有一个有用的`__builtin_assume`  函数，你可以在里面放一个保证为true的语句，然后编译器可以使用这个 假设来进行优化。在GCC里，可以使用`__builtin_unreachable`:

```cpp
void assume(bool pred) {
    if (!pred)
        __builtin_unreachable();
}
```

对于这个例子，你可以把`assume(k < vector.size())` 放在`at`之前，这样边界检查 会被进行优化

结合 `assume` 、`assert` 和 `static_assert` 来查找错误也非常有用：您可以使用相同的函数来检查调试版本中的前提条件，然后使用它们来提高生产版本中的性能。

## Arithmetic 算术

您应该记住极端情况，尤其是在优化算术时。

对于浮点运算，这不太重要，因为您可以使用 `-ffast-math` 标志 禁用的严格标准合规性（也包含在 `-Ofast` 中）。您几乎都必须这样做，因为否则，编译器只能按照与源代码中相同的顺序执行算术运算，而无法进行任何优化。

对于整数算术，这是不同的，因为结果必须精确。考虑除以 2 的情况：:

```cpp
unsigned div_unsigned(unsigned x) {
    return x / 2;
}
```

一个广为周知的优化是使用右移(`x >> 1`):

```nasm
shr eax
```

这对正数来说总是正确的，但是一般情况呢？

```cpp
int div_signed(int x) {
    return x / 2;
}
```

如果x是负数，那么简单的移位不起作用——无论移位是以零还是符号位完成的：

- 如果使用0进行移位，那会得到一个非负数（符合位是0）.
- 如果使用符号位，那么舍入将变为朝向负无穷大而不是零（`-5 / 2` 将等于`-3` 而不是`-2`;

因此，对于一般情况，我们必须插入一些拐杖才能使其工作：:

```nasm
mov  ebx, eax
shr  ebx, 31    ; extract the sign bit
add  eax, ebx   ; add 1 to the value if it is negative to ensure rounding towards zero
sar  eax        ; this one shifts in sign bits
```

当预期只用正数时，我们也可以使用 `assume` 机制来消除 `x` 为负的情况的可能性，并避免处理此极端情况：

```cpp
int div_assume(int x) {
    assume(x >= 0);
    return x / 2;
}
```

不过在这种特殊情况下，也许表达我们只期望非负数的最佳语法是使用无符号整数类型。

由于这样的细微差别，在中间函数中扩展代数、并自己手动简化算术，通常是有益的，而不是依赖编译器来完成。

## Memory Aliasing 内存混叠

编译器在优化涉及内存读写的操作时 通常表现很差。这是因为它们通常没有足够的上下文来保证优化的正确性。

看看下面的例子：

```c++
void add(int *a, int *b, int n) {
    for (int i = 0; i < n; i++)
        a[i] += b[i];
}
```

由于此循环的每次迭代都是独立的，因此可以并行执行和矢量化。但从技术上讲，是这样吗？

如果数组a、b 相交，就可能存在问题。假设 `b == a + 1`,即 `b` 只是 `a`从第二个元素开始的 内存视图。在这种情况下，下一个迭代依赖之前的一个，唯一正确的处理方式是 顺序执行。编译器必须检查所有可能性，哪怕程序员知道这不可能发生。

这就是为什么有 `const` 和 `restrict` 关键字。第一个强制我们不会修改指针变量执行的内存，第二个是告诉编译器保证不会重叠。
```cpp
void add(int * __restrict__ a, const int * __restrict__ b, int n) {
    for (int i = 0; i < n; i++)
        a[i] += b[i];
}
```


这些关键字用于自我记录（文档？）也是一个好主意，。

## C++ Contracts契约

契约编程是一个未被充分利用 但是非常强大的技术。

有一个最新提案，通过  [contract attributes](http://www.hellenico.gr/cpp/w/cpp/language/attributes/contract.html) 添加契约设计到 C++标准，功能上等价于我们上面手写的 、编程器特定的`assume`:

```c++
T at(size_t k) [[ expects: k < n ]] {
    return _memory[k];
}
```

有三种属性 — `expects`, `ensures`, and `assert` — 分别用于 函数的前置、后置条件 ，以及可以放到任何地方的一般断言.

不幸的是， [这个新特性还没有被标准化](https://www.reddit.com/r/cpp/comments/cmk7ek/what_happened_to_c20_contracts/), 更不用说在主要的C++编译器中实现了. 但也许，几年后，我们将能够编写这样的代码：

```c++
bool is_power_of_two(int m) {
    return m > 0 && (m & (m - 1) == 0);
}

int mod_power_of_two(int x, int m)
    [[ expects: x >= 0 ]]
    [[ expects: is_power_of_two(m) ]]
    [[ ensures r: r >= 0 && r < m ]]
{
    int r = x & (m - 1);
    [[ assert: r = x % m ]];
    return r;
}
```

某些形式的合约编程也可以在其他面向性能的语言中使用，例如[Rust](https://docs.rs/contracts/latest/contracts/) and [D](https://dlang.org/spec/contracts.html).

一个与语言无关的通用建议是始终检查编译器生成的汇编，如果它不是您所希望的，请尝试考虑可能限制编译器优化它的极端情况。