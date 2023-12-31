
浮点算术的用户应该得到这些智商钟形曲线memes之一 —— 因为它与大多数人之间的关系通常是这样进行的：

- 初学者到处使用，就好像它是无限精度的魔法数据类型。
- 然后他们发现 `0.1 + 0.2 != 0.3` 或者类似的怪事，吓坏了，开始认为每次计算都会添加一些随机误差项，并且多年来完全避免任何实数类型。
- 最后他们终于站起来，阅读了IEEE-754 float如何工作的规范，并开始适当地使用它们。

不幸的是，太多人仍处于第 2 阶段，滋生了对浮点算术的各种误解——认为它从根本上是不精确和不稳定的，并且比整数算术慢。

![](../img/iq.svg)

但这些都是谬论。浮点数计算通常比整数计算要快，因为有专用指令。实数表示是完全标准化的，并且在舍入方面遵循简单和确定的规则，使你能够可靠地管理计算错误。

事实上，它是如此可靠，以至于一些高级编程语言，尤其是JavaScript，根本没有整数。在 JavaScript 中，只有一种 `number` 类型，它在内部存储为 64 位 `double` ，并且由于浮点运算的工作方式， $-2^{53}$ and $2^{53}$ 之间的整数和结果都可以精确存储，因此从程序员的角度来看，实际上几乎不需要单独的整数类型。

一个值得注意的例外是，当需要对数字执行按位运算时，浮点单元（负责对浮点数进行操作的协处理器）通常不支持这种运算。在这种情况下，它们需要转换为整数，这在支持 JavaScript 的浏览器中非常频繁使用，因此 arm 添加了一个特殊的[“FJCVTZS”指令](https://community.arm.com/developer/ip-products/processors/b/processors-ip-blog/posts/armv8-a-architecture-2016-additions)，代表“浮点 Javascript 转换为有符号定点，四舍五入到零”并按照它所说的去做- 以与 JavaScript 完全相同的方式将实数转换为整数 - 这是软件 - 硬件反馈循环的一个有趣例子。

但是，除非你是专门使用实数类型来模拟整数算术的 JavaScript 开发人员，否则你可能需要有关浮点运算的更深入的指南，因此我们将从一个更广泛的主题开始。

## 实数表示

如果需要处理实数（非整数），有几个适用性不同的选项。在直接跳到浮点数之前，，我们想讨论可用的替代方案及其背后的动机 - 毕竟，避免浮点运算的人确实有道理。

### Symbolic Expressions 

第一种也是最繁琐的方法是： 不存储结果值本身，而是存储生成它们的代数表达式。

下面是一个简单的示例。在某些应用中，例如计算几何，除了加减乘数字外，还需要除法而且不四舍五入，产生一个有理数，可以用两个整数的比率精确表示：


```c++
struct r {
    int x, y;
};

r operator+(r a, r b) { return {a.x * b.y + a.y * b.x, a.y * b.y}; }
r operator*(r a, r b) { return {a.x * b.x, a.y * b.y}; }
r operator/(r a, r b) { return {a.x * b.x, a.y * b.y}; }
bool operator<(r a, r b) { return a.x * b.y < b.x * a.y; }
// ...and so on, you get the idea
```

这个比率可以变得不可约，这甚至会使这个表示形式独一无二：

```c++
struct r {
    int x, y;
    r(int x, int y) : x(x), y(y) {
        if (y < 0)
            x = -x, y = -y;
        int g = gcd(x, y);
        x /= g;
        y /= g;
    }
};
```


这就是 WolframAlpha 和 SageMath 等计算机代数系统的工作方式：它们只对符号表达式进行操作，避免将任何内容计算为实值。

使用此方法，您可以获得绝对精度，并且当您的范围有限（例如仅支持有理数）时，它效果很好。但这需要很大的计算成本，因为一般来说，您需要以某种方式存储产生结果的整个操作历史记录，并在每次执行新操作时将其考虑在内 - 随着历史记录的增长，这很快就会变得不可行。

### 定点数

另一种方法是坚持使用整数，但将它们视为乘以固定常量。由于某些值无法准确表示，这使得计算不精确：您需要将结果舍入到最接近的可表示值。


这种方法通常用于财务软件，您确实需要一种简单的方法来管理舍入误差，以便最终数字相加。例如，纳斯达克在其股票列表中使用  $\frac{1}{10000}$-th美元作为其基本单位，这意味着您在所有交易中获得逗号后4位的精度。

```c++
struct money {
    uint v; // 1/10000th of a dollar
};

std::string to_string(money) {
    return std::format("${0}.{1:04d}", v / 10000, v % 10000);
}

money operator*(money x, money y) { return {x.v * y.v / 10000}; }
```

除了引入舍入误差之外，更大的问题是当缩放常量不合适时，它们变得毫无用处。如果您使用的数字太大，则内部整数值将溢出，如果数字太小，它们将向下舍入为零。有趣的是，前一种情况曾经成为纳斯达克的一个[问题](https://www.wsj.com/articles/berkshire-hathaways-stock-price-is-too-much-for-computers-11620168548)，当时Berkshire Hathawa公司的股价接近 $\frac{2^{32} - 1}{10000}$​ = 429,496.7295美元，无符号的32位整数无法再容纳。 

这个问题使得定点算术从根本上不适合需要同时使用小数和大数的应用，例如，计算某些物理方程：

$$
E = m c^2
$$

In this particular one, $m$ is typically of the same order of magnitude as the mass of a proton ($1.67 \cdot 10^{-27}$ kg) and $c$ is the speed of light ($3 \cdot 10^9$ m/s).

在这个特定的案例中， $m$通常与质子的质量($1.67 \cdot 10^{-27}$ kg)具有相同的数量级，  $c$ 是光速($3 \cdot 10^9$ m/s)。

### 浮点数

在大多数数值应用中，我们主要关注相对误差。比如说 我们希望我们的计算结果与事实的差异不超过，0.01% ， 我们并不真正关心这 0.01% 相当于绝对单位的值。


浮点数通过存储一定数量的最有效数字和数字的数量级来解决这个问题。更准确地说，它们用一个整数（称为有效数*significand* 或尾数*mantissa*）表示，并使用某个固定基数的指数进行缩放 - 最常见的是2或10。例如：

$$
1.2345 =
\underbrace{12345}_\text{mantissa}
\times {\underbrace{10}_\text{base}\!\!\!\!}
      ^{\overbrace{-4}^\text{exponent}}
$$


计算机使用固定长度的二进制字进行操作，因此在为硬件设计浮点格式时，您需要一种固定长度的二进制格式，其中一些位专用于尾数（为了更精确），一些位专用于指数（用于更大的范围）。

这个手工制作的浮点实现希望传达这个想法：

```cpp
// DIY floating-point number
struct fp {
    int m; // mantissa
    int e; // exponent
};
```


这样我们可以表示  $\pm \\; m \times 2^e$ 形式的数字，m和 e都是有界 ，且可能为负的整数——它们分别对应于负数或小数。这些数字的分布非常不均匀，$[0, 1)$ 和 $[1, +\infty)$ 范围内可表示的数字个数一致。

请注意，这些表示对于某些数字并不是唯一的。例如，数字 11 可以表示为

$$
1 \times 2^0 = 2 \times 2^{-1} = 256 \times 2^{-8}
$$

以及其他 28 种不会溢出尾数的方式。.


对于某些应用程序（例如比较或哈希），这可能会有问题。为了解决这个问题，我们可以使用某种约定对这些表示进行归一化。在十进制中，[标准形式](https://en.wikipedia.org/wiki/Scientific_notation)是始终将逗号放在第一个数字 （ `6.022e23` ） 之后。对于二进制，我们可以做同样的事情：

$$
42 = 10101_2 = 1.0101_2 \times 2^5
$$


请注意，按照此规则，第一位始终为 1。显式存储它是多余的，所以我们假装它在那里，只存储其他位，对应于$[0, 1)$ 范围内的某个有理数。可表示数字的集合现在大致为

$$
\{ \pm \; (1 + m) \cdot 2^e \; | \; m = \frac{x}{2^{32}}, \; x \in [0, 2^{32}) \}
$$

由于现在 $m$ 是非负值，我们现在将其设为无符号整数，而是为符号添加一个单独的布尔字段：

```cpp
struct fp {
    bool s;     // sign: "0" for "+", "1" for "-" 
    unsigned m; // mantissa
    int e;      // exponent
};
```

现在，让我们尝试使用手工制作的浮点数实现一些算术运算，例如乘法。使用新公式，结果应为：

$$
\begin{aligned}
c  &= a \cdot b
\\ &= (s_a \cdot (1 + m_a) \cdot 2^{e_a}) \cdot (s_b \cdot (1 + m_b) \cdot 2^{e_b})
\\ &= s_a \cdot s_b \cdot (1 + m_a) \cdot (1 + m_b) \cdot 2^{e_a} \cdot 2^{e_b} 
\\ &= \underbrace{s_a \cdot s_b}_{s_c} \cdot (1 + \underbrace{m_a + m_b + m_a \cdot m_b}_{m_c}) \cdot 2^{\overbrace{e_a + e_b}^{e_c}}
\end{aligned}
$$

现在看起来很容易计算，但有两个细微差别：

- 新的尾数现在在$[0, 3)$ 范围内，我们需要检查它是否大于1，并归一化 表示，使用下面的公式$1 + m = (1 + 1) + (m - 1) = (1 + \frac{m - 1}{2}) \cdot 2$
- 由于缺乏精度，结果数字可能（并且很可能）无法准确表示。我们需要两倍的位来计算该项 $m_a \cdot m_b$，我们在这里能做的最好的事情就是将其四舍五入到最接近的可表示数字。

由于我们需要一些额外的位来正确处理尾数溢出问题，因此我们将保留一位，将$m$ 限制在  $[0,2^{31})$ 范围内。

> 这里保留一位后其实符号位可以合并

```c++
fp operator*(fp a, fp b) {
    fp c;
    c.s = a.s ^ b.s;
    c.e = a.e + b.e;
    
    uint64_t x = a.m, y = b.m; // casting to wider types
    uint64_t m = (x << 31) + (y << 31) + x * y; // 62- or 63-bit intermediate result
    if (m & (1<<62)) { // checking for overflow
        m -= (1<<62); // m -= 1; 
        m >>= 1; 
        c.e++;
    }

    m += (1<<30); // "rounding" by adding 0.5 to a value that will be floored next
    c.m = m >> 31; // 这里可能使最高位溢出
    
    return c;
}
```


许多需要更高精度的应用程序以类似的方式使用软件浮点运算。但是，当然，您不希望每次要乘以两个实数时执行此代码编译的 10 条左右指令序列，因此在现代 CPU 上，浮点运算是在硬件中实现的——由于其复杂性，通常作为单独的协处理器。

x86 的浮点单元（通常称为 x87）具有单独的寄存器和自己的微小指令集，支持内存运算、基本算术、三角函数和一些常见运算，如对数、指数和平方根。为了使这些操作正确地协同工作，需要澄清浮点数表示的一些其他细节 — 我们将在下一节中进行。
