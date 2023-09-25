
作为一个软件工程师，我们当然喜欢构建和使用抽象

想象一下，加载URL的时候会发生多少事。你在键盘上输入内容； OS通过某种方式检测到键盘按下，并发送给浏览器；浏览器建解析URL，并通过OS发送网络请求；然后是DNS，路由，TCP，HTTP和所有其他OSI层；浏览器解析HTML；JavaScript开始工作 ；页面的某些表示形式被送到GPU进行渲染；图片帧被送到显示器…… 这些步骤中的每一步可能涉及 在进程中做数十件具体的事

抽象帮助我们将所有这些复杂度降低到一个接口中，接口描述了某个具体模块在不涉及具体实现的情况下能做什么。这提供了两个好处：

- 从事高层级模块的工程师 只需要知道底层级的接口
- 工程师可以随意优化和重构当前模块，只需要最后编译产物符合契约

硬件工程师同样喜欢抽象。CPU的一个抽象被称为 指令集架构（ *instruction set architecture* ISA），它描述了在程序员 的视角下 CPU应该如何工作。和软件接口类似，它使计算机工程师能够改进现有的CPU设计，同时也让用户（程序员）相信以前可以工作的东西不会在新芯片上挂掉。

ISA 定义了硬件应该如何解释机器码。除了指令集和它们的二进制编码外，ISA还定义了寄存器的 大小、数量、用途，内存模型，输入输出模型。和软件接口类似，ISA同样可以拓展，事实上，它们经常更新，而且大多向后兼容，以添加新的和更专业的指令，从而提高性能。
## RISC vs CISC

历史上，曾经有许多相互竞争的ISA在使用。但是不像字符编码和通信协议，开发和维护完全独立的ISA成本很高，因此主流CPU设计 最后融合到两个系列之一：

- **Arm** 芯片, 用于几乎所有移动设备, 以及其他类似计算机的设备，比如：TVs, 智能冰箱,  [汽车自动驾驶仪](https://en.wikipedia.org/wiki/Tesla_Autopilot)……. 它们由一家同名的英国公司以及包括苹果和三星在内的多家电子制造商设计.
- **x86** 芯片,几乎用于所有服务器和台式机，除了一些值得注意的例外，例如Apple的M1 MacBook, AWS的Graviton处理器以及当前世界上最快的超级计算机，这些都使用基于Arm的CPU。它们是由英特尔和AMD设计的。

[^x86]: Modern 64-bit versions of x86 are known as "AMD64," "Intel 64," or by the more vendor-neutral names of "x86-64" or just "x64." A similar 64-bit extension of Arm is called "AArch64" or "ARM64." In this book, we will just use plain "x86" and "Arm" implying the 64-bit versions.

它们之间的主要区别在于架构复杂性，这更像是一种设计理念，而不是一些严格定义的属性：

- Arm CPUs 是 *reduced* instruction set computers (RISC). 他们通过精简指令集并高度优化来提高性能,  不过一些不太常见的操作需要使用 多个指令来实现.
- x86 CPUs 是 *complex* instruction set computers (CISC). 他们通过添加专用指令来提高性能, 其中一些指令可能很少在实际的程序中使用.

 RISC 设计的优势是 可以产出更小、更简单的芯片, 从而降低制造和使用成本. 市场细分为Arm主导的电池供电的通用设备，并将复杂的神经网络和Galois field 计算留给服务器级、高度专业化的x86。