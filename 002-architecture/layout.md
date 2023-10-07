
计算机工程师 精神上 喜欢把 CPU 流水线划分为两部分： 前端，从内存读取和解码指令；后端，指令调度和最终执行。典型的，性能瓶颈在于执行部分，出于这个原因，本书中的大部分工作 围绕后端优化

但是有时相反，前端填充指令的速度不过快，使得后端不饱和。这可能有多种原因，所有这些最终都需要 在某种程度上 改变机器码在内存的布局，比如 删除无用代码，交互if 分支，甚至是改变 函数声明顺序

## CPU Front-End 前端

在机器码转换成指令，CPU理解程序员想要做什么之前，有两个重要的阶段需要我们关注： *fetch* 和 *decode*.

在 **fetch**阶段，CPU简单的从内存中加载固定大小的块，包含一些 二进制编码的指令和数字。在 x64上块大小典型是 32 byte，尽管这在不同机器上各不相同。一个重要的差别是 块必须对齐： 块地址必须是 大小的倍数。

接下来是 **decode** 阶段：CPU查看二进制块，丢弃指令指针之前的内容，然后把剩余的部分 分割为 指令。机器指令编码 使用不同大小的byte：一些简单常见的指令 花费1byte，比如 `inc rax`,  而一些晦涩的指令 带上编码的常数和 前缀可以达到 15 。所以，从一个 32byte 大小的块可以 解码出不同数量的指令，但不超过一个固定的 机器相关的限制 叫做 解码宽度*decode width*。 在我的CPU（ [Zen 2](https://en.wikichip.org/wiki/amd/microarchitectures/zen_2)）上是4，这意味着 每个周期，最多只有4个指令可以被解码并发送到下一阶段。

The stages work in a pipelined fashion: if the CPU can tell (or [predict](/hpc/pipelining/branching/)) which instruction block it needs next, then the fetch stage doesn't wait for the last instruction in the current block to be decoded and loads the next one right away.

这个阶段以流水线的形式工作： 如果CPU 可以 告诉（或着预测）下一个指令块需要什么，那么 fetch阶段可以 不用等待上一条指令完成 就去加载和 解码下一条。

## Code Alignment 代码对齐

在其他条件相同的情况下，编译器通常会选择 机器码更短的指令，因为这样 一个 32B 的 fetch 块可以包含更多指令，而且可以减少二进制大小。但有时相反，因为 fetch的 指令块必须对齐。

Imagine that you need to execute an instruction sequence that starts on the last byte of a 32B-aligned block. You may be able to execute the first instruction without additional delay, but for the subsequent ones, you have to wait for one additional cycle to do another instruction fetch. If the code block was aligned on a 32B boundary, then up to 4 instructions could be decoded and then executed concurrently (unless they are extra long or interdependent).

Having this in mind, compilers often do a seemingly harmful optimization: they sometimes prefer instructions with longer machine codes, and even insert dummy instructions that do nothing[^nop] in order to get key jump locations aligned on a suitable power-of-two boundary.

[^nop]: Such instructions are called no-op, or NOP instructions. On x86, the "official way" of doing nothing is `xchg rax, rax` (swap a register with itself): the CPU recognizes it and doesn't spend extra cycles executing it, except for the decode stage. The `nop` shorthand maps to the same machine code.

In GCC, you can use `-falign-labels=n` flag to specify a particular alignment policy, [replacing](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html) `-labels` with `-function`, `-loops`, or `-jumps` if you want to be more selective. On `-O2` and `-O3` levels of optimization, it is enabled by default — without setting a particular alignment, in which case it uses a (usually reasonable) machine-dependent default value.

<!-- Having to decode a bunch of extra NOPs is usually not a problem. -->

### Instruction Cache

The instructions are stored and fetched using largely the same [memory system](/hpc/cpu-cache) as for the data, except maybe the lower layers of cache are replaced with a separate *instruction cache* (because you wouldn't want a random data read to kick out the code that processes it).

The instruction cache is crucial in situations when you either:

- don't know what instructions you are going to execute next, and need to fetch the next block with [low latency](/hpc/cpu-cache/latency),
- or are executing a long sequence of verbose-but-quick-to-process instructions, and need [high bandwidth](/hpc/cpu-cache/bandwidth).

The memory system can therefore become the bottleneck for programs with large machine code. This consideration limits the applicability of the optimization techniques we've previously discussed:

- [Inlining functions](../functions) is not always optimal, because it reduces code sharing and increases the binary size, requiring more instruction cache.
- [Unrolling loops](../loops) is only beneficial up to some extent, even if the number of iterations is known during compile time: at some point, the CPU would have to fetch both instructions and data from the main memory, in which case it will likely be bottlenecked by the memory bandwidth.
- Huge [code alignments](#code-alignment) increase the binary size, again requiring more instruction cache. Spending one more cycle on fetch is a minor penalty compared to missing the cache and waiting for the instructions to be fetched from the main memory.

Another aspect is that placing frequently used instruction sequences on the same [cache lines](/hpc/cpu-cache/cache-lines) and [memory pages](/hpc/cpu-cache/paging) improves [cache locality](/hpc/external-memory/locality). To improve instruction cache utilization, you should  group hot code with hot code and cold code with cold code, and remove dead (unused) code if possible. If you want to explore this idea further, check out Facebook's [Binary Optimization and Layout Tool](https://engineering.fb.com/2018/06/19/data-infrastructure/accelerate-large-scale-applications-with-bolt/), which was recently [merged](https://github.com/llvm/llvm-project/commit/4c106cfdf7cf7eec861ad3983a3dd9a9e8f3a8ae) into LLVM.

### Unequal Branches

Suppose that for some reason you need a helper function that calculates the length of an integer interval. It takes two arguments, $x$ and $y$, but for convenience, it may correspond to either $[x, y]$ or $[y, x]$, depending on which one is non-empty. In plain C, you would probably write something like this:

```c++
int length(int x, int y) {
    if (x > y)
        return x - y;
    else
        return y - x;
}
```

In x86 assembly, there is a lot more variability to how you can implement it, noticeably impacting performance. Let's start with trying to map this code directly into assembly:

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

While the initial C code seems very symmetrical, the assembly version isn't. This results in an interesting quirk that one branch can be executed slightly faster than the other: if `x > y`, then the CPU can just execute the 5 instructions between `cmp` and `ret`, which, if the function is aligned, are all going to be fetched in one go; while in case of `x <= y`, two more jumps are required.

It may be reasonable to assume that the `x > y` case is *unlikely* (why would anyone calculate the length of an inverted interval?), more like an exception that mostly never happens. We can detect this case, and simply swap `x` and `y`:

```c++
int length(int x, int y) {
    if (x > y)
        swap(x, y);
    return y - x;
}
```

The assembly would go like this, as it typically does for the if-without-else patterns:

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

The total instruction length is 6 now, down from 8. But it is still not quite optimized for our assumed case: if we think that `x > y` never happens, then we are wasteful when loading the `xchg edi, esi` instruction that is never going to be executed. We can solve this by moving it outside the normal execution path:

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

This technique is quite handy when handling exceptions cases in general, and in high-level code, you can give the compiler a [hint](/hpc/compilation/situational) that a certain branch is more likely than the other:

```c++
int length(int x, int y) {
    if (x > y) [[unlikely]]
        swap(x, y);
    return y - x;
}
```

This optimization is only beneficial when you know that a branch is very rarely taken. When this is not the case, there are [other aspects](/hpc/pipelining/hazards) more important than the code layout, that compel compilers to avoid any branching at all — in this case by replacing it with a special "conditional move" instruction, roughly corresponding to the ternary expression `(x > y ? y - x : x - y)` or calling `abs(x - y)`:

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

Eliminating branches is an important topic, and we will spend [much of the next chapter](/hpc/pipelining/branching) discussing it in more detail.

<!--

This architecture peculiarity

When you have branches in your code, there is a variability in how you can place their instruction sequences in the memory — and surprisingly, .

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

Granted that `x > y` never or almost never happens, the branchy variant will be 2 instructions shorter.

https://godbolt.org/z/bb3a3ahdE

(The compiler can't optimize it because it's technically [not allowed to](/hpc/compilation/contracts): despite `y - x` being valid, `x - y` could over/underflow, causing undefined behavior. Although fully correct, I guess the compiler just doesn't date executing it.)

We will spend [much of the next chapter](/hpc/pipelining/branching) discussing it in more detail.

You don't have to decode the things you are not going to execute anyway.

In general, you want to, and put rarely executed code away — even in the case of if-without-else patterns.

-->
