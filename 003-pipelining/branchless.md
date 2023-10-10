
和上衣章节提到的意义，分支无法预测对CPU是昂贵的，因为分支预测失败之后 会造成流水线停止来获取少量的指令。 在本章节，我们首先讨论删除分支的方法。
## Predication 谓词

继续我们之前开始的例子—— 我们创建了一个随机数字的数组 让后计算小于50的元素的和：

```c++
for (int i = 0; i < N; i++)
    a[i] = rand() % 100;

volatile int s;

for (int i = 0; i < N; i++)
    if (a[i] < 50)
        s += a[i];
```


我们的目标是消除 `if` 造成的分支。可以像这样：

```c++
for (int i = 0; i < N; i++)
    s += (a[i] < 50) * a[i];
```

现在这个循环花费 ~7个循环而不是原来的~14。而且，即便我们将 50 替换为其他值，性能也保持稳定，所以它现在是分支概率无关的。

但是等等...不应该还有分支吗？ `(a[i] < 50)` 如何映射到汇编？

汇编中没有 Boolean 类型，也没有根据比较结果产生 0/1 的指令，但是我们像这样间接计算：`(a[i] - 50) >> 31`. 这个trick 依赖整数的二进制表示，如果  `a[i] - 50` 是负数（即`a[i] < 50`），那么最高bit位就变为1，我们可以使用 右移提取出来。

```nasm
mov  ebx, eax   ; t = x
sub  ebx, 50    ; t -= 50
sar  ebx, 31    ; t >>= 31
imul  eax, ebx   ; x *= t
```

另外，有更复杂的办法来实现： 将符号bit位转换为mask，然后使用二进制运算 `and` 而不是乘法`((a[i] - 50) >> 31 - 1) & a[i]`。 这可以使整个序列快1个循环，因为不像其他指令`imul` 使用3个循环：

```nasm
mov  ebx, eax   ; t = x
sub  ebx, 50    ; t -= 50
sar  ebx, 31    ; t >>= 31
; imul  eax, ebx ; x *= t
sub  ebx, 1     ; t -= 1 (causing underflow if t = 0)
and  eax, ebx   ; x &= t
```

注意这个优化在编译器视角，在技术上是不正确的：对于最小的 50个整数 ——  $[-2^{31}, - 2^{31} + 49]$ 范围内 —— 因为溢出，结果是不正确的。我们知道 所有的数字在 0～100 ，但是编译器不知道。

但是 编译器确实做了不一样的事。 不是使用算术技巧，而是使用一个特殊的指令 `cmov`, 根据计算结果进行赋值：

```nasm
mov     ebx, 0      ; cmov doesn't support immediate values, so we need a zero register
cmp     eax, 50
cmovge  eax, ebx    ; eax = (eax >= 50 ? eax : ebx=0)
```

所以代码实际上接近于使用三目运算符：

```c++
for (int i = 0; i < N; i++)
    s += (a[i] < 50 ? a[i] : 0);
```

两种变体都可以由编译器优化并产生下面的代码:

```nasm
    mov     eax, 0
    mov     ecx, -4000000
loop:
    mov     esi, dword ptr [rdx + a + 4000000]  ; load a[i]
    cmp     esi, 50
    cmovge  esi, eax                            ; esi = (esi >= 50 ? esi : eax=0)
    add     dword ptr [rsp + 12], esi           ; s += esi
    add     rdx, 4
    jnz     loop                                ; "iterate while rdx is not zero"
```


这种通用技术被称为 谓词 *predication*, 大致等于这种代数技巧：

$$
x = c \cdot a + (1 - c) \cdot b
$$

这种方式可以消除分支，但是代价是 计算两个分支 和 `cmov`。 由于执行">=" 分支的成本不高，因此性能和 分支版本中的 “始终是“ 情况 相同

## When Predication Is Beneficial 谓词什么时候是有用的

使用谓词 消除了control hazard 但是带来了 data hazard。这里任然有 流水线停止，但是是 廉价的一个： 你只需要等待 `cmov`执行 而不是在预测错误时刷新整个流水线。

但是，有很多情况下保留分支代码更有效。比如 计算两个分支的代价 超过 分支预测失败的惩罚。

在我们的例子中，当分支 预测概率 超过 ～75%时，分支代码胜出。

![](../img/branchy-vs-branchless.svg)


编译器通常将此 75% 阈值用作确定是否使用 `cmov` 的启发式方法。不幸的是，在编译时，这个概率通常不知道，因此需要通过以下几种方式之一提供它：

- 可以使用 profile-guided optimization，让其自身决定是否使用谓词。
- 可以使用 likely、unlikely 属性，以及编译器内建函数（`__builtin_expect_with_probability` in GCC ， `__builtin_unpredictable` in Clang）来标记分支可能性。
- 我们可以使用 三目运算符和各种算数技巧 来重写分支代码，这充当程序员和编译器之间的隐式契约：如果程序员以这种方式编写代码，那么它可能意味着无分支。

”正确的方式“ 应该是使用 分支提示，但是不幸的是，编译器支持不好。现在，当编译器后端决定是否使用 `cmov`时 这些提示似乎已经丢失了，[issue](https://bugs.llvm.org/show_bug.cgi?id=40027), 有一些[进展](https://discourse.llvm.org/t/rfc-cmov-vs-branch-optimization/6040)， 但是目前为止，没有强制编译器生成无分支代码的好方法。

## Larger Examples 更大的例子

**Strings.**  一个非常简化的模型，`std::string` 可以认为是 一个 null 结束、堆上分配的`char` 数组 和一个整数表示 string大小。

string的一个常见值 是 空字符串——这也是默认值。你任然需要处理它，一个比较自然的方式是：一个空指针加字符串大小赋值为0，然后在每个涉及处理string的地方先判断 指针是否为空或者长度是否为0。

但是，这样需要一个单独的分支，代价较大（除非大多数字符串为空或非空）。为了移除检查，从而删除分支，我们可以分配 ”0字符串“，它只是在某处分配的零字节，然后简单地将所有空字符串指向那里。现在所有带有空字符串的字符串操作都必须读取这个无用的零字节，但这仍然比分支错误预测便宜得多。

**Binary search. 二分查找** 标准二分查找可以无分支实现,在小数组（适合缓存）上可以比`std::lower_bound` 快4倍:

```c++
int lower_bound(int x) {
    int *base = t, len = n;
    while (len > 1) {
        int half = len / 2;
        base += (base[half - 1] < x) * half; // will be replaced with a "cmov"
        len -= half;
    }
    return *base;
}
```


除了更复杂之外，他还有另一个小缺点，它可能会执行更多的比较（常量 $\lceil \log_2 n \rceil$  而不是  $\lfloor \log_2 n \rfloor$ 或者 $\lceil \log_2 n \rceil$），而且无法推测未来的内存读取。

通常，数据结构通过隐式或显式填充数据来实现无分支，以便其操作进行恒定的迭代次数。有关更复杂的示例，请参阅[文章](https://en.algorithmica.org/hpc/data-structures/binary-search/)。


**Data-parallel programming.** 无分支编程对SIMD应用十分重要，因为它们没有分支

在我们的数组求和示例中，从累加器中删除 `volatile` 类型限定符允许编译器对循环进行矢量化：

```c++
/* volatile */ int s = 0;

for (int i = 0; i < N; i++)
    if (a[i] < 50)
        s += a[i];
```

它现在每个元素工作~0.3循环，主要是内存的瓶颈。

编译器通常能够对迭代之间没有分支或依赖关系的任何循环进行矢量化，以及一些特定的小偏差，例如仅包含一个 if-without-else 的简化或简单循环。任何更复杂的矢量化都是一个非常重要的问题，它可能涉及各种技术，例如masking和寄存器内排列。
