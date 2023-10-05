
To "call a function" in assembly, you need to [jump](../loops) to its beginning and then jump back. But then two important problems arise:

在汇编中实现 "函数调用"，需要先跳转到函数开头，函数结束时跳回。但这会产生两个问题：

1. 如果调用方将数据存储在与被调方相同的寄存器中，该怎么办？
2. 跳回哪里?

Both of these concerns can be solved by having a dedicated location in memory where we can write all the information we need to return from the function before calling it. This location is called *the stack*.

这两个问题都可以使用内存中一个专用位置来解决，我们可以 在调用函数之前 向其中写入 所有需要返回的信息。这个位置叫做**栈**

## 栈

硬件栈 和 软件(蒜法)里的栈工作原理类似，同样的由两个指针实现：

- 基指针 标记栈底，通常存储在寄存器 `rbp`.
- 栈指针 标记栈最后一个元素（栈底），通常存储在寄存器`rsp`.

当你调用一个函数时，你需要把所有本地变量 `push` 到栈上（当然在其他情况下也可以这样做，比如 寄存器不够用时），然后`push` 当前指令指针，接着跳转到函数开始。

当退出函数时，查找栈底元素，然后跳转到那里，接着小心地 将所有存储在栈中的变量读回其寄存器。

你可以使用通常的内存操作指令和 跳转来实现所有操作，但是因为使用频繁，这里有四个指令专门用于实现：

- `push` 将数据写入栈指针的位置，然后 栈指针 减少 
- `pop`  读栈顶数据，然后 栈指针 增加
- `call` 将下一条指令写到栈里 ，然后跳转到标签位置.
- `ret` 读取栈顶的返回位置，然后跳转.

你可以叫他们“语法糖”，因为他们不是真正的硬件指令，他们等价于两条指令的融合产物

```nasm
; "push rax"
sub rsp, 8
mov QWORD PTR[rsp], rax

; "pop rax"
mov rax, QWORD PTR[rsp]
add rsp, 8

; "call func"
push rip ; <- instruction pointer (although accessing it like that is probably illegal)
jmp func

; "ret"
pop  rcx ; <- choose any unused register
jmp rcx
```


`rbp` 和 `rsp`之间的内存区域被称为栈帧 *stack frame*, 这是函数局部变量通常存储的位置。它在程序启动时预先分配好，如果向栈上push 超过栈大小（linux默认8MB）的数据，会遇到栈溢出 *stack overflow*。 因为现代操作系统 在你读写内存地址之前并不会真 的分配内存页，所以你可以指定一个巨大的栈大小，它更像是对可以使用多少栈内存的限制，而不是每个程序必须使用的固定数量。

## 调用约定

编译器和操作系统开发人员提出了如何编写和调用函数的 调用约定[conventions](https://wiki.osdev.org/Calling_Conventions)。这些约定使得一些 重要的软件工程 marvels 成为可能，比如 划分编译单元，重复使用已编译的库，甚至使用不同的语言进行开发

考虑下面的C例子:

```c
int square(int x) {
    return x * x;
}

int distance(int x, int y) {
    return square(x) + square(y);
}
```


契约规定，一个函数需要把他的参数放在`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` （不够用的话剩下的放在栈上），返回值放在`rax`，然后返回。

所以 `square` 函数可以这样简单的实现：

```nasm
square:             ; x = edi, ret = eax
    imul edi, edi
    mov  eax, edi
    ret
```

每次从 `distance`调用时, 需要保留其局部变量:

```nasm
distance:           ; x = rdi/edi, y = rsi/esi, ret = rax/eax
    push rdi
    push rsi
    call square     ; eax = square(x)
    pop  rsi
    pop  rdi

    mov  ebx, eax   ; save x^2
    mov  rdi, rsi   ; move new x=y

    push rdi
    push rsi
    call square     ; eax = square(x=y)
    pop  rsi
    pop  rdi

    add  eax, ebx   ; x^2 + y^2
    ret
```

还有更多的细微差别，但我们不会在这里详细介绍，因为这本书是关于性能的，处理函数调用的最好方法实际上是首先避免进行调用。
## Inline

Moving data to and from the stack creates noticeable overhead for small functions like these. The reason you have to do this is that, in general, you don't know whether the callee is modifying the registers where you store your local variables. But when you have access to the code of `square`, you can solve this problem by stashing the data in registers that you know won't be modified.

```nasm
distance:
    call square
    mov  ebx, eax
    mov  edi, esi
    call square
    add  eax, ebx
    ret
```

This is better, but we are still implicitly accessing stack memory: you need to push and pop the instruction pointer on each function call. In simple cases like this, we can *inline* function calls by stitching the callee's code into the caller and resolving conflicts over registers. In our example:

```nasm
distance:
    imul edi, edi       ; edi = x^2
    imul esi, esi       ; esi = y^2
    add  edi, esi
    mov  eax, edi       ; there is no "add eax, edi, esi", so we need a separate mov
    ret
```

This is fairly close to what optimizing compilers produce out of this snippet — only they use the [lea trick](../assembly) to make the resulting machine code sequence a few bytes smaller:

```nasm
distance:
    imul edi, edi       ; edi = x^2
    imul esi, esi       ; esi = y^2
    lea  eax, [rdi+rsi] ; eax = x^2 + y^2
    ret
```

In situations like these, function inlining is clearly beneficial, and compilers mostly do it [automatically](/hpc/compilation/situational), but there are cases when it's not — and we will talk about them [in a bit](../layout).

### Tail Call Elimination

Inlining is straightforward to do when the callee doesn't make any other function calls, or at least if these calls are not recursive. Let's move on to a more complex example. Consider this recursive computation of a factorial:

```cpp
int factorial(int n) {
    if (n == 0)
        return 1;
    return factorial(n - 1) * n;
}
```

Equivalent assembly:

```nasm
; n = edi, ret = eax
factorial:
    test edi, edi   ; test if a value is zero
    jne  nonzero    ; (the machine code of "cmp rax, 0" would be one byte longer)
    mov  eax, 1     ; return 1
    ret
nonzero:
    push edi        ; save n to use later in multiplication
    sub  edi, 1
    call factorial  ; call f(n - 1)
    pop  edi
    imul eax, edi
    ret
```

If the function is recursive, it is still often possible to make it "call-less" by restructuring it. This is the case when the function is *tail recursive*, that is, it returns right after making a recursive call. Since no actions are required after the call, there is also no need for storing anything on the stack, and a recursive call can be safely replaced with a jump to the beginning — effectively turning the function into a loop.

To make our `factorial` function tail-recursive, we can pass a "current product" argument to it:

```cpp
int factorial(int n, int p = 1) {
    if (n == 0)
        return p;
    return factorial(n - 1, p * n);
}
```

Then this function can be easily folded into a loop:

```nasm
; assuming n > 0
factorial:
    mov  eax, 1
loop:
    imul eax, edi
    sub  edi, 1
    jne  loop
    ret
```

The primary reason why recursion can be slow is that it needs to read and write data to the stack, while iterative and tail-recursive algorithms do not. This concept is very important in functional programming, where there are no loops and all you can use are functions. Without tail call elimination, functional programs would require way more time and memory to execute.
