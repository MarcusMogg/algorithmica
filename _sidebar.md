
- [1. 复杂度模型](001-complexity/)
   
    - [1.1. 现代硬件](001-complexity/hardware.md)
    - [1.2. 编程语言](001-complexity/languages.md)
    - [1.3. 计算模型](001-complexity/lmodels.md)
    - [1.4. 何时优化](001-complexity/levels.md)

- [2.计算机体系结构](002-architecture/)

    - [2.1. 指令集](002-architecture/isa.md)
    - [2.2. 汇编语言](002-architecture/assembly.md)
    - [2.3. 循环和条件](002-architecture/loops.md)
    - [2.4. 函数和递归](002-architecture/functions.md)
    - [2.5. 间接分支](002-architecture/indirect.md)
    - [2.6. 机器码布局](002-architecture/layout.md)
    - [2.7. System Calls]()
    - [2.8. Virtualization]()

- [3. 指令级并行](003-pipelining/)

    - [3.1. Pipeline Hazards](003-pipelining/hazards.md)
    - [3.2. 分支的成本](003-pipelining/branching.md)
    - [3.3. 无分支编程](003-pipelining/branchless.md)
    - [3.4. 指令表](003-pipelining/tables.md)
    - [3.5. Instruction Scheduling](003-pipelining/scheduling.md)
    - [3.6. 吞吐量计算](003-pipelining/throughput.md)
    - [3.7. Theoretical Performance Limits](003-pipelining/limits.md)

- [4. 编译](004-compilation/)

    - [4.1. 编译阶段](004-compilation/stages.md)
    - [4.2. Flags and Targets](004-compilation/flags.md)
    - [4.3. 情景优化](004-compilation/situational.md)
    - [4.4. 契约编程](004-compilation/contracts.md)
    - [4.5. Non-Zero-Cost Abstractions](004-compilation/abstractions.md)
    - [4.6. 编译期计算](004-compilation/precalc.md)
    - [4.7. Arithmetic Optimizations](004-compilation/arithmetic.md)
    - [4.8. What Compilers Can and Can't Do](004-compilation/limitations.md)

- [5. Profiling](005-profiling)

    - [5.1. Instrumentation](005-profiling/instrumentation.md)
    - [5.2. 统计分析](005-profiling/events.md)
    - [5.3. 程序模拟](005-profiling/simulation.md)
    - [5.4. 机器码分析](005-profiling/mca.md)
    - [5.5. Benchmarking](005-profiling/benchmarking.md)
    - [5.6. 获取准确的结果](005-profiling/noise.md)

- [6. 算术](006-arithmetic)

    - [6.1. 浮点数](006-arithmetic/float.md)
    - [6.2. IEEE 754 浮点数](006-arithmetic/ieee-754.md)
    - [6.3. 舍入误差](006-arithmetic/errors.md)
    - [6.4. Newton's Method](006-arithmetic/newton.md)
    - [6.5. 平方根倒数速算法](006-arithmetic/rsqrt.md)
    - [6.6. 整数](006-arithmetic/integer.md)
    - [6.7. 整数除法](006-arithmetic/division.md)

- [7. Number Theory](007-number-theory)
    - [7.1. Modular Inverse](007-number-theory)
    - [7.2. Montgomery Multiplication](007-number-theory)
    - [(7.3. Finite Fields)](007-number-theory)
    - [(7.4. Error Correction)](007-number-theory)
    - [7.5. Cryptography](007-number-theory)
    - [7.6. Hashing](007-number-theory)
    - [7.7. Random Number Generation](007-number-theory)

- [8. External Memory](008-external-memory)

    - [8.1. Memory Hierarchy](008-external-memory)
    - [8.2. Virtual Memory](008-external-memory)
    - [8.3. External Memory Model](008-external-memory)
    - [8.4. External Sorting](008-external-memory)
    - [8.5. List Ranking](008-external-memory)
    - [8.6. Eviction Policies](008-external-memory)
    - [8.7. Cache-Oblivious Algorithms](008-external-memory)
    - [8.8. Spacial and Temporal Locality](008-external-memory)
    - [(8.9. B-Trees)](008-external-memory)
    - [(8.10. Sublinear Algorithms)](008-external-memory)
    - [(9.13. Memory Management)](008-external-memory)

9. RAM & CPU Caches
 9.1. Memory Bandwidth
 9.2. Memory Latency
 9.3. Cache Lines
 9.4. Memory Sharing
 9.5. Memory-Level Parallelism
 9.6. Prefetching
 9.7. Alignment and Packing
 9.8. Pointer Alternatives
 9.9. Cache Associativity
 9.10. Memory Paging
 9.11. AoS and SoA
10. SIMD Parallelism
 10.1. Intrinsics and Vector Types
 10.2. Moving Data
 10.3. Reductions
 10.4. Masking and Blending
 10.5. In-Register Shuffles
 10.6. Auto-Vectorization and SPMD
11. Algorithm Case Studies
 11.1. Binary GCD
(11.2. Prime Number Sieves)
 11.3. Integer Factorization
 11.4. Logistic Regression
 11.5. Big Integers & Karatsuba Algorithm
 11.6. Fast Fourier Transform
 11.7. Number-Theoretic Transform
 11.8. Argmin with SIMD
 11.9. Prefix Sum with SIMD
 11.10. Reading Decimal Integers
 11.11. Writing Decimal Integers
(11.12. Reading and Writing Floats)
(11.13. String Searching)
 11.14. Sorting
 11.15. Matrix Multiplication
12. Data Structure Case Studies
 12.1. Binary Search
 12.2. Static B-Trees
(12.3. Search Trees)
 12.4. Segment Trees
(12.5. Tries)
(12.6. Range Minimum Query)
 12.7. Hash Tables
(12.8. Bitmaps)
(12.9. Probabilistic Filters)