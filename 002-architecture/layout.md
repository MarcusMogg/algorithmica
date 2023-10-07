
计算机工程师 精神上 喜欢把 CPU 流水线划分为两部分： 前端，从内存读取和解码指令；后端，指令调度和最终执行。典型的，性能瓶颈在于执行部分，出于这个原因，本书中的大部分工作 围绕后端优化

但是有时相反，前端填充指令的速度不过快，使得后端不饱和。这可能有多种原因，所有这些最终都需要 在某种程度上 改变机器码在内存的布局，比如 删除无用代码，交互if 分支，甚至是改变 函数声明顺序

## CPU Front-End 前端

在机器码转换成指令，CPU理解程序员想要做什么之前，有两个重要的阶段需要我们关注： *fetch* 和 *decode*.

在 **fetch**阶段，CPU简单的从内存中加载固定大小的块，包含一些 二进制编码的指令和数字。在 x64上块大小典型是 32 byte，尽管这在不同机器上各不相同。一个重要的差别是 块必须对齐： 块地址必须是 大小的倍数。

接下来是 **decode** 阶段：CPU查看二进制块，丢弃指令指针之前的内容，然后把剩余的部分 分割为 指令。机器指令编码 使用不同大小的byte：一些简单常见的指令 花费1byte，比如 `inc rax`,  而一些晦涩的指令 带上编码的常数和 前缀可以达到 15 。所以，从一个 32byte 大小的块可以 解码出不同数量的指令，但不超过一个固定的 机器相关的限制 叫做 解码宽度*decode width*。 在我的CPU（ [Zen 2](https://en.wikichip.org/wiki/amd/microarchitectures/zen_2)）上是4，这意味着 每个周期，最多只有4个指令可以被解码并发送到下一阶段。

这个阶段以流水线的形式工作： 如果CPU 可以 告诉（或着预测）下一个指令块需要什么，那么 fetch阶段可以 不用等待上一条指令完成 就去加载和 解码下一条。

## Code Alignment 代码对齐

在其他条件相同的情况下，编译器通常会选择 机器码更短的指令，因为这样 一个 32B 的 fetch 块可以包含更多指令，而且可以减少二进制大小。但有时相反，因为 fetch的 指令块必须对齐。

试想一下 你需要执行一个起始位置在 32B对齐块最后一个字节的 指令序列。你可能可以执行第一条指令而不需要额外延时，但是对于剩余部分，你必须等一个周期来加载其他指令。如果代码块可以32B边界对齐，那么最多4条指令可以被解码 让后并发执行。

考虑到这一点，编译器有时会做一些看起来有害的优化：它有时会选择那些 机器码更长的指令，甚至插入无用指令 来使得关键跳转指令对齐在一个合适的2的幂次方边界

GCC中，你可以使用 `-falign-labels=n`  指定特殊的对齐策略，如果你想有更精细的选择，可以 [替换](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html) `-labels` 为 `-function`, `-loops`, or `-jumps` 。 在 `-O2` 或者 `-O3`基本这是默认开启的，如果没设置特定对齐，通常会选择机器相关的默认值
## Instruction Cache 指令缓存


指令的存储、读取  和 数据的内存系统有很多相同，除了低级别缓存被替换为一个独立的指令缓存（因为你不希望随机的数据读取 赶走 处理它的代码）

指令缓存在某些场景下至关重要：

-  不知道你将执行的下一条指令，需要低延时加载
-  需要执行一长串 复杂但是快速执行的指令，并且需要高带宽

对于机器码很大的应用来说，内存可能成为瓶颈。这可能会影响我们之前讨论的优化技术的应用

- Inline 不总是有效的, 因为它减少了代码共享并增大二进制体积，需要更多的指令缓存
- 循环展开只在某些上下文有效，即便迭代数量在编译期已知：在某些时候，CPU需要从内存同时获取指令和数据，在这种情况下，内存带宽会成为瓶颈。
- 大的代码对齐 会增加二进制体积，进而需要更大的指令缓存。缓存未命中和等待指令获取 会 额外1个周期的惩罚。

另一方面，把频繁使用的指令串 放在相同的缓存行和 内存页 可以改善 缓存局部性。为了提升缓存利用率，你应该把 热代码和热代码 放在一起，冷代码和冷代码放在一起，删除无用代码。如果你想探索更多，可以查看 Facebook 的[Binary Optimization and Layout Tool](https://engineering.fb.com/2018/06/19/data-infrastructure/accelerate-large-scale-applications-with-bolt/)

## Unequal Branches 不相等分支

计算整数区间长度：

```c++
int length(int x, int y) {
    if (x > y)
        return x - y;
    else
        return y - x;
}
```

在x86汇编中，实现方式有许多变量，可能会影响性能。让我们首先直接把代码转换为汇编

```nasm
length:
    cmp  edi, esi
    jle  less
    ; x > y
    sub  edi, esi
    mov  eax, edi
done:
    ret
less:
    ; x <= y
    sub  esi, edi
    mov  eax, esi
    jmp  done
```


开始的c代码可能十分对称，但是汇编视角不是。这倒是的结果是一个分支的执行比另一个分支要快的多： 如果 `x > y` CPU只需要执行`cmp` 和 `ret`之间的5条指令，如果函数是对齐的，可以在一次 fetch完成；但在`x <= y`  时 需要两个额外跳转，

有理由认为 `x > y` 是 *unlikely*，更像是几乎不会发生的异常。这个例子里我们可以简单的交换 xy

```c++
int length(int x, int y) {
    if (x > y)
        swap(x, y);
    return y - x;
}
```

汇编代码可能是这样：

```nasm
length:
    cmp  edi, esi
    jle  normal     ; if x <= y, no swap is needed, and we can skip the xchg
    xchg edi, esi
normal:
    sub  esi, edi
    mov  eax, esi
    ret
```

总指令数现在为6 最多为8。但这对我们的汇编来说 不是个明显的优化：如果我们任务`x > y` 从不发生，那 加载 `xchg edi, esi` 指令对我们来说就是浪费，因为它几乎不执行。我们可以通过把它移除正常执行路径来解决这个问题

```nasm
length:
    cmp  edi, esi
    jg   swap
normal:
    sub  esi, edi
    mov  eax, esi
    ret
swap:
    xchg edi, esi
    jmp normal
```

这个技巧对处理异常情况 十分便利，在高级别代码中，你可以给编译器提示，告诉它 某个分支是 更可能发生的

```c++
int length(int x, int y) {
    if (x > y) [[unlikely]]
        swap(x, y);
    return y - x;
}
```

This optimization is only beneficial when you know that a branch is very rarely taken. When this is not the case, there are [other aspects](/hpc/pipelining/hazards) more important than the code layout, that compel compilers to avoid any branching at all — in this case by replacing it with a special "conditional move" instruction, roughly corresponding to the ternary expression `(x > y ? y - x : x - y)` or calling `abs(x - y)`:

这个优化只有你知道哪个分支几乎不可能发生时有用。如果不是这种情况，会有其他问题严重影响代码布局，迫使编译器尽量避免任何分支——在这种情况下，将其替换为特殊的“条件移动”指令，大致对应于三元表达式`(x > y ? y - x : x - y)` 或者 `abs(x - y)`:

```nasm
length:
    mov   edx, edi
    mov   eax, esi
    sub   edx, esi
    sub   eax, edi
    cmp   edi, esi
    cmovg eax, edx  ; "mov if edi > esi"
    ret
```


分支消除 是一个重要的话题，我们将在下一章的大部分内容中讨论细节
