
从源码或者其汇编开始是一个流行，但不高效的发现性能问题的方式。但性能不满足你的预期时，你可以使用 叫做*profilers* 的专门工具集来快速识别根本问题。

有多种不同的profiler。我喜欢通过类比物理学家和其他自然科学家如何研究小东西来思考它们，根据所需的精度水平选择合适的工具：

- 当物体在微米级时，它们使用光学显微镜。
- 当物体在纳米尺度上并且光不再与它们相互作用时，它们使用电子显微镜。
- 当物体小于此值时（例如，原子的内部），它们会诉诸关于事物如何工作的理论和假设（并使用复杂和间接的实验来测试这些假设）。

类似的，有三种主要的profiling技术，每种有其自己的准则，有不同的应用场景：

- [Instrumentation](instrumentation) 允许你对整个程序或者你感兴趣的部分进行计时。
- [Statistical profiling ](events) 让你深入汇编以及各种重要的硬件事件例如分支预测失败或者缓存失效，
- [Program simulation](mca)  让你深入到单个周期级别，并查看CPU在执行小的汇编片段时每个周期发生的情况。

实用的算法设计也可以被认为是一个经验领域，我们在很大程度上依赖于相同的实验方法，尽管这并不是因为我们不知道自然界的一些基本秘密，而主要是因为现代计算机太复杂而无法分析——此外，我们——普通的软件工程师，由于硬件公司的知识产权保护而无法知道一些细节（事实上， 考虑到最准确的x86指令表是[逆向工程]((https://arxiv.org/pdf/1810.04610.pdf),)的，有理由相信英特尔自己不知道这些细节）。

在本章中，我们将研究这三种关键的分析方法，以及一些经过时间考验的实践，用于管理涉及性能评估的计算实验。