
流水线通过并发执行指令以隐藏其延迟，但是也创造了一下潜在的障碍—— 通常称为流水线危险 *pipeline hazards*，即下一条指令不能在下一个周期执行的情况。

有几种情况：

* *structural hazard* 当两个或者更多指令需要 CPU的同一部分 (e.g., 一个执行单元).
* *data hazard* 有一个操作数需要等待之前的步骤计算完.
* *control hazard* CPU无法分辨接下来需要执行哪些指令.

解决 hazard 的唯一方式是流水线停止： 停止所有之前步骤的进度，直到阻塞的原因消失。 这会在流水线中造成 气泡 —— 执行单元空闲切未完成任何有用的工作。

![Pipeline stall on the execution stage](../img/bubble.png)

不同的 hazards 有不同的惩罚：

- structural hazards, 你需要等待（通常至少一个周期）直到执行单元完成.  它们是性能的根本瓶颈，而且无法避免—— 你必须围绕它们进行设计。
- data hazards, 你必须等待依赖的数据计算完成( *critical path* 关键路径的延迟). Data hazards 可以通过重新组织计算、缩短关键路径来解决.
- control hazards, 你通常需要刷新整个流水线然后重启，浪费 15-20 个周期.  它们可以通过 移除分支、使得分支更容易预测 来解决，让CPU可以更高效的猜测下一条指令。

由于它们对性能的影响非常不同，我们将以危害性相反的顺序开始。