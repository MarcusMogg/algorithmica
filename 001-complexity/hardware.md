
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

Consider what happens when we scale a microchip down. A smaller circuit requires proportionally fewer materials, and smaller transistors take less time to switch (along with all other physical processes in the chip), allowing reducing the voltage and increasing the clock rate.

想一想当我们缩小微芯片大小时会发生什么。

A more detailed observation, known as the *Dennard scaling*, states that reducing transistor dimensions by 30%

- doubles the transistor density ($0.7^2 \approx 0.5$),
- increases the clock speed by 40% ($\frac{1}{0.7} \approx 1.4$),
- and leaves the overall *power density* the same.

Since the per-unit manufacturing cost is a function of area, and the exploitation cost is mostly the cost of power[^power], each new "generation" should have roughly the same total cost, but 40% higher clock and twice as many transistors, which can be promptly used, for example, to add new instructions or increase the word size — to keep up with the same miniaturization happening in memory microchips.

[^power]: The cost of electricity for running a busy server for 2-3 years roughly equals the cost of making the chip itself.

Due to the trade-offs between energy and performance you can make during the design, the fidelity of the fabrication process itself, such as "180nm" or "65nm," directly translating to the density of transistors, became the trademark for CPU efficiency[^fidelity].

[^fidelity]: At some point, when Moore's law started to slow down, chip makers stopped delineating their chips by the size of their components — and it is now more like a marketing term. [A special committee](https://en.wikipedia.org/wiki/International_Technology_Roadmap_for_Semiconductors) has a meeting every two years where they take the previous node name, divide it by the square root of two, round to the nearest integer, declare the result to be the new node name, and then drink lots of wine. The "nm" doesn't mean nanometer anymore.

Throughout most of the computing history, optical shrinking was the main driving force behind performance improvements. Gordon Moore, the former CEO of Intel, predicted in 1975 that the transistor count in microprocessors will double every two years. His prediction held to this day and became known as *Moore's law*.

![](../img/dennard.png)

Both Dennard scaling and Moore's law are not actual laws of physics, but just observations made by savvy engineers. They are both destined to stop at some point due to fundamental physical limitations, the ultimate one being the size of silicon atoms. In fact, Dennard scaling already did — due to power issues.

Thermodynamically, a computer is just a very efficient device for converting electrical power into heat. This heat eventually needs to be removed, and there are physical limits to how much power you can dissipate from a millimeter-scale crystal. Computer engineers, aiming to maximize performance, essentially just choose the maximum possible clock rate so that the overall power consumption stays the same. If transistors become smaller, they have less capacitance, meaning less required voltage to flip them, which in turn allows increasing the clock rate.

Around 2005–2007, this strategy stopped working because of *leakage* effects: the circuit features became so small that their magnetic fields started to make the electrons in the neighboring circuitry move in directions they are not supposed to, causing unnecessary heating and occasional bit flipping.

The only way to mitigate this is to increase the voltage; and to balance off power consumption you need to reduce clock frequency, which in turn makes the whole process progressively less profitable as transistor density increases. At some point, clock rates could no longer be increased by scaling, and the miniaturization trend started to slow down.

<!--

### Power Efficiency

It may come as a surprise, but the primary metric for modern CPUs is not the clock frequency, but rather "useful operations per joule," or, more practically put, "useful operations per dollar."

Thermodynamically, a computer is just a very efficient device for converting electrical power into heat. This heat eventually needs to be removed, and it's not straightforward to do when you are working with a millimeter-scale crystal. There are physical limits to how much power you can consume and then dissipate.

Historically, the three main variables guiding microchip designs are power, performance, and area (PPA), commonly defined in watts, hertz, and nanometers. Until ~2005, cost, which was mainly a function of area, and performance, used to be the most important criteria. But as battery-driven mobile devices started replacing PCs, power quickly and firmly moved up on top of the list, followed by cost and performance.

Leakage: interfering magnetic fields make electrons move in the directions they are not supposed to and cause unnecessary heating. It isn't bad by itself: to mitigate it you need to increase the voltage, and it won't flick any bits. But the problem is that the smaller a circuit is, the harder it is to cope with this by isolating the wires. So modern chips keep the clock frequency at a level that won't cause overheat, although physically there aren't other reasons why they shouldn't.

-->

### Modern Computing

Dennard scaling has ended, but Moore's law is not dead yet.

Clock rates plateaued, but the transistor count is still increasing, allowing for the creation of new, *parallel* hardware. Instead of chasing faster cycles, CPU designs started to focus on getting more useful things done in a single cycle. Instead of getting smaller, transistors have been changing shape.

This resulted in increasingly complex architectures capable of doing dozens, hundreds, or even thousands of different things every cycle.

![Die shot of a Zen CPU core by AMD (~1,400,000,000 transistors)](../img/die-shot.jpg)

Here are some core approaches making use of more available transistors that are driving recent computer designs:

- Overlapping the execution of instructions so that different parts of the CPU are kept busy (pipelining);
- Executing operations without necessarily waiting for the previous ones to complete (speculative and out-of-order execution);
- Adding multiple execution units to process independent operations simultaneously (superscalar processors);
- Increasing the machine word size, to the point of adding instructions capable of executing the same operation on a block of 128, 256, or 512 bits of data split into groups ([SIMD](/hpc/simd/));
- Adding [layers of cache](/hpc/cpu-cache/) on the chip to speed up [RAM and external memory](/hpc/external-memory/) access time (memory doesn't quite follow the laws of silicon scaling);
- Adding multiple identical cores on a chip (parallel computing, GPUs);
- Using multiple chips in a motherboard and multiple cheaper computers in a data center (distributed computing);
- Using custom hardware to solve a specific problem with better chip utilization (ASICs, FPGAs).

For modern computers, the "[let's count all operations](../)" approach for predicting algorithm performance isn't just slightly wrong but is off by several orders of magnitude. This calls for new computation models and other ways of assessing algorithm performance.

<!--

Pointer jumping and processing in most scripting languages: $10^7$
Branchy operations in native languages: $10^8$
Branchless scalar processing in native languages: $10^9$
Bandwidth-bound or complex SIMD applications: $10^{10}$
Linear algebra, single core: $10^{11}$
Typical desktop CPU: $10^{12}$
Typical mobile phone GPU: $10^{12}$
Typical integrated graphics card: $2 \cdot 10^{12}$
High-end gaming setups: $10^{13}$
Deep learning hardware: $10^{14}$
Deep learning full rigs: $10^{15}$
Being considered a supercomputer: $10^{16}$
Setups used to train LM neural networks: $5 \cdot 10^{17}$
Fugaku (#1): $2 \cdot 10^{18}$
Folding@home: $3 \cdot 10^{18}$

-->
