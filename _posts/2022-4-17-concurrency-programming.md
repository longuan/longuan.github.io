


## 什么是并发？与并行的区别

网上关于并发和并行众说纷纭，我也看了很多，比较认可和理解的是go blog中的这段话。

> In programming, concurrency is the _composition_ of independently executing processes, while parallelism is the simultaneous _execution_ of (possibly related) computations. Concurrency is about _dealing with_ lots of things at once. Parallelism is about _doing_ lots of things at once.

一个application，比如DBMS、web server或者operating system，可以分解成很多的task。对于并发concurrency，是说此application的多个task都在进行（in progress）。对于并行parallelism，是说此application的多个task同时在执行（being executing）。这两个的视角还是不太一样的，并发更多是从application角度，并行更多是从run-time behaviour角度。按我的理解，time-slicing是跟并行是一个层次的，都是**并发执行**的一种方式。

我们面对的大多数application的各个task之间都是有很强的依赖关系，而且瓶颈并不在processor，比如pipeline、network io、disk io，这时候把tasks并行执行收益甚小。一些计算密集型任务，而且task或者data可以独立拆分的application，比如矩阵的数学运算，并行运行就可以有O(n)的收益比。

操作系统往往会托管processors，也有着自己的调度系统，管理各个进程和线程的运行。在这个基础上，我们更多地谈论并发编程，更加关注application design、synchronization层面的问题，实际的执行就交给操作系统。

我们日常所说的并发编程concurrency programming也是从application出发的，囊括application design、thread/process synchronization、program language、operation system、hardware等等方面。

并发的好处有很多，比如提升对CPU的使用效率、降低系统的响应时间等等。缺点就是不容易正确的实现，而且难以调试和复现。

为了克服这个缺点，也就是又快又好地实现并发，需要从三方面入手：
- 软件设计。自顶向下地对应用程序进行分解剖析，理清总体流程，进行模块化分工。在设计之初就应该考虑程序的并发模型以及并发方式。这块目前还不是很了解，不细说。
- 熟悉环境。详细地了解操作系统、编程语言以及硬件架构。目前主要的开发平台是Linux，这个应该不用多说，还有posix 线程模型，cpu和内存的访存体系也应该关注。不同编程语言提供的特性差别比较大，灵活性也各有所异。
- 代码实现。前面已经把程序分解为不同的task，这一步就是借助操作系统以及编程语言确保各个task正常工作、task之间的交互以及通信不出错，协同工作。


## 并发是如何工作的

### 从硬件层面

每一个程序和操作系统代码都是一条条的指令instructions。处理器processor从内存中一条条地取指令、解码、执行，循环往复。处理器根据指令去读写和计算数据，如果有多个处理器同时在工作，就会出现数据争用等等问题。对于多处理器的并行问题，各个CPU提供商也都提供了相应的能力，例如intel的[开发者手册](http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf) 中第八章和第11章等等内容。

> The 32-bit IA-32 processors support locked atomic operations on locations in system memory. These operations are typically used to manage shared data structures (such as semaphores, segment descriptors, system segments, or page tables) in which two or more processors may try simultaneously to modify the same field or flag. The processor uses three interdependent mechanisms for carrying out locked atomic operations:
>  - Guaranteed atomic operations
>  - Bus locking, using the LOCK# signal and the LOCK instruction prefix
>  - Cache coherency protocols that ensure that atomic operations can be carried out on cached data structures (cache lock); this mechanism is present in the Pentium 4, Intel Xeon, and P6 family processors

>The Intel486 processor (and newer processors since) guarantees that the following basic memory operations will always be carried out atomically:
>    - Reading or writing a byte
>    - Reading or writing a word aligned on a 16-bit boundary
>    - Reading or writing a doubleword aligned on a 32-bit boundary
>
>The Pentium processor (and newer processors since) guarantees that the following additional memory operations will always be carried out atomically:
>   - Reading or writing a quadword aligned on a 64-bit boundary
>   - 16-bit accesses to uncached memory locations that fit within a 32-bit data bus
>
>The P6 family processors (and newer processors since) guarantee that the following additional memory operation will always be carried out atomically:
>  - Unaligned 16-, 32-, and 64-bit accesses to cached memory that fit within a cache line

> To explicitly force the LOCK semantics, software can use the LOCK prefix with the following instructions when they are used to modify a memory location. An invalid-opcode exception (#UD) is generated when the LOCK prefix is used with any other instruction or when no write operation is made to memory (that is, when the destination operand is in a register).
> - The bit test and modify instructions (BTS, BTR, and BTC). 
> - The exchange instructions (XADD, CMPXCHG, and CMPXCHG8B).
> - The LOCK prefix is automatically assumed for XCHG instruction.
> - The following single-operand arithmetic and logical instructions: INC, DEC, NOT, and NEG.
> - The following two-operand arithmetic and logical instructions: ADD, ADC, SUB, SBB, AND, OR, and XOR.

> To allow performance optimization of instruction execution, the IA-32 architecture allows departures from strong ordering model called processor ordering in Pentium 4, Intel Xeon, and P6 family processors. These processor ordering variations (called here the memory-ordering model) allow performance enhancing operations such as allowing reads to go ahead of buffered writes. The goal of any of these variations is to increase instruction execution speeds, while maintaining memory coherency, even in multiple-processor systems.
> 
> In a single-processor system for memory regions defined as write-back cacheable, the memory-ordering model respects the following principles:
>  - Reads are not reordered with other reads.
>  - Writes are not reordered with older reads.
>  - Writes to memory are not reordered with other writes, with the following exceptions:...
>  - Reads may be reordered with older writes to different locations but not with older writes to the same location.
>  - Reads or writes cannot be reordered with I/O instructions, locked instructions, or serializing instructions.
>  - Reads cannot pass earlier LFENCE and MFENCE instructions.
>  - ......
> 
> 
> In a multiple-processor system, the following ordering principles apply:
>  - ......


- 处理器会保证有一些操作是原子的
- 可以使用LOCK来强制一条指令是原子的
- 处理器为了加速指令的执行，往往会做一些重排，导致读写顺序发生变化。这个优化在单线程模型下是不会影响程序正确性的，但是在多线程情况下就会出现一些问题。通过引入memory-ordering model和**memory fence**来控制重排行为
- 对多处理器系统来说，在硬件视角，有这么几个内存一致模型：
	- Sequential consistency (all reads and all writes are in-order)
	- Relaxed consistency (some types of reordering are allowed)
		- Loads can be reordered after loads (for better working of cache coherency, better scaling)
		- Loads can be reordered after stores
		- Stores can be reordered after stores
		- Stores can be reordered after loads
	- Weak consistency (reads and writes are arbitrarily reordered, limited only by explicit memory barriers)

### 从编译器层面

编写程序时，我们肯定不会手写汇编或者cpu指令，而是依靠编译器将用高级编程语言编写的源代码转换为对应的机器语言。编译器在背后替我们做了很多事，包括代码展开、翻译、链接、优化等等。

由于更接近源代码，相比与CPU只知道流水线里面的指令，编译器能够看到所有的代码。如果编译器能够分析出整个程序的控制依赖和数据依赖，那么就既能够保证正确性又能提高执行效率。可是目前编译器还无法做到这一点，特别是在复杂的多线程环境下。那怎么办？于是编译器依然按照它的方式进行优化，同时提供选项给程序员，由程序员告诉编译器额外的控制依赖和数据依赖。

编译器优化的一个基本原则就是**as-if-serial**语义（不能改变单线程程序的行为），这也是处理器必须遵守的。为了遵守as-if-serial语义，编译器和处理器不会对存在依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果编译器和处理器没有识别出来操作之间存在依赖关系，这些操作就可能被编译器重排序。

为了阻止一些错误的优化，我们可以使用compiler barriers告诉编译器应该如何做。

https://en.wikipedia.org/wiki/Memory_ordering

> A memory model is an agreement between the machine architects and the compiler writers to ensure that most programmers do not have to think about the details of modern computer hardware. Without a memory model, few things related to threading, locking, and lock-free programming would make sense. The key guarantee is: Two threads of execution can update and access separate memory locations without interfering with each other. But what is a “memory location?” A memory location is either an object of scalar type or a maximal sequence of adjacent bit-fields all having non-zero width.          https://isocpp.org/wiki/faq/cpp11-language-concurrency#memory-model

### 从操作系统层面

从常用的操作系统来说，并发的基本单位是线程。线程是进程中的一个执行流，进程是操作系统进行资源管理的基本单位。线程/进程也就是我们常用的task的“容器”。


- each thread maintains exception handlers, a scheduling priority, thread local storage, a unique thread identifier, and a set of structures the system will use to save the thread context until it is scheduled. The thread context includes the thread's set of machine registers, the kernel stack, a thread environment block, and a user stack in the address space of the thread's process. Threads can also have their own security context, which can be used for impersonating clients.
- 多线程间共享进程的address space

操作系统有着自己的调度系统来协调多个线程和进程的并发执行。这个调度策略以及调度顺序对我们一般是完全透明的，好像我们的代码是真正地在CPU上不间断运行。如果你的程序是多线程，可以认为自己的代码是在“并行执行”。

有了线程的概念，可以更形式化地描述并发中的一些问题（在后面介绍）：
- A **critical section** is a piece of code that accesses a shared resource, usually a variable or data structure.
- data race
- race condition

出现这些问题的原因就是多线程同时执行critical section的代码导致的。前面说到硬件层面提供了原子指令来保证一条指令的执行不会受到干扰，而critical section很可能有很多条指令，操作系统也提供了**synchronization primitives**来保证critical section的执行原子性以及协调多线程之间的执行顺序。

操作系统已经实现了很多的synchronization primitives。当然你也可以在自己的软件内实现synchronization primitives或者使用编程语言library里的。

Linux上有sys_futex系统调用。

POSIX Threads模型上有：
- pthread_mutex，基于硬件提供的原子操作和操作系统提供的futex，保证了三件事
	- Atomicity。对一个mutex加锁和解锁是一个原子的操作，操作系统保证同一时刻没有两个线程能同时lock一个mutex
	- Singularity。另一个线程要想获取已经locked的mutex，只能等这个mutex被持有者unlock
	- Non-Busy Wait。想获取已经locked的mutex的线程会被挂起（让出CPU），直到锁被释放，此线程就会被唤醒，然后持有锁。
- pthread_cond，A condition variable is a mechanism that allows threads to wait (without wasting CPU cycles) for some even to occur. 通常需要结合一个mutex使用。
- samaphore。可以把信号量当作mutex加上条件变量使用。

### 从编程语言层面

平时用的都是高级语言来写代码，不同的高级语言也都做了不同程度的抽象。

我自己本来也没用过几种编程语言，而且对这方面涉猎很少，所以没有什么见解。关于这方面的知识推荐几个链接：
- 《Programming Language Memory Models》https://research.swtch.com/plmm 
- 《Updating the Go Memory Model》https://research.swtch.com/gomm 
- 《A Survey of Programming Language Memory Models》 https://link.springer.com/article/10.1134/S0361768821060050

下面一部分是讲几个编程语言中的原子操作的，来自[StackOverflow](https://stackoverflow.com/questions/1762148/atomic-instruction)：

**C#**
-   C# guarantees that operations on any built-in value type that takes up to 4-bytes are atomic
-   Operations on value types that take more than four bytes (double, long, etc.) are not guaranteed to be atomic
-   The CLI guarantees that reads and writes of variables of value type that are the size (or smaller) of the processor's natural pointer size are atomic
    -   Ex - running C# on a 64-bit OS in a 64-bit version of the CLR performs reads and writes of 64-bit doubles and long integers atomically
-   Creating atomic operations :
    -   .NET provodes the Interlocked Class as part of the System.Threading namespace
    -   The Interlocked Class provides atomic operations such as increment, compare, exchange, etc.

> ```
> using System.Threading;             
> 
> int unsafeCount;                          
> int safeCount;                           
> 
> unsafeCount++;                              
> Interlocked.Increment(ref safeCount);
> ```

**C++**
-   C++ standard does not guarantee atomic behavior
-   All C / C++ operations are presumed non-atomic unless otherwise specified by the compiler or hardware vendor - including 32-bit integer assignment
-   Creating atomic operations :
    -   The C++ 11 concurrency library includes the - Atomic Operations Library ()
	    - In the new C++11 (formerly known as C++0x) atomic library standard, every non-relaxed atomic operation acts as a compiler barrier as well.
    -   The Atomic library provides atomic types as a template class to use with any type you want
    -   Operations on atomic types are atomic and thus thread-safe

**Java**
-   Java guarantees that operations on any built-in value type that takes up to 4-bytes are atomic
-   Assignments to volatile longs and doubles are also guaranteed to be atomic
-   Java provides a small toolkit of classes that support lock-free thread-safe programming on single variables through java.util.concurrent.atomic
-   This provides atomic lock-free operations based on low-level atomic hardware primitives such as compare-and-swap (CAS) - also called compare and set :
    -   CAS form - boolean compareAndSet(expectedValue, updateValue );
        -   This method atomically sets a variable to the updateValue if it currently holds the expectedValue - reporting true on success


### 总结

总结来说，并发编程基本涉及到计算机的方方面面，从硬件到软件，从体系结构到软件工程，而且每一层都有大量的问题都需要关注和取舍。其实如果我们的程序不涉及到什么高性能的无锁编程、不需要极限压榨cpu，那么硬件层面和编译器层面我们基本不需要了解。我们只需要正常使用操作系统和编程语言提供的并发支持就可以了。甚至操作系统层面的东西也不需要考虑，只用学一个编程语言就可以了，比如为并发而生的golang。


## 并发中常见的问题

综合上面对各个层面的分析，根据物理架构以及运行环境，还有一些编程中需要注意的点，总结一下并发中常见的问题。

### deadlock

死锁产生的四个必要条件。
- 互斥：一个资源每次只能被一个进程使用
- 请求与保持：一个进程请求资阻塞时，不释放已获得的资源
- 不剥夺：进程已获得的资源不能强行剥夺
- 循环等待：若干进程之间形成头尾相接的循环等待资源关系

常见的死锁有以下几种类型：
- AA型死锁
- ABBA型死锁

如何避免：编码规范，加锁顺序（lock ordering），死锁预防和死锁避免算法。

### data race

data race就是我们常见的“多个线程访问同一个变量，有读有写，并且没有做互斥保护”带来的问题。

Conflicting accesses of the same variable that are not ordered by a happens-before relationship

https://stackoverflow.com/questions/11276259/are-data-races-and-race-condition-actually-the-same-thing-in-context-of-conc

如何避免：识别哪些变量是会被多个线程访问，并且至少一个线程是写的，记得加锁，也不要加错了锁

### race condition
#### Atomicity-Violation Bugs

严格意义上来讲，Data Race只是Atomicity Violation的一个特例，因为即使加锁保证了某块代码的原子性，但是Atomic + Atomic != Atomic，还是有可能出现Atomicity-Violation bugs。

例如queue是一个线程安全的队列，isEmpty和poll两个函数都是线程安全的，但是两者组合形成的下面三行代码就不是原子的。

```c
if(!queue.isEmpty()) {  
   queue.poll(obj);  
}  
```

如何避免：加锁

#### Order-Violation Bugs

意思就是程序运行的顺序不符合程序员预期的顺序。

比如程序对全局变量的初始化顺序有依赖。

如何避免：着重分析变量的数据依赖关系，使用锁或者条件变量或者信号量等等保证顺序


### memory model/order

由于CPU与memory之间存在多层的cache，并且CPU和编译器存在对指令重排的优化，所以抽象出来了一层memory model，来描述多线程是如何通过内存来访问共享数据的。

正常情况下是不用管memory model的，因为编程语言一般默认要求强一致，但当你使用其他的memory model就要当心这些问题的发生。这里面牵扯到memory consistency 和 cache coherence。

例如c++11中的`std::memory_model`：

> `std::memory_order` specifies how memory accesses, including regular, non-atomic memory accesses, are to be ordered around an atomic operation. Absent any constraints on a multi-core system, when multiple threads simultaneously read and write to several variables, one thread can observe the values change in an order different from the order another thread wrote them. Indeed, the apparent order of changes can even differ among multiple reader threads. Some similar effects can occur even on uniprocessor systems due to compiler transformations allowed by the memory model. 
   The default behavior of all atomic operations in the library provides for _sequentially consistent ordering_ (see discussion below). That default can hurt performance, but the library's atomic operations can be given an additional `std::memory_order` argument to specify the exact constraints, beyond atomicity, that the compiler and processor must enforce for that operation.

如何避免：使用memory barrier/memory fence；或者使用高级编程语言中封装好的特性，例如java中的volatile、cpp中的atomic variable。

#### Out-of-order execution

指CPU的乱序执行。在单线程情况下是没有问题的，但是多线程情况下，依赖关系比较复杂，很容易出现问题。

#### compiler reordering optimizations

指编译器优化造成的指令重排。同out-of-order execution，多线程环境下可能出现问题。

#### Store buffer、Invalidate queues

现代CPU中通常会引入Store buffer和Invalidate queues两个组件来提高CPU的利用率，但缺点就是会导致并行情况下的一些乱序行为。不过硬件层面也都提供了各种手段来避免这种乱序的发生，比如memory barrier。


## 如何更好地编写并发代码


在总结并发中常见的问题时就已经给出了常用的规避bug的方法。

主要就几个方面：
- 注意编码，不要死锁。这个是重中之重。
- 避免data race。保护共享变量，访问的时候一定要保证原子性。
- 避免race condition。识别并保护critical section，避免上面提到的顺序性并发问题。
- 设计memory model时，关注共享变量在多线程中的可见性。

另外还要加一点：多用sanitizer，趁早检测出bug。

- [ThreadSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) (tsan): 数据竞争
- [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) (asan); [(paper)](https://www.usenix.org/conference/atc12/technical-sessions/presentation/serebryany): 非法内存访问
-  [MemorySanitizer](https://clang.llvm.org/docs/MemorySanitizer.html) (msan): 未初始化的读取
- [UBSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) (ubsan): undefined behavior
- 上锁顺序


## 参考

- https://www.zhihu.com/question/33515481
- https://go.dev/blog/waza-talk
- https://preshing.com/20130618/atomic-vs-non-atomic-operations/
- https://preshing.com/20120625/memory-ordering-at-compile-time/
- https://www.kernel.org/doc/html/v4.10/process/volatile-considered-harmful.html
- https://docs.microsoft.com/en-gb/windows/win32/procthread/about-processes-and-threads
- https://research.swtch.com/hwmm
- https://pages.cs.wisc.edu/~remzi/OSTEP/
- https://en.wikipedia.org/wiki/Memory_model_(programming)
- https://en.wikipedia.org/wiki/Memory_barrier
- https://en.wikipedia.org/wiki/Memory_ordering
- https://zhuanlan.zhihu.com/p/48157076
- http://jyywiki.cn/OS/2022/