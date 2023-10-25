
浮点数舍入在硬件中的工作方式非常简单：当且仅当操作结果不能完全表示时，才会发生这种情况，并且默认情况下四舍五入到最接近的可表示数字（在平局的情况下选择以零结尾的数字）。

请考虑以下代码片段：

```c++
float x = 0;
for (int i = 0; i < (1 << 25); i++)
    x++;
printf("%f\n", x);
```

它不是打印 $2^{25} = 33554432$ （数学上的结果），而是输出 $16777216 = 2^{24}$.为什么？

当我们反复递增一个浮点数 x 时，我们最终会达到一个点，它变得如此之大，以至于 (x+1) 四舍五入到 x 。第一个这样的数字是  $2^{24}$ （24是尾数位数加一），因为

$$2^{24} + 1 = 2^{24} \cdot (1.\underbrace{0\ldots0}_{\times 23} 1)$$

这个值 距离 $2^{24}$ 和 $(2^{24} + 1)$相同，但是被上述决胜规则向下舍入 到$2^{24}$ 。同时，低于该值的所有内容的加法可以精确地表示，因此之前不会发生四舍五入。

## 舍入错误和操作顺序

浮点计算的结果可能取决于运算顺序，尽管它在代数上是正确的。

例如，虽然加法和乘法在纯数学意义上是可交换的和结合的，但它们的舍入误差不是：当我们有三个浮点变量 $x$, $y$, and $z$时， $(x+y+z)$ 的结果取决于求和的顺序。相同的非交换性原则适用于大多数其他浮点运算。

编译器不允许生成不符合规范的结果，因此这种烦人的细微差别禁用了一些潜在的优化，这些优化涉及在算术中重新排列操作数。您可以在 GCC 和 Clang 中 使用`-ffast-math`禁用对 标志的严格遵守。如果我们添加它并重新编译上面的代码片段，它的运行速度要快得多，并且还恰好输出正确的结果，33554432（尽管您需要注意编译器也可以选择不太精确的计算路径）。

## 舍入模式

除了默认模式（也称为 Banker 舍入）之外，您还可以使用另外 4 种模式[设置](https://cplusplus.com/reference/cfenv/fesetround/)其他舍入逻辑：

- 四舍五入到最近，平局总是“远离”零;
- 向上舍入（朝向  $+∞$;负结果因此四舍五入为零）;
- 向下舍入（朝向  $-∞$;负结果因此四舍五入为“远离”零）;
- 舍入到零（二进制结果的截断）。


修改舍入模式的用途之一是诊断数值不稳定性。如果在舍入到正无穷大和负无穷大之间切换时算法的结果有很大差异，则表明容易出现舍入误差。

此测试通常比将所有计算切换为较低的精度并检查结果是否变化太大更好，因为默认的舍入到最接近策略收敛到正确的“预期”值，给定足够的平均值：一半的时间误差向上舍入，另一半时间向下舍入 - 因此，从统计上讲，它们相互抵消。

## 测量错误

期望从执行复杂计算（如自然对数和平方根）的硬件获得这种保证似乎令人惊讶，但就是这样：您可以保证从所有操作中获得尽可能高的精度。这使得分析舍入误差变得非常容易，正如我们稍后将看到的那样。

有两种自然的方法可以测量计算误差：

* 创建硬件或符合规范的精确软件的工程师关注的是最后一位单位（[ulps](https://en.wikipedia.org/wiki/Unit_in_the_last_place)），即两个数字之间的距离，即在精确的实数值和计算的实际结果之间可以容纳多少个可表示的数字。
* 从事数值算法工作的人关心相对精度，即近似误差除以真实答案的绝对值：$|\frac{v-v'}{v}|$

无论哪种情况，分析错误的常用策略都是假设最坏的情况并简单地限制它们。

如果您执行单个基本算术运算，那么可能发生的最糟糕的事情是结果四舍五入到最接近的可表示数字，这意味着误差不超过 0.5 ulps。为了以同样的方式推理相对误差，我们可以定义一个称为机器 epsilon 的数字 $\epsilon$  ，等于1与下一个可表示值之间的差（它应该等于 2 表示任何专用于尾数的位的负幂）。

这意味着，如果在一次算术运算后你得到结果  $x$ ，那么实际值在范围内的某个地方

$$
[x \cdot (1-\epsilon),\; x \cdot (1 + \epsilon)]
$$
在根据浮点计算结果做出离散的“是或否”决策时，记住错误的普遍存在尤为重要。例如，您应该如何检查相等性：

```c++
const float eps = std::numeric_limits<float>::epsilon; // ~2^(-23)
bool eq(float a, float b) {
    return abs(a - b) <= eps;
}
```

`eps`的值应该取决于应用程序：上面的那个 - `float`的机器epsilon  - 只适用于不超过一个的浮点运算。
## Interval Arithmetic区间数学

如果算法的误差（无论其原因如何）在计算过程中没有变大，则称为数值稳定 *numerically stable*。这只有在问题本身条件良好的情况下才会发生，这意味着如果输入数据发生少量更改，则解决方案只会发生少量更改。

在分析数值算法时，采用与实验物理学相同的方法通常是有用的：我们将使用它们可能所在的区间，而不是使用未知的实值。

例如，考虑一个操作链，其中我们连续将变量乘以任意实数：

```cpp
float x = 1;
for (int i = 0; i < n; i++)
    x *= a[i];
```

在第一次乘法之后，  $x$ 相对于实积的值以  $(1 + \epsilon)$为界，并且每增加一次乘法后，该上限乘以另一个  $(1 + \epsilon)$, 。通过归纳法，n次乘法后 ，计算值受和类似的下限约束 $(1 + \epsilon)^n = 1 + n \epsilon + O(\epsilon^2)$  。

这意味着相对误差是 $O(n \epsilon)$，通常是没问题的，因为通常 $n \ll \frac{1}{\epsilon}$

数值不稳定计算的例子，请考虑函数

$$
f(x, y) = x^2 - y^2
$$

假设 $x > y$, 这个函数可能返回的最大值大致为：

$$
x^2 \cdot (1 + \epsilon) - y^2 \cdot (1 - \epsilon)
$$

对应的绝对误差
$$
x^2 \cdot (1 + \epsilon) - y^2 \cdot (1 - \epsilon) - (x^2 - y^2) = (x^2 + y^2) \cdot \epsilon
$$

因此相对误差

$$
\frac{x^2 + y^2}{x^2 - y^2} \cdot \epsilon
$$

如果$x$ and $y$ 的大小接近，那么误差为 $O(\epsilon \cdot |x|)$.

在直接计算下，减法“放大”平方误差。但这可以通过使用以下公式来解决：

$$
f(x, y) = x^2 - y^2 = (x + y) \cdot (x - y)
$$

在这个中，很容易表明 误差 以 $\epsilon \cdot |x - y|$.为界。

## Kahan 求和

https://oi-wiki.org/misc/kahan-summation/

From the previous example, we can see that long chains of operations are not a problem, but adding and subtracting numbers of different magnitude is. The general approach to dealing with such problems is to try to keep big numbers with big numbers and small numbers with small numbers.

Consider the standard summation algorithm:

```c++
float s = 0;
for (int i = 0; i < n; i++)
    s += a[i];
```

Since we are performing summations and not multiplications, its relative error is no longer just bounded by $O(\epsilon \cdot n)$, but heavily depends on the input.

In the most ridiculous case, if the first value is $2^{24}$ and the other values are equal to $1$, the sum is going to be $2^{24}$ regardless of $n$, which can be verified by executing the following code and observing that it simply prints $16777216 = 2^{24}$ twice:

```cpp
const int n = (1<<24);
printf("%d\n", n);

float s = n;
for (int i = 0; i < n; i++)
    s += 1.0;

printf("%f\n", s);
```

This happens because `float` has only 23 mantissa bits, and so $2^{24} + 1$ is the first integer number that can't be represented exactly and has to be rounded down, which happens every time we try to add $1$ to $s = 2^{24}$. The error is indeed $O(n \cdot \epsilon)$ but in terms of the absolute error, not the relative one: in the example above, it is $2$, and it would go up to infinity if the last number happened to be $-2^{24}$.

The obvious solution is to switch to a larger type such as `double`, but this isn't really a scalable method. An elegant solution is to store the parts that weren't added in a separate variable, which is then added to the next variable:

```c++
float s = 0, c = 0;
// 核心是 c保存上一次计算的误差
for (int i = 0; i < n; i++) {
    float y = a[i] - c; // c is zero on the first iteration
    float t = s + y;    // s may be big and y may be small, losing low-order bits of y
    c = (t - s) - y;    // (t - s) cancels high-order part of y 
    s = t;
}
```

This trick is known as *Kahan summation*. Its relative error is bounded by $2 \epsilon + O(n \epsilon^2)$: the first term comes from the very last summation, and the second term is due to the fact that we work with less-than-epsilon errors on each step.

Of course, a more general approach that works not just for array summation would be to switch to a more precise data type, like `double`, also effectively squaring the machine epsilon. Furthermore, it can (sort of) be scaled by bundling two `double` variables together: one for storing the value and another for its non-representable errors so that they represent the value $a+b$. This approach is known as double-double arithmetic, and it can be similarly generalized to define quad-double and higher precision arithmetic.

