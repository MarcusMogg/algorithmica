
机器代码分析器是一种程序，它使用一小段汇编代码，并使用编译器可用的信息模拟其在特定微架构上的执行，并输出整个块的延迟和吞吐量，以及CPU内各种资源的周期执行利用率。

## 使用 `llvm-mca`


有很多机器码分析工具，我个人喜欢 `llvm-mca`，你可能通过包管理器和 `clang`一起安装。你可以使用在线工具[UICA](https://uica.uops.info) 或者在  [Compiler Explorer](https://godbolt.org/) 中 选中"Analysis" 作为语言

`llvm-mca` 的作用是多次运行给定汇编片段，并计算有关每个指令的资源使用情况的统计信息，这对于找出瓶颈所在非常有用。

将数组总和视为我们的简单示例：

```asm
loop:
    addl (%rax), %edx
    addq $4, %rax
    cmpq %rcx, %rax
    jne	 loop
````

以下是 `llvm-mca` 对Skylake微架构的分析：

```yaml
Iterations:        100
Instructions:      400
Total Cycles:      108
Total uOps:        500

Dispatch Width:    6
uOps Per Cycle:    4.63
IPC:               3.70
Block RThroughput: 0.8
```

首先，它输出有关循环和硬件的通用信息：

- 它“运行”循环 100 次，在 108 个周期内总共执行 400 条指令，这与平均每个周期执行指令 （IPC） $\frac{400}{108} \approx 3.7$ 相同
-  CPU理论上每个周期可以执行6个指令（Dispatch Width）
- 理论上每个循环平均可以在0.8个周期内执行（Block RThroughput）。
- 这里的“uOps”是CPU将每条指令拆分为的微操作（例如，融合的load-add由两个uOp组成）。

然后它继续提供有关每个单独指令的信息：

```yaml
Instruction Info:
[1]: uOps
[2]: Latency
[3]: RThroughput
[4]: MayLoad
[5]: MayStore
[6]: HasSideEffects (U)

[1]    [2]    [3]    [4]    [5]    [6]    Instructions:
 2      6     0.50    *                   addl	(%rax), %edx
 1      1     0.25                        addq	$4, %rax
 1      1     0.25                        cmpq	%rcx, %rax
 1      1     0.50                        jne	-11
```

这里没有不在指令表的信息：

- 每条指令被拆分为多少uOps;
- 每条指令完成需要多少个周期（延迟);
- 每条指令在摊销意义上（倒数吞吐量）需要多少个周期才能完成，考虑到它可以同时执行它的多个副本.

然后它输出可能是最重要的部分 —— 哪些指令在何时何地执行：

```yaml
Resource pressure by instruction:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]    [8]    [9]    Instructions:
 -      -     0.01   0.98   0.50   0.50    -      -     0.01    -     addl (%rax), %edx
 -      -      -      -      -      -      -     0.01   0.99    -     addq $4, %rax
 -      -      -     0.01    -      -      -     0.99    -      -     cmpq %rcx, %rax
 -      -     0.99    -      -      -      -      -     0.01    -     jne  -11
```

由于对执行端口的争用会导致structural hazards，因此端口通常成为面向吞吐量的循环的瓶颈，此图表有助于诊断原因。它不会给你一个周期完美的甘特图，但它给你用于每条指令的执行端口的聚合统计数据，让你找到哪个指令是负载高。