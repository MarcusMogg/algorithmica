
1960年代超级计算机的主要劣势不是慢，而是体积大、使用复杂，而且价格昂贵，只有超级大国可以负担的起。尺寸是它们如此昂贵的原因：需要大量定制组件，这些组件必须由拥有电气工程高级学位的人员非常仔细地组装，这个过程无法扩大规模以进行大规模生产。

转折点是微芯片的发展，独立、微小、完整的电路。它彻底改变了整个行业，可以说是20世纪最伟大的发明。1965年价值百万、橱柜大小的计算机 在1975年可以放在 4nm × 4mm 的硅片上，你只需要花25美元就可以买到。在接下来的10年里，成本的显著减低开启了家用机的革命。


## 微芯片是如何制作的

微芯片使用被称为 [光刻](https://en.wikipedia.org/wiki/Photolithography)的工艺打印在硅晶体上, 该过程涉及：

1. 生产和切割 [非常纯粹的硅晶体](https://en.wikipedia.org/wiki/Wafer_(electronics)),
2. 用一层 [光子撞击时溶解的物质](https://en.wikipedia.org/wiki/Photoresist)覆盖
3. 使用光子按照指定方式撞击,
4. 对暴露的部分进行 [蚀刻](https://en.wikipedia.org/wiki/Etching_(microfabrication)) ,
5. 去除剩余的光刻胶


![](../img/lithography.png)

考虑光子撞击部分。为此，我们可以使用一个透镜系统，将图案投射到更小的区域上，有效地制作出具有所需属性的微小电路。通过这种方式，1970 年代的光学器件能够在指甲盖大小上安装数千个晶体管，这为微芯片提供了宏计算机所不具备的几个关键优势：

- 更高的时钟速率（受光速限制）
- 扩大生产规模;
-  材料和电力使用量大大降低，单位成本大大降低

除了这些直接的好处之外，光刻技术还为进一步提高性能提供了一条清晰的途径：你可以使镜头更强大，这反过来又会以相对较少的努力创造出更小但功能相同的设备。

## Dennard Scaling

想一想当我们缩小微芯片大小时会发生什么。更小的电路需要更少的物料，更小的晶体管需要更少的时间来切换（以及芯片中的所有其他物理过程），从而可以降低电压并提高时钟速率。

更详细的观察结果,被称为 *Dennard scaling*, 指出 晶体管尺寸每减少 30%

- 晶体管密度加倍 ($0.7^2 \approx 0.5$),
- 时钟速度提高 40% ($\frac{1}{0.7} \approx 1.4$),
- **功率密度**（每单位体积的功率） 不变

Since the per-unit manufacturing cost is a function of area, and the exploitation cost is mostly the cost of power[^power], each new "generation" should have roughly the same total cost, but 40% higher clock and twice as many transistors, which can be promptly used, for example, to add new instructions or increase the word size — to keep up with the same miniaturization happening in memory microchips.

[^power]: The cost of electricity for running a busy server for 2-3 years roughly equals the cost of making the chip itself.

Due to the trade-offs between energy and performance you can make during the design, the fidelity of the fabrication process itself, such as "180nm" or "65nm," directly translating to the density of transistors, became the trademark for CPU efficiency[^fidelity].

[^fidelity]: At some point, when Moore's law started to slow down, chip makers stopped delineating their chips by the size of their components — and it is now more like a marketing term. [A special committee](https://en.wikipedia.org/wiki/International_Technology_Roadmap_for_Semiconductors) has a meeting every two years where they take the previous node name, divide it by the square root of two, round to the nearest integer, declare the result to be the new node name, and then drink lots of wine. The "nm" doesn't mean nanometer anymore.

在计算机历史的大部分时间里，光学缩小是性能改进的主要驱动力。英特尔前首席执行官戈登·摩尔（Gordon Moore）在1975年预测，微处理器中的晶体管数量将每两年翻一番。他的预言一直持续到今天，被称为摩尔定律。

![](../img/dennard.png)

 Dennard scaling 和 摩尔定律 都不是实际的物理定律，而是聪明的工程师所做的观察。由于物理限制，它们注定会在某个时刻停止。事实上，由于功率问题， Dennard scaling  已死

Thermodynamically, a computer is just a very efficient device for converting electrical power into heat. This heat eventually needs to be removed, and there are physical limits to how much power you can dissipate from a millimeter-scale crystal. Computer engineers, aiming to maximize performance, essentially just choose the maximum possible clock rate so that the overall power consumption stays the same. If transistors become smaller, they have less capacitance, meaning less required voltage to flip them, which in turn allows increasing the clock rate.

Around 2005–2007, this strategy stopped working because of *leakage* effects: the circuit features became so small that their magnetic fields started to make the electrons in the neighboring circuitry move in directions they are not supposed to, causing unnecessary heating and occasional bit flipping.

The only way to mitigate this is to increase the voltage; and to balance off power consumption you need to reduce clock frequency, which in turn makes the whole process progressively less profitable as transistor density increases. At some point, clock rates could no longer be increased by scaling, and the miniaturization trend started to slow down.

## Modern Computing

Dennard scaling 已经结束了，但是摩尔定律还在继续

时钟速率趋于稳定，但是晶体管数量仍在增加，并且允许创建新的并行硬件。CPU设计不再追求更快的周期，而是开始专注于在单个周期内完成更有用的事情。晶体管没有变得更小，而是一直在改变形状。

这导致了越来越复杂的架构，每个周期能够做几十、几百甚至数千件不同的事情。

![Die shot of a Zen CPU core by AMD (~1,400,000,000 transistors)](../img/die-shot.jpg)

以下是一些利用更多可用晶体管的核心方法，这些方法正在推动最近的计算机设计：

-  重叠指令的执行 ，使CPU的不同部分同时工作( 流水线 pipelining);
- 执行操作而不等待前面的操作完成 (speculative and out-of-order execution 分支预测和无序执行);
- 添加多个执行单元以同时处理独立操作 (superscalar processors 超标量处理器);
- 增加机器字的大小，能够在拆分成组的 128、256 或 512 位数据块上执行相同操作的指令 ([SIMD](/hpc/simd/));
- 添加多级缓存以加快RAM 和外部存储的访问速度
- 在芯片上加多个相同内核 (parallel computing, GPUs);
- 在主板中使用多个芯片，在数据中心使用多台更便宜的计算机 (distributed computing，分布式计算);
- 使用定制硬件解决特定问题(ASICs, FPGAs).
