
在汇编期间，所有的标签会被转换为地址（直接或者间接），然后编码到跳转指令

你也可以跳转到存储在寄存器中的非 常量地址， 称为 *计算跳转*

```nasm
jmp rax
```

## Multiway Branch

如果你忘了`switch` 是干什么的，这里有一个小程序来计算美国毕业体系下的GPA

```cpp
switch (grade) {
    case 'A':
        return 4.0;
        break;
    case 'B':
        return 3.0;
        break;
    case 'C':
        return 2.0;
        break;
    case 'D':
        return 1.0;
        break;
    case 'E':
    case 'F':
        return 0.0;
        break;
    default:
        return NAN;
}
```

通常来说，switch 相当于 一系列 "if, else if, else if, else if…" ,因此很多语言没有switch。尽管如此，这样的控制流结构在实现状态机时非常重要。通常就是一个 `while (true)` 循环 加一个 `switch (state)` 

当我们可以控制变量的取值范围时，我们可以使用下面的技巧，利用计算跳转。我们可以创建一个包含所有可能跳转地址条件表 而不是使用n个条件分支，然后只需要使用 取值在 $[0, n)$ 的state 变量进行索引
 
但数据密集时（不一定是严格排列，但必须值得在表中放一个空白字段），编译器可以使用这种技术。他也可以通过 计算 goto 显示实现：


```cpp
void weather_in_russia(int season) {
    static const void* table[] = {&&winter, &&spring, &&summer, &&fall};
    goto *table[season];

    winter:
        printf("Freezing\n");
        return;
    spring:
        printf("Dirty\n");
        return;
    summer:
        printf("Dry\n");
        return;
    fall:
        printf("Windy\n");
        return;
}
```

Switch-based code is not always straightforward for compilers to optimize, so in the context of state machines, `goto` statements are often used directly. The I/O-related part of `glibc` is full of examples.

Switch-based 对编译器来说 不一定可以简单的优化，所以在 状态机上下文中，`goto` 也会直接使用，`glibc`的 I/O 相关部分是一个例子
## Dynamic Dispatch 动态分发

间接分支 在运行时多态的实现中也很有用


```cpp
struct Animal {
    virtual void speak() { printf("<abstract animal sound>\n");}
};

struct Dog {
    void speak() override { printf("Bark\n"); }
};

struct Cat {
    void speak() override { printf("Meow\n"); }
};
```

我们想要创建一个动物，并且在事先不知道其类型的情况下调用它的 `.speak()` 方法，该方法应该以某种方式调用正确的实现：

```c++
Dog sparkles;
Cat mittens;

Animal *catdog = (rand() & 1) ? &sparkles : &mittens;
catdog->speak();
```

There are many ways to implement this behavior, but C++ does it using a *virtual method table*.

有许多方法来实现这种行为，C++ 使用 虚函数表

For all concrete implementations of `Animal`, compiler pads all their methods (that is, their instruction sequences) so that they have the exact same length for all classes (by inserting some [filler instructions](../layout) after `ret`) and then just writes them sequentially somewhere in the instruction memory. Then it adds a *run-time type information* field to the structure (that is, to all its instances), which is essentially just the offset in the memory region that points to the right implementation of the virtual methods of the class.

With a virtual method call, that offset field is fetched from the instance of a structure and a normal function call is made with it, using the fact that all methods and other fields of every derived class have exactly the same offsets.

Of course, this adds some overhead:

- You may need to spend another 15 cycles or so for the same pipeline flushing reasons as for [branch misprediction](/hpc/pipelining).
- The compiler most likely won't be able to inline the function call itself.
- Class size increases by a couple of bytes or so (this is implementation-specific).
- The binary size itself increases a little bit.

For these reasons, runtime polymorphism is usually avoided in performance-critical applications.
