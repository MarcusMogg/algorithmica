
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
### 指令和寄存器


处于历史原因，汇编语言中的指令助记符非常简洁。当时人们习惯于手写汇编并反复编写同一组通用指令时，少输入一个字符就远离精神错乱一步。

For example, `mov` is for "store/load a word," `inc` is for "increment by 1," `mul` is for "multiply," and `idiv` is for "integer division." You can look up the description of an instruction by its name in [one of x86 references](https://www.felixcloutier.com/x86/), but most instructions do what you'd think they do.

Most instructions write their result into the first operand, which can also be involved in the computation like in the `add eax, [rdi]` example we saw before. Operands can be either registers, constant values, or memory locations.

**Registers** are named `rax`, `rbx`, `rcx`, `rdx`, `rdi`, `rsi`, `rbp`, `rsp`, and `r8`-`r15` for a total of 16 of them. The "letter" ones are named like that for historical reasons: `rax` is "accumulator," `rcx` is "counter," `rdx` is "data" and so on — but, of course, they don't have to be used only for that.

There are also 32-, 16-bit and 8-bit registers that have similar names (`rax` → `eax` → `ax` → `al`). They are not fully separate but *aliased*: the lowest 32 bits of `rax` are `eax`, the lowest 16 bits of `eax` are `ax`, and so on. This is made to save die space while maintaining compatibility, and it is also the reason why basic type casts in compiled programming languages are usually free. 

These are just the *general-purpose* registers that you can, with [some exceptions](../functions), use however you like in most instructions. There is also a separate set of registers for [floating-point arithmetic](/hpc/arithmetic/float), a bunch of very wide registers used in [vector extensions](/hpc/simd), and a few special ones that are needed for [control flow](../loops), but we'll get there in time.

**Constants** are just integer or floating-point values: `42`, `0x2a`, `3.14`, `6.02e23`. They are more commonly called *immediate values* because they are embedded right into the machine code. Because it may considerably increase the complexity of the instruction encoding, some instructions don't support immediate values or allow just a fixed subset of them. In some cases, you have to load a constant value into a register and then use it instead of an immediate value.

Apart from numeric values, there are also string constants such as `hello` or `world\n` with their own little subset of operations, but that is a somewhat obscure corner of the assembly language that we are not going to explore here.

### Moving Data

Some instructions may have the same mnemonic, but have different operand types, in which case they are considered distinct instructions as they may perform slightly different operations and take different times to execute. The `mov` instruction is a vivid example of that, as it comes in around 20 different forms, all related to moving data: either between the memory and registers or just between two registers. Despite the name, it doesn't *move* a value into a register, but *copies* it, preserving the original.

When used to copy data between two registers, the `mov` instruction instead performs *register renaming* internally — informs the CPU that the value referred by register X is actually stored in register Y — without causing any additional delay except for maybe reading and decoding the instruction itself. For the same reason, the `xchg` instruction that swaps two registers also doesn't cost anything.

As we've seen above with the fused `add`, you don't have to use `mov` for every memory operation: some arithmetic instructions conveniently support memory locations as operands.

<!--

Some operations are fused like `add r m` or `inc m` (this is one of the rare instructions that doesn't use any register values as operands).

When address is used,

Mirroring

-->

### Addressing Modes

Memory addressing is done with the `[]` operator, but it can do more than just reinterpret a value stored in a register as a memory location. The address operand takes up to 4 parameters presented in the syntax:

```
SIZE PTR [base + index * scale + displacement]
```

where `displacement` needs to be an integer constant and `scale` can be either 2, 4, or 8. What it does is calculate the pointer `base + index * scale + displacement` and dereferences it.

<!-- You can use them in any order: the assembler will figure it out. -->

Using complex addressing is [at most one cycle slower](/hpc/cpu-cache/pointers) than dereferencing a pointer directly, and it can be useful when you have, for example, an array of structures and want to load a specific field of its $i$-th element.

Addressing operator needs to be prefixed with a size specifier for how many bits of data are needed:

- `BYTE` for 8 bits
- `WORD` for 16 bits
- `DWORD` for 32 bits
- `QWORD` for 64 bits

There is also a more rare `TBYTE` for [80 bits](/hpc/arithmetic/float), and `XMMWORD`, `YMMWORD`, and `ZMMWORD` for [128, 256, and 512 bits](/hpc/simd) respectively. All these types don't have to be written in uppercase, but this is how most compilers emit them.

The address computation is often useful by itself: the `lea` ("load effective address") instruction calculates the memory address of the operand and stores it in a register in one cycle, without doing any actual memory operations. While its intended use is for actually computing memory addresses, it is also often used as an arithmetic trick that would otherwise involve 1 multiplication and 2 additions — for example, you can multiply by 3, 5, and 9 with it.

It also frequently serves as a replacement for `add` because it doesn't need a separate `mov` instruction if you need to move the result somewhere else: `add` only works in the two-register `a += b` mode, while `lea` lets you do `a = b + c` (or even `a = b + c + d` if one of them is a constant).

### Alternative Syntax

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
