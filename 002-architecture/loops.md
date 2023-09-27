
来看一个稍微复杂一点的例子

```nasm
loop:
    add  edx, DWORD PTR [rax]
    add  rax, 4
    cmp  rax, rcx
    jne  loop
```

一个简单的`for` 循环 计算 32-bit 整数数组的和。

循环体是 `add edx, DWORD PTR [rax]`: 这个指令从迭代器 `rax` 加载数据，然后累加到 `edx`. 接着迭代器移动4字节 `add rax, 4`. 最后发生了稍微复杂的事.

## 跳转 Jumps


汇编不像高级语言这样有 `if`, `for` , 函数 或者其他控制流结构，只有`goto` 或者叫“jump”

**Jump** 将指令指针移动到其操作数所指定的位置。这个位置可以是内存中的绝对地址，也可以是相对当前位置的地址，还可以是运行时计算的值。
为了避免直接管理内存的头疼，你可以在任意指令前 用 后面跟着 `:`的 字符串  来标记，然后你就可以使用 这个字符串作为标签。在转换成机器码时，标签会转换为相对地址。

标签可以是任意值，但编译器不是具有创造性的，在为标签选择名称时，[通常](https://godbolt.org/z/T45x8GKa5)只使用源代码中的行号和带有签名的函数名称。

无条件跳转`jmp` 只能用于实现 `while(true)` 或者 switch 之类的。一系列 **条件跳转** 用于实现真实的控制流。

可以合理的认为将 条件计算为bool然后传递给 条件跳转指令作为操作数：毕竟，编程语言就是这样运行的。但是，硬件不是这样实现的。条件判断操作使用了一个特殊的寄存器`FLAG`, 首先需要通过执行某种检查的指令来填充该寄存器

在我们的例子中，`cmp rax, rcx` 比较了迭代器`rax`和 数组结束指针 `rcx`。这会更新寄存器，然后就可以使用 `jne loop`。 它会查找指定的位，然后告诉我们 这两个值倒是是不是相等，然后跳转回循环开始 或者继续执行下一条指令（即跳出循环）


## 循环展开

你可能已经注意到，上面的循环处理单个元素有许多额外开销。每个循环中只有一个指令有用，其他三个指令都是用于处理循环是否结束

我们可以把这个循环展开（*unroll* ），等价于下面的c代码

```c++
for (int i = 0; i < n; i += 4) {
    s += a[i];
    s += a[i + 1];
    s += a[i + 2];
    s += a[i + 3];
}
```

汇编：

```nasm
loop:
    add  edx, [rax]
    add  edx, [rax+4]
    add  edx, [rax+8]
    add  edx, [rax+12]
    add  rax, 16
    cmp  rax, rsi
    jne  loop
```

先在有 3 个循环控制指令， 4 个有用的指令 (从$\frac{1}{4}$ 提升到 $\frac{4}{7}$ ), 这可以继续 展开直到额外负载为0.

但在实践中，循环展开对性能往往不是必要的，因为现代处理器不是一条一条执行指令的，而是维护指令队列，所以可以并行执行两条无关的操作，而不用等待彼此操作完成。

对我们的例子也是如此，循环展开 对速度的提升不会有四倍（~~实测约等于没有~~），因为递增和检查指令 独立于循环体，所以可以并行执行。但要求编译器在某种程度上展开它可能仍然是有益的。

## 一个替代方法

不必显示使用`cmp`或者其他相似的指令来进行条件跳转。还有许多其他的指令可以读写 `FLAGS`寄存器，有时作为副产品 启用可选的异常检查。

For example, `add` always sets a number of flags, denoting whether the result is zero, is negative, whether an overflow or an underflow occurred, and so on. Taking advantage of this mechanism, compilers often produce loops like this:

例如，`add` 往往设置一系列寄存器，取决于结果是 0、是负数、是否发生溢出……

```nasm
    mov  rax, -100  ; replace 100 with the array size
loop:
    add  edx, DWORD PTR [rax + 100 + rcx]
    add  rax, 4
    jnz  loop       ; checks if the result is zero
```

This code is a bit harder to read for a human, but it is one instruction shorter in the repeated part, which may meaningfully affect performance.

