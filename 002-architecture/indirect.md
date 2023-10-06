
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

有许多方法来实现这种行为，C++ 使用 虚函数表 *virtual method table*


对于 的所有具体实现，编译器会填充它们的所有 `Animal` 方法（即它们的指令序列），以便它们对所有类具有完全相同的长度（通过在后面 `ret` 插入一些填充指令），然后将它们按顺序写入指令存储器中的某个位置。 然后，它将运行时类型信息字段添加到结构（即，添加到其所有实例）中，该字段实质上只是内存区域中指向类虚拟方法的正确实现的偏移量。

在虚函数调用中，先从 类实例中读取偏移自动，然后进行一个普通的函数调用（每个派生类的所有虚方法都具有完全相同的偏移量这一事实)。

当然，这会造成额外开销：

- 由于与分支错误预测 相同的管道 刷新原因，可能需要多花费 15 个周期左右
- 编译器没法inline
- 类大小多2个字节或者更多（取决于实现）
- 二进制大小略微增加.

处于这些原因，在性能敏感应用中，运行时多态应该避免使用
