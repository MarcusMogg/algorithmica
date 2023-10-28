

与其他算术运算相比，除法在 x86 和一般计算机上的效果非常差。众所周知，浮点和整数除法在硬件中都很难实现。该电路在ALU中占用大量空间，计算有很多阶段，因此，`div` 和其兄弟姐妹通常需要10-20个周期才能完成， 在较小的数据类型大延迟略低。

## x86 中的除法和取模


由于没有人愿意为单独的模运算复制所有这些复杂操作， `div` 指令可以达到同时这两个目的。要执行 32 位整数除法，需要将被除数专门放入 `eax` 寄存器中，并使用除数作为唯一操作数调用 `div`。商将存储在`eax` ，余数将存储在`edx`中

唯一需要注意的是，除法实际上需要存储在两个寄存器中，`eax` `edx` ：这种机制支持64-32甚至128-64的除法， 类似于128位乘法的工作方式。在执行通常的 32 - 32 有符号除法时，我们需要将`eax` 符号扩展到 64 位并将其较高部分存储在 `edx` ：

```nasm
div(int, int):
    mov  eax, edi
    cdq  // 把EAX 的第32 bit 复制到EDX 的每一个bit 上
    idiv esi
    ret
```

对于无符号除法，您可以设置为 `edx` 零，这样它就不会干扰：

```nasm
div(unsigned, unsigned):
    mov  eax, edi
    xor  edx, edx
    div  esi
    ret
```

在这两种情况下，除了`eax`  中的 商之外，您还可以访问余数 `edx` ：
```nasm
mod(unsigned, unsigned):
    mov  eax, edi
    xor  edx, edx 
    div  esi
    mov  eax, edx
    ret
```

您还可以将 128 位整数（存储在`rdx:rax` 中 ）除以 64 位整数：

```nasm
div(u128, u64):
    ; a = rdi + rsi, b = rdx
    mov  rcx, rdx
    mov  rax, rdi
    mov  rdx, rsi
    div  edx 
    ret
```


被除数 的高部分应小于除数，否则会发生溢出。由于此约束，[很难](https://danlark.org/2020/06/14/128-bit-division/)让编译器自己生成此代码：如果将 128 位整数类型除以 64 位整数，编译器将通过其他检查将其气泡包装，这实际上可能是不必要的。

## 除常量

即使在硬件中完全实现，整数除法也非常慢，但在某些情况下，如果除数是恒定的，则可以避免。一个众所周知的例子是除以 2 的幂，可以用一个二进制移位代替

在一般情况下，有几个聪明的技巧可以用乘法代替除法，代价是一些预计算。所有这些技巧都基于以下想法。考虑将一个浮点数除以另一个浮点数 的任务，当事先知道y时，我们能做的是计算一个常数

$$
d \approx y^{-1}
$$

然后，在运行时，我们将计算

$$
x / y = x \cdot y^{-1} \approx x \cdot d
$$

 $\frac{1}{y}$ 的结果误差最多为 $\epsilon$ , 乘法 $x \cdot d$ 只会添加另一个 $\epsilon$ ，因此误差最多 $2 \epsilon + \epsilon^2 = O(\epsilon)$ , 这对于浮点情况是可以容忍的。.


### Barrett Reduction

如何为整数推广这个技巧？计算`int d = 1 / y` 不起作用，因为结果为0. 我们能做的最好的事情就是将其表达为

$$
d = \frac{m}{2^s}
$$

and then find a "magic" number $m$ and a binary shift $s$ such that `x / y == (x * m) >> s` for all `x` within range.

$$
  \lfloor x / y \rfloor
= \lfloor x \cdot y^{-1} \rfloor
= \lfloor x \cdot d \rfloor
= \lfloor x \cdot \frac{m}{2^s} \rfloor
$$

It can be shown that such a pair always exists, and compilers actually perform an optimization like that by themselves. Every time they encounter a division by a constant, they replace it with a multiplication and a binary shift. Here is the generated assembly for dividing an `unsigned long long` by $(10^9 + 7)$:

```nasm
;  input (rdi): x
; output (rax): x mod (m=1e9+7)
mov    rax, rdi
movabs rdx, -8543223828751151131  ; load magic constant into a register
mul    rdx                        ; perform multiplication
mov    rax, rdx
shr    rax, 29                    ; binary shift of the result
```

This technique is called *Barrett reduction*, and it's called "reduction" because it is mostly used for modulo operations, which can be replaced with a single division, multiplication and subtraction by the virtue of this formula:

$$
r = x - \lfloor x / y \rfloor \cdot y
$$

This method requires some precomputation, including performing one actual division. Therefore, this is only beneficial when you perform not just one but a few divisions, all with the same constant divisor.

### Why It Works

It is not very clear why such $m$ and $s$ always exist, let alone how to find them. But given a fixed $s$, intuition tells us that $m$ should be as close to $2^s/y$ as possible for $2^s$ to cancel out. So there are two natural choices: $\lfloor 2^s/y \rfloor$ and $\lceil 2^s/y \rceil$. The first one doesn't work, because if you substitute

$$
\Bigl \lfloor \frac{x \cdot \lfloor 2^s/y \rfloor}{2^s} \Bigr \rfloor
$$

then for any integer $\frac{x}{y}$ where $y$ is not even, the result will be strictly less than the truth. This only leaves the other case, $m = \lceil 2^s/y \rceil$. Now, let's try to derive the lower and upper bounds for the result of the computation:

$$
  \lfloor x / y \rfloor
= \Bigl \lfloor \frac{x \cdot m}{2^s} \Bigr \rfloor
= \Bigl \lfloor \frac{x \cdot \lceil  2^s /y \rceil}{2^s} \Bigr \rfloor
$$

Let's start with the bounds for $m$:

$$
2^s / y
\le
\lceil 2^s / y \rceil
<
2^s / y + 1
$$

And now for the whole expression:

$$
x / y - 1
<
\Bigl \lfloor \frac{x \cdot \lceil  2^s /y \rceil}{2^s} \Bigr \rfloor
<
x / y + x / 2^s
$$

We can see that the result falls somewhere in a range of size $(1 + \frac{x}{2^s})$, and if this range always has exactly one integer for all possible $x / y$, then the algorithm is guaranteed to give the right answer. Turns out, we can always set $s$ to be high enough to achieve it.

What will be the worst case here? How to pick $x$ and $y$ so that the $(x/y - 1, x/y + x / 2^s)$ range contains two integers? We can see that integer ratios don't work because the left border is not included, and assuming $x/2^s < 1$, only $x/y$ itself will be in the range. The worst case is actually the $x/y$ that comes closest to $1$ without exceeding it. For $n$-bit integers, that is the second-largest possible integer divided by the first-largest:

$$
\begin{aligned}
    x = 2^n - 2
\\  y = 2^n - 1
\end{aligned}
$$

In this case, the lower bound will be $(\frac{2^n-2}{2^n-1} - 1)$ and the upper bound will be $(\frac{2^n-2}{2^n-1} + \frac{2^n-2}{2^s})$. The left border is as close to a whole number as possible, and the size of the whole range is the second largest possible. And here is the punchline: if $s \ge n$, then the only integer contained in this range is $1$, and so the algorithm will always return it.

### Lemire Reduction

Barrett reduction is a bit complicated, and also generates a length instruction sequence for modulo because it is computed indirectly. There is a new ([2019](https://arxiv.org/pdf/1902.01961.pdf)) method, which is simpler and actually faster for modulo in some cases. It doesn't have a conventional name yet, but I am going to refer to it as [Lemire](https://lemire.me/blog/) reduction.

Here is the main idea. Consider the floating-point representation of some integer fraction:

$$
\frac{179}{6} = 11101.1101010101\ldots = 29\tfrac{5}{6} \approx 29.83
$$

How can we "dissect" it to get the parts we need?

- To get the integer part (29), we can just floor or truncate it before the dot.
- To get the fractional part (⅚), we can just take what is after the dots.
- To get the remainder (5), we can multiply the fractional part by the divisor.

Now, for 32-bit integers, we can set $s = 64$ and look at the computation that we do in the multiply-and-shift scheme:

$$
  \lfloor x / y \rfloor
= \Bigl \lfloor \frac{x \cdot m}{2^s} \Bigr \rfloor
= \Bigl \lfloor \frac{x \cdot \lceil  2^s /y \rceil}{2^s} \Bigr \rfloor
$$

What we really do here is we multiply $x$ by a floating-point constant ($x \cdot m$) and then truncate the result $(\lfloor \frac{\cdot}{2^s} \rfloor)$.

What if we took not the highest bits but the lowest? This would correspond to the fractional part — and if we multiply it back by $y$ and truncate the result, this will be exactly the remainder:

$$
r = \Bigl \lfloor \frac{ (x \cdot \lceil  2^s /y \rceil \bmod 2^s) \cdot y }{2^s} \Bigr \rfloor
$$

This works perfectly because what we do here can be interpreted as just three chained floating-point multiplications with the total relative error of $O(\epsilon)$. Since $\epsilon = O(\frac{1}{2^s})$ and $s = 2n$, the error will always be less than one, and hence the result will be exact.

```c++
uint32_t y;

uint64_t m = uint64_t(-1) / y + 1; // ceil(2^64 / y)

uint32_t mod(uint32_t x) {
    uint64_t lowbits = m * x;
    return ((__uint128_t) lowbits * y) >> 64; 
}

uint32_t div(uint32_t x) {
    return ((__uint128_t) m * x) >> 64;
}
```

We can also check divisibility of $x$ by $y$ with just one multiplication using the fact that the remainder of division is zero if and only if the fractional part (the lower 64 bits of $m \cdot x$) does not exceed $m$ (otherwise, it would become a nonzero number when multiplied back by $y$ and right-shifted by 64).

```c++
bool is_divisible(uint32_t x) {
    return m * x < m;
}
```

The only downside of this method is that it needs integer types four times the original size to perform the multiplication, while other reduction methods can work with just the double.

There is also a way to compute 64x64 modulo by carefully manipulating the halves of intermediate results; the implementation is left as an exercise to the reader.

### Further Reading

Check out [libdivide](https://github.com/ridiculousfish/libdivide) and [GMP](https://gmplib.org/) for more general implementations of optimized integer division.

It is also worth reading [Hacker's Delight](https://www.amazon.com/Hackers-Delight-2nd-Henry-Warren/dp/0321842685), which has a whole chapter dedicated to integer division.
