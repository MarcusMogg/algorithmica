
CPU 由机器语言控制。机器语言就是二进制编码的指令流，指定：
- 指令码 ( *opcode*),
- 操作数,
- 存储结果的位置（如果有）.

汇编是一种更人性化的机器语言“翻译”，使用助记符（Mnemonics）来代替和表示特定机器语言的指令，使用符号名称（Symbolic names）标记成为存储器地址以及其它的存储位置

下面是 Arm 汇编来表示 (`*c = *a + *b`) :

```nasm
; *a = x0, *b = x1, *c = x2
ldr w0, [x0]    ; load 4 bytes from wherever x0 points into w0
ldr w1, [x1]    ; load 4 bytes from wherever x1 points into w1
add w0, w0, w1  ; add w0 with w1 and save the result to w0
str w0, [x2]    ; write contents of w0 to wherever x2 points
```

 x86 汇编:

```nasm
; *a = rsi, *b = rdi, *c = rdx 
mov eax, DWORD PTR [rsi]  ; load 4 bytes from wherever rsi points into eax
add eax, DWORD PTR [rdi]  ; add whatever is stored at rdi to eax
mov DWORD PTR [rdx], eax  ; write contents of eax to wherever rdx points
```

汇编非常简单，和高级语言比 没有复杂的语法结构。从上面的例子可以观察到：

-  程序是一系列指令，每个指令都是 指令名称，后跟可变数量的操作数。
-  `[reg]` 符号用于 "解引用" 存储在寄存器的指针, 在x86你需要在前面指定大小信息 (`DWORD`表示 32 bit).
-  `;`用于注释

汇编是一种非常简约的语言。它尽可能的反应机器语言，直到 和机器码之间一一对应。事实上，你可以使用 反汇编程序 将任意已编译的程序变为 汇编形式，不过像注释之类的不必要内容不会被保留。

请注意，上面两份代码片段不仅是语法不同，两者都是编译器优化后产生的代码，x86使用3条指令但是arm 使用4条。`add eax, [rdi]` 指令是 *融合指令fused instruction* ，一次性执行加载和 加法，这是使用CISC 提供的好处

由于架构之间的差异远不止这个，从这里开始和本书的其余部分，我们将只提供 x86 的示例，这可能是我们大多数读者会优化的内容，尽管许多引入的概念将与架构无关。
## 指令和寄存器


处于历史原因，汇编语言中的指令助记符非常简洁。当时人们习惯于手写汇编并反复编写同一组通用指令时，少输入一个字符就远离精神错乱一步。

例如，`mov` 用于加载/存储字，`inc`是加一，`mul` 是 乘法，`idiv` 是整数除法。你可以在[x86 手册](https://www.felixcloutier.com/x86/)中用名字查找一个指令的说明，但它们大部分执行的操作就和你想的一样

大部分指令将结果写回到第一个操作数，就像我们上面的例子`add eax, [rdi]`一样，他也可以参与运算。操作数可以是 寄存器、常数或者内存地址

寄存器 命名为 `rax`, `rbx`, `rcx`, `rdx`, `rdi`, `rsi`, `rbp`, `rsp`,  `r8`-`r15`   16个。字母命名是出于历史原因：  `rax` 累加器 ，`rcx` 是计数器，“ `rdx` 是数据 等等 。 但是，当然，没有限制只能这样用。

有 32bit、16bit、8bit的寄存器，其名字类似 (`rax` → `eax` → `ax` → `al`)，它们不是完全独立的而是相互重叠的：例如 `rax` 的低16位是 `eax`，`eax` 的低8位是 `ax`，依次类推。这样做是为了节省芯片空间，同时保证兼容性。这也是为什么在编译型语言中，基本类型转换是免费的。

这只是通用寄存器，除了一些例外，你可以随意使用大部分寄存器。也有一些独立的寄存器用于浮点计算，一些用于向量拓展的宽寄存器，一些特殊的寄存器用于控制流。我们到时候会介绍它们。

常量只能是 整数或 浮点数 `42`, `0x2a`, `3.14`, `6.02e23`。它们通常被叫作立即数（*immediate values*），因为它们被直接编码到机器码中。由于它可能会大大增加指令编码的复杂度，一些指令 不支持立即数，或者只能有限使用子集。某些场景下，你必须先把常量加载到寄存器中。

## Moving Data


某些指令有相同的助记符，但是不同的操作数类型。这种情况下，它们被认为是不同的指令，因为它们可能执行略微不同的操作 或者花费不同的时间来执行。

`mov`指令是一个鲜明的例子，他大概有20种不同的形式，全是关于 移动数据：可能是在内存和寄存器之间，也可以是在寄存器和寄存器之间。

尽管名字是 移动 ,但事实上 它是原样拷贝数据。

当用于在两个寄存器之间复制数据时， `mov` 指令改为在内部执行寄存器重命名 - 通知CPU寄存器X引用的值实际上存储在寄存器Y中 - 除了可能读取和解码指令本身之外，不会造成任何额外的延迟。出于同样的原因，交换两个寄存器的 `xchg` 指令也不需要任何费用。

正如我们在上面看到的 `add` 融合指令 ，您不必为 每个内存操作 使用`mov`：一些算术指令支持内存位置作为操作数。

## 寻址模式

内存寻址 由 `[]` 操作符完成， 它做的不仅仅是把 存储在寄存器中的值解释为内存地址。地址操作数最多接受下面表达式中的四个参数。

```
SIZE PTR [base + index * scale + displacement]
```

 `displacement` 需要是整数常量
 `scale` 可以是 2, 4, or 8.
  他执行的操作是计算 `base + index * scale + displacement` 然后接引用

使用复杂的寻址通常会比直接解引用指针最多慢一个 时钟周期，但你需要时会比较有用，例如 ，加载一个 struct 数组的第i个元素的某个字段。

寻址操作符 需要加一个前缀指示数据需要多少bit

- `BYTE` for 8 bits
- `WORD` for 16 bits
- `DWORD` for 32 bits
- `QWORD` for 64 bits

也有比较少见的 `TBYTE` for 80 bit,   `XMMWORD`, `YMMWORD`, and `ZMMWORD` for 128, 256,  and 512 bits . 这些类型不必大写，但是通常编译器这样写。

地址计算本身通常很有用： `lea` （“加载有效地址”）指令计算操作数的内存地址，并在一个周期内将其存储在寄存器中，而无需执行任何实际的内存操作。虽然它的预期用途是实际计算内存地址，但它也经常被用作算术技巧，否则将涉及 1 次乘法和 2 次加法——例如，你可以用它乘以 3、5 和 9。

它也经常作为 `add` 的替代品 ，因为如果需要将结果移动到其他地方，它不需要单独的 `mov` 指令： `add` 仅在双寄存器 `a += b` 模式下工作，而 `lea` 允许这样做 `a = b + c` （或者 `a = b + c + d` 其中一个是常量）。

## 选择语法

There are actually multiple *assemblers* (the programs that produce machine code from assembly) with different assembly languages, but only two x86 syntaxes are widely used now. They are commonly called after the two companies that used them and had a dominant influence on programming during that era:

- The *AT&T syntax*, used by default by all Linux tools.
- The *Intel syntax*, used by default, well, by Intel.

These syntaxes are also sometimes called *GAS* and *NASM* respectively, by the names of the two primary assemblers that use them (*GNU Assembler* and *Netwide Assembler*).

We used Intel syntax in this chapter and will continue to preferably use it for the rest of the book. For comparison, here is how the same `*c = *a + *b` example looks like in AT&T asm:

```asm
movl (%rsi), %eax
addl (%rdi), %eax
movl %eax, (%rdx)
```

The key differences can be summarized as follows:

1. The *last* operand is used to specify the destination.
2. Registers and constants need to be prefixed by `%` and `$` respectively (e.g., `addl $1, %rdx` increments `rdx`).
3. Memory addressing looks like this: `displacement(%base, %index, scale)`.
4. Both `;` and `#` can be used for line comments, and also `/* */` can be used for block comments.

And, most importantly, in AT&T syntax, the instruction names need to be "suffixed" (`addq`, `movl`, `cmpq`, etc.) to specify what size operands are being manipulated:

- `b` = byte (8 bit)
- `w` = word (16 bit)
- `l` = long (32 bit integer or 64-bit floating-point)
- `q` = quad (64 bit)
- `s` = single (32-bit floating-point)
- `t` = ten bytes (80-bit floating-point)

In Intel syntax, this information is inferred from operands (which is why you also need to specify sizes of pointers).

Most tools that produce or consume x86 assembly can do so in both syntaxes, so you can just pick the one you like more and don't worry.
