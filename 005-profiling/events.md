
Instrumentation 进行 profiling 是一个相当麻烦的方法，特别是当你对程序中的多个小节感兴趣。而且即便它可以部分由工具自动完成，它固有的开销也让你无法进行更细致的分析。

另一种侵入性比较小的分析方式是 以随机间隔暂停程序，然后查看指令指针。指针在每个函数块听停留的次数，和执行函数花费的时间大致成比例。你也可以使用这种方式得到其他有用信息，比如通过观察调用栈 找到一个函数被哪些函数调用。

原则上来说，这可以通过`gdb`运行程序 + 随机执行`ctrl+c`来完成，但是现代CPU和操作系统提供了专门的工具来完成这种类型的 分析。
## 硬件事件

硬件性能计数器*performance counters*  是微处理器内置的特殊的寄存器s，可以存储多个特定的硬件相关活动。在集成电路上添加是便宜的，因为它们基本上只是连接有激活线(?)的二进制计数器。

每个性能计数器 链接到一个大的电路集，而且可以配置为在特定的硬件事件时增加，比如 分支预测错误、缓存失效。你可以在程序开始时将 计数器置为0，然后运行，然后在结束时输出存储的值，它等于在执行期间 特定事件被触发的次数。

你也可以使用多路执行 来跟踪多个事件，即 在固定的时间间隔 暂停程序，然后重新配置计数器。 这种场景下结果可能不是精确的，但是统计学上接近。一个细节是它不能通过简单地增加采样频率来增加准确性，因为这样会影响性能因此扭曲分布，所以为了执行多个分析，你需要执行程序一个长的周期。

总的来说，事件驱动的统计 分析 通常是最有效 而且简单的方案来诊断性能问题。

## perf

依赖上述的事件采样技术的性能分析器 称为*statistical profilers* 。有多种，但本书中主要使用的是 [perf](https://perf.wiki.kernel.org/)，linux内核装载的一个 statistical profilers。 在非linux平台，可以使用Intel 提供的  [VTune](https://software.intel.com/content/www/us/en/develop/tools/oneapi/components/vtune-profiler.html#gs.cuc0ks) ，几乎提供了相同的功能。

perf是一个命令行工具，它基于程序的执行实时生成报告。它不需要源代码 而且可以 分析非常多的应用，即使是那些涉及多个进程和与操作系统交互的应用程序。

出于解释目的，我编写了一个小程序，它创建一个由一百万个随机整数组成的数组，对其进行排序，然后对其进行一百万个二分搜索：

```c++
void setup() {
    for (int i = 0; i < n; i++)
        a[i] = rand();
    std::sort(a, a + n);
}

int query() {
    int checksum = 0;
    for (int i = 0; i < n; i++) {
        int idx = std::lower_bound(a, a + n, rand()) - a;
        checksum += idx;
    }
    return checksum;
}
```

编译之后 (`g++ -O3 -march=native example.cc -o run`), 可以使用perf运行 `perf stat ./run`, 输出执行期间基本性能事件的计数:

```yaml
 Performance counter stats for './run':

        646.07 msec task-clock:u               # 0.997 CPUs utilized          
             0      context-switches:u         # 0.000 K/sec                  
             0      cpu-migrations:u           # 0.000 K/sec                  
         1,096      page-faults:u              # 0.002 M/sec                  
   852,125,255      cycles:u                   # 1.319 GHz (83.35%)
    28,475,954      stalled-cycles-frontend:u  # 3.34% frontend cycles idle (83.30%)
    10,460,937      stalled-cycles-backend:u   # 1.23% backend cycles idle (83.28%)
   479,175,388      instructions:u             # 0.56  insn per cycle         
                                               # 0.06  stalled cycles per insn (83.28%)
   122,705,572      branches:u                 # 189.925 M/sec (83.32%)
    19,229,451      branch-misses:u            # 15.67% of all branches (83.47%)

   0.647801770 seconds time elapsed
   0.647278000 seconds user
   0.000000000 seconds sys
```


你可以看到这个执行花费 0.53s 或者说 1.32GHz时钟频率下 852M循环，479M指令被执行。有 122.7M  分支，其中 15.7% 预测失败。

你可以使用 `perf list` 获取 所以支持的事件，让后使用 `-e` 选项🔝一系列你想要的数据。例如，为了诊断二分搜索，我们主要关注 缓存未命中:

```yaml
> perf stat -e cache-references,cache-misses ./run

91,002,054      cache-references:u                                          
44,991,746      cache-misses:u      # 49.440 % of all cache refs
```

就其本身而言， `perf stat` 只需为整个程序设置性能计数器。它可以告诉你分支错误预测的总数，但它不会告诉你它们在哪里发生，更不用说它们为什么会发生了。

为了尝试之前说的 stop-the-world 方式，我们需要使用 `perf record <cmd>`，它会记录分析数据，然后导出为`perf.data` 文件，然后使用`perf report` 进行分析。

当你调用`perf report`，它首先会展示一个 `top`类似的交互报告，告诉你 每个函数被执行多少次：

```
Overhead  Command  Shared Object        Symbol
  63.08%  run      run                  [.] query
  24.98%  run      run                  [.] std::__introsort_loop<...>
   5.02%  run      libc-2.33.so         [.] __random
   3.43%  run      run                  [.] setup
   1.95%  run      libc-2.33.so         [.] __random_r
   0.80%  run      libc-2.33.so         [.] rand
```


注意，对每个函数，只有它的*overhead* 被列出，而不是总的执行时间（e.g., `setup` 包含 `std::__introsort_loop`但是只有它自身的overhead 被统计为  3.43%）。有工具可以将perf 报告组织为  [火焰图](https://www.brendangregg.com/flamegraphs.html) ，以使它们更清晰。你还需要考虑到 可能的inline，这显然是发生在`std::lower_bound`。Perf 还跟踪共享库（如 `libc` ），通常还跟踪任何其他生成的进程：如果需要，您可以使用 perf 启动 Web 浏览器并查看其中发生了什么。

然后，你可以在这些函数的任何地方进行“缩放”，它将提供带有相关热力图的反汇编。例如，下面是 `query`的汇编

```asm
       │20: → call   rand@plt
       │      mov    %r12,%rsi
       │      mov    %eax,%edi
       │      mov    $0xf4240,%eax
       │      nop    
       │30:   test   %rax,%rax
  4.57 │    ↓ jle    52
       │35:   mov    %rax,%rdx
  0.52 │      sar    %rdx
  0.33 │      lea    (%rsi,%rdx,4),%rcx
  4.30 │      cmp    (%rcx),%edi
 65.39 │    ↓ jle    b0
  0.07 │      sub    %rdx,%rax
  9.32 │      lea    0x4(%rcx),%rsi
  0.06 │      dec    %rax
  1.37 │      test   %rax,%rax
  1.11 │    ↑ jg     35
       │52:   sub    %r12,%rsi
  2.22 │      sar    $0x2,%rsi
  0.33 │      add    %esi,%ebp
  0.20 │      dec    %ebx
       │    ↑ jne    20
```

左列是指令指针在特定行上停止的次数。你可以看到，我们在跳转指令上花费了 ~65% 的时间，因为它前面有一个比较运算符，表明控制流在那里等待这个比较的决定。

由于流水线和无序执行等复杂性，“now”在现代 CPU 中不是一个定义明确的概念，因此当指令指针向前漂移一点时，数据略有不准确。指令级数据仍然有用，但在单个周期级别，我们需要切换到更精确的东西。