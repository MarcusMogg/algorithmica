
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

无条件跳转`jmp` 只能用于实现 `while(true)` 或者 switch 之类的。一系列 条件跳转 用于实现真实的控制流。

It is reasonable to think that these conditions are computed as `bool`-s somewhere and passed to conditional jumps as operands: after all, this is how it works in programming languages. But that is not how it is implemented in hardware. Conditional operations use a special `FLAGS` register, which first needs to be populated by executing instructions that perform some kind of check.

In our example, `cmp rax, rcx` compares the iterator `rax` with the end-of-array pointer `rcx`. This updates the `FLAGS` register, and now it can be used by `jne loop`, which looks up a certain bit there that tells whether the two values are equal or not, and then either jumps back to the beginning or continues to the next instruction, thus breaking the loop.

### Loop Unrolling

One thing you might have noticed about the loop above is that there is a lot of overhead to process a single element. During each cycle, there is only one useful instruction executed, and the other 3 are incrementing the iterator and trying to find out if we are done yet.

What we can do is to *unroll* the loop by grouping iterations together — equivalent to writing something like this in C:

```c++
for (int i = 0; i < n; i += 4) {
    s += a[i];
    s += a[i + 1];
    s += a[i + 2];
    s += a[i + 3];
}
```

In assembly, it would look something like this:

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

Now we only need 3 loop control instructions for 4 useful ones (an improvement from $\frac{1}{4}$ to $\frac{4}{7}$ in terms of efficiency), and this can be continued to reduce the overhead almost to zero.

In practice, unrolling loops isn't always necessary for performance because modern processors don't actually execute instructions one-by-one, but maintain a [queue of pending instructions](/hpc/pipelining) so that two independent operations can be executed concurrently without waiting for each other to finish.

This is our case too: the real speedup from unrolling won't be fourfold, because the operations of incrementing the counter and checking if we are done are independent from the loop body, and can be scheduled to run concurrently with it. But may still be beneficial to [ask the compiler](/hpc/compilation/situational) to unroll it to some extent.

### An Alternative Approach

You don't have to explicitly use `cmp` or a similar instruction to make a conditional jump. Many other instructions either read or modify the `FLAGS` register, sometimes as a by-product enabling optional exception checks.

For example, `add` always sets a number of flags, denoting whether the result is zero, is negative, whether an overflow or an underflow occurred, and so on. Taking advantage of this mechanism, compilers often produce loops like this:

```nasm
    mov  rax, -100  ; replace 100 with the array size
loop:
    add  edx, DWORD PTR [rax + 100 + rcx]
    add  rax, 4
    jnz  loop       ; checks if the result is zero
```

This code is a bit harder to read for a human, but it is one instruction shorter in the repeated part, which may meaningfully affect performance.

<!--

### A More Complex Example

Let's do a more complicated example.

```c++
int collatz(int n) {
    int cnt = 0;
    while (n != 1) {
        cnt++;
        if (n & 2 == 1)
            n = 3 * n + 1;
        else
            n = n / 2;
    }
    return cnt;
}
```

It is a notoriously difficult math problem that seems ridiculously simple.

Make use of [lea instruction](../assembly).

E.g., if you want to make a computational experiment [Collatz conjecture](https://en.wikipedia.org/wiki/Collatz_conjecture), you may use `lea rax, [rax + rax * 2 + 1]`, and then try to `sar` it.

Another way is to check add.

Eliminating branching. Or at least making it easier for the compiler to predict which instructions are going to be executed next.

tzcnt

cmov

Need to somehow link it to branchless programming and layout article. We now have 3 places introducing the concept.

Many other operations set something in the `FLAGS` register. For example, add often. It is useful to, and then decrement or increment it to save on instruction. Like a while loop:

```
while (n--) {
    // ...
}
```

There is an important "conditional move" operation.

-->
