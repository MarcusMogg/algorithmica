
学习汇编语言的主要好处不是使用它写代码，而是了解编译代码执行过程中发生的事情，及其性能影响。

在极少数情况下，我们确实需要切换到手写汇编以获得最大性能，但大多数情况下，编译器能够自行生成接近最优的代码。当他们做不到时，通常是因为程序员对问题的了解比从源代码中推断出来的要多，但未能将这些额外的信息传达给编译器。

在本章中，我们将讨论让编译器完全按照我们想要的方式进行操作，以及收集可以指导进一步优化的有用信息。
