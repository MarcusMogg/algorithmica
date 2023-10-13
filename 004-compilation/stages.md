
在学习编译器优化之前，让我们在本章简要回顾一下 ”big picture“。忽略掉无聊的阶段，C代码转换为可执行文件有 4个阶段：

1. **预处理**，预处理器展开宏, 从头文件拉取内容, 然后从源代码中移除注释: `gcc -E source.c` (输出预处理之后的内容到 stdout)
2. **编译**， 解析源代码，检查语法错误，转换为中间表示，执行优化，并最终输出汇编语言: `gcc -S file.c` (输出 `.s` 文件)
3. **汇编** ，将汇编代码转换为机器码, 除了像`printf`这种外部调用被替换为占位符: `gcc -c file.c` (输出 `.o` 文件, 称为 *object file*)
4. **链接** 最终将函数调用替换为真实的地址, 产出可执行产物: `gcc -o binary file.c`


每个阶段都可能可以改善 程序性能。

## Interprocedural Optimization 过程优化


我们有最后一个阶段，链接，因为 逐个文件编译程序，然后将这些文件链接在一起既容易又快捷——这样你就可以并行执行此操作，也可以缓存中间结果。

它还提供了将代码作为库分发的功能，这些库可以是静态*static*的，也可以是动态/共享*shared*的：

- *Static* 静态库只是预编译对象文件的集合，编译器将这些对象文件与其他源合并以生成单个可执行文件，就像通常一样。
- *Dynamic* or *shared* 动态库或共享库是预编译的可执行文件，它具有 有关其可调用对象位置的其他元信息，这些元信息在运行时解析。顾名思义，这允许在多个程序之间共享编译的二进制文件。

使用静态库的主要优势是你可以执行多种 过程间优化 *interprocedural optimizations* ，这需要除了函数签名之外更多的上下文信息，比如函数inline、死代码消除。你可以加上  `-static` 选项 来强制 链接器（只）使用 静态库。

这个过程被称为链接时优化（*link-time optimization LTO*），现代编译器会在 对象文件（obj） 中存储 某种形式的中间表示（*intermediate representation*），这使得编译器可以对整个程序进行某些优化。这也使得在同一个程序中使用不同的编译的编译语言成为可能，如果编译器使用相同 的中间表示，甚至可以跨语言进行优化。

LTO是一个相对较新的功能（gcc大概2014年才有），而且离完美还有很远。在c/c++ 中，确保没有任何优化因为分离编译丢失的方式是 使用 *header-only library*。 就像名字一样，它们是包含所有函数的完整定义的头文件，因此只需包含它们，编译器就可以访问所有可能的优化。尽管每次都必须从头开始重新编译库代码，但此方法保留完全控制并确保不会丢失任何性能。

## Inspecting the Output 检查输出

检查每个阶段的输出可以对程序中发生的情况产生有用的见解。

You can get assembly from the source by passing the `-S` flag to the compiler, which will then generate a human-readable `*.s` file. If you pass `-fverbose-asm`, this file will also contain compiler comments about source code line numbers and some info about variables being used. If it is just a little snippet and you are feeling lazy, you can use [Compiler Explorer](https://godbolt.org/), which is a very handy online tool that converts source code to assembly, highlights logical asm blocks by color, includes a small x86 instruction set reference, and also has a large selection of other compilers, targets, and languages.

Apart from the assembly, the other most helpful level of abstraction is the intermediate representation on which compilers perform optimizations. The IR defines the flow of computation itself and is much less dependent on architecture features like the number of registers or a particular instruction set. It is often useful to inspect these to get insight into how the compiler *sees* your program, but this is a bit out of the scope of this book.

在本章中，我们将主要使用 GCC，但在必要时也会尝试为 Clang 复制示例。这两个编译器在很大程度上相互兼容，仅在一些优化标志和次要语法细节上有所不同。