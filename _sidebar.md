
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
    - [4.2. Flags and Targets](004-compilation)
    - [4.3. Situational Optimizations](004-compilation)
    - [4.4. Contract Programming](004-compilation)
    - [4.5. Non-Zero-Cost Abstractions](004-compilation)
    - [4.6. Compile-Time Computation](004-compilation)
    - [4.7. Arithmetic Optimizations](004-compilation)
    - [4.8. What Compilers Can and Can't Do](004-compilation)

5. Profiling
 5.1. Instrumentation
 5.2. Statistical Profiling
 5.3. Program Simulation
 5.4. Machine Code Analyzers
 5.5. Benchmarking
 5.6. Getting Accurate Results
6. Arithmetic
 6.1. Floating-Point Numbers
 6.2. Interval Arithmetic
 6.3. Newton's Method
 6.4. Fast Inverse Square Root
 6.5. Integers
 6.6. Integer Division
 6.7. Bit Manipulation
(6.8. Data Compression)
7. Number Theory
 7.1. Modular Inverse
 7.2. Montgomery Multiplication
(7.3. Finite Fields)
(7.4. Error Correction)
 7.5. Cryptography
 7.6. Hashing
 7.7. Random Number Generation
8. External Memory
 8.1. Memory Hierarchy
 8.2. Virtual Memory
 8.3. External Memory Model
 8.4. External Sorting
 8.5. List Ranking
 8.6. Eviction Policies
 8.7. Cache-Oblivious Algorithms
 8.8. Spacial and Temporal Locality
(8.9. B-Trees)
(8.10. Sublinear Algorithms)
(9.13. Memory Management)
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