
作为一个软件工程师，我们当然喜欢构建和使用抽象

想象一下，加载URL的时候会发生多少事。你在键盘上输入内容； OS通过某种方式检测到键盘按下，并发送给浏览器；浏览器建解析URL，并通过OS发送网络请求；然后是DNS，路由，TCP，HTTP和所有其他OSI层；浏览器解析HTML；JavaScript开始工作 ；页面的某些表示形式被送到GPU进行渲染；图片帧被送到显示器…… 这些步骤中的每一步可能涉及 在进程中做数十件具体的事

抽象帮助我们将所有这些复杂度降低到一个接口中，接口描述了某个具体模块在不涉及具体实现的情况下能做什么。这提供了两个好处：

- 从事高层级模块的工程师 只需要知道底层级的接口
- 工程师可以随意优化和重构当前模块，只需要最后编译产物符合契约

硬件工程师同样喜欢抽象。CPU的一个抽象被称为 指令集架构（ *instruction set architecture* ISA），它描述了在程序员 的视角下 CPU应该如何工作。和软件接口类似，它使计算机工程师能够改进现有的CPU设计，同时也让用户（程序员）相信以前可以工作的东西不会在新芯片上挂掉。

An ISA essentially defines how the hardware should interpret the machine language. Apart from instructions and their binary encodings, an ISA also defines the counts, sizes, and purposes of registers, the memory model, and the input/output model. Similar to software interfaces, ISAs can be extended too: in fact, they are often updated, mostly in a backward-compatible way, to add new and more specialized instructions that can improve performance.

ISA 定义了硬件应该如何解释机器码。
### RISC vs CISC

Historically, there have been many competing ISAs in use. But unlike [character encodings and instant messaging protocols](https://xkcd.com/927/), developing and maintaining a completely separate ISA is costly, so mainstream CPU designs ended up converging to one of the two families:

- **Arm** chips, which are used in almost all mobile devices, as well as other computer-like devices such as TVs, smart fridges, microwaves, [car autopilots](https://en.wikipedia.org/wiki/Tesla_Autopilot), and so on. They are designed by a British company of the same name, as well as a number of electronics manufacturers including Apple and Samsung.
- **x86**[^x86] chips, which are used in almost all servers and desktops, with a few notable exceptions such as Apple's M1 MacBooks, AWS's Graviton processors, and the current [world's fastest supercomputer](https://en.wikipedia.org/wiki/Fugaku_(supercomputer)), all of which use Arm-based CPUs. They are designed by a duopoly of Intel and AMD.

[^x86]: Modern 64-bit versions of x86 are known as "AMD64," "Intel 64," or by the more vendor-neutral names of "x86-64" or just "x64." A similar 64-bit extension of Arm is called "AArch64" or "ARM64." In this book, we will just use plain "x86" and "Arm" implying the 64-bit versions.

The main difference between them is that of architectural complexity, which is more of a design philosophy rather than some strictly defined property:

- Arm CPUs are *reduced* instruction set computers (RISC). They improve performance by keeping the instruction set small and highly optimized, although some less common operations have to be implemented with subroutines involving several instructions.
- x86 CPUs are *complex* instruction set computers (CISC). They improve performance by adding many specialized instructions, some of which may only be rarely used in practical programs.

The main advantage of RISC designs is that they result in simpler and smaller chips, which projects to lower manufacturing costs and power usage. It's not surprising that the market segmented itself with Arm dominating battery-powered, general-purpose devices, and leaving the complex neural network and Galois field calculations to server-grade, highly-specialized x86s.

<!--

The two architectures are functionally similar, both sharing concepts such as pipelines, execution ports, and SIMD instructions, but since most readers are interested in optimizing applications for mainstream servers and desktops, we will mainly focus on x86 in this book.

-->
