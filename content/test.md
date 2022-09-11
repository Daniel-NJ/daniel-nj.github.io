Title: OSDI2022 两篇论文阅读
Date: 2022-08-24 20:16
Category: Paper Reading

《XRP: In-Kernel Storage Functions with eBPF》这一篇文章介绍了一种对文件系统访问优化的手段，
通过eBPF（extend Berkeley Packet Filter) 一种将user space的函数offload到kernel space的linux框架，
通过将某些中间的文件系统IO读写直接在kernel space处理，节省了其通过linux kernel stack的overhead 从而提高性能，

文章的背景是当前出现了性能很高的NVMe存储器，带宽达到7GB/s， latency达到3us，这直接导致了linux kernel stack的开销占总通路的比重越来越大，
如何bypass掉这些开销变成一种理所应当的优化方向。
![Kernel’s latency overhead with 512 B random reads](/images/overhead.png)

bypass kernel software stack有两种思路，一种是用user mode软件替代kernel mode的调用，另一种是将一些用户态的逻辑放到kernel里来执行，
对于文件系统的优化貌似第二种更适合，因此提到了eBPF。
eBPF提供了一个机制，能避免中间数据在kernel和user space间移动，例如遍历一个B-tree，遍历的中间节点实际上可以不用返回给user space，如果能
在kernel space处理，直接发起访问下一个节点。
接着文章提出了两种hook的地方，一个是在syscall，一个是在NVMe driver，显然hook点放在NVMe driver取得更大的优化效果。
![two hook point](/images/two-kernel-hook.png)

但是放到NVMe driver，存在一些技术障碍，因为在这里没法看到文件系统的meta data, 最后文章讲其是如何去克服这些障碍的....




C.1 最简单的流水线包含了IF(instruction fetch) ID(instruction decode) EXEC(execute) MEM(memory action) WB(write back)

从上图流水线有三个观察：
1）指令cache和数据cache分开了，这样能消除IF和MEM反问内存带来的冲突
2）register file被用在两个stage(ID中需要读register和WB中要写)，我们要确保能同一个cycle做到读写同一个寄存器，就需要在一个cycle的前半段写，后半段读。
3）没有画出PC寄存器的处理过程，实际上在IF阶段需要处理PC自加的逻辑，同样在ID阶段，需要一个adder来时计算branch指令的target地址。

在流水线运行过程中，避免不同instruction在同一cycle中去使用相同的硬件资源之外，还需要保证不同instructions在不同stage不会相互干扰，做到这一点需要在流水
线的stage间增加pipeline register，用来缓存每个stage的输出，同时做为下一个cycle时 下一个stage的输入。

流水线只是增大了处理器的指令吞吐率，但是确增加了单条指令的执行时长。有如下原因导致了单条指令的执行时间变长：
1）流水线的深度导致的
2）流水线不同阶段所需的时长不一样，cycle数不能短于耗时最长的stage，这样必然导致大部分stage要有额外耗时
3）pipeline register delay和clock skew，第一个是指pipeline register对于输入要求其达到stable 才能触发寄存器的写过程，同时还包含了propagation delay。
clock skew指的是时钟到任意两个寄存器的有时间偏差，不能确保它们同时到达，所以要留一个余量，clock skew同时决定了pipeline的clock cycle的下限。




C.2 流水线的三大冒险：
1）structural hazards arise from resource conflicts
2) data hazards arise when an instruction depends on the result of a previous instruction.
3) control hazards arise from the pipelining of branches and other instruction that change the pc.

当遇到hazards时 流水线中的某条instruction需要stalled，其前面已经在跑的instruction仍然在跑，但是其后面已经issued的instruction也需要stalled，并且整个pipeline
不再接受新issued的instruction。
理论上的流水线处理器对于非流水线处理的加速比
speedup = pipline depth / (1 + pipline stall cycles per instruction)


Data Hazards的类型
1）Read After Write
2）Write After Read 这种在简单的5级流水里不会出现，但是加了指令乱序执行后就会发生
3）Write After Write 这种在简单的5级流水里不会出现，但是加了指令乱序执行后就会发生 

通过bypassing解决数据冒险，但是有些情况下没法解决，只能通过stalled当前指令以及后面的指令来解决


Control hazards
主要是涉及branch预测
branch预测失败带来了惩罚，需要重新建立流水线
如何减少分支预测失败带来的惩罚，主要能做的是提升分支预测的成功率
分支预测在ID计算一个猜测的address，在EXE去比较改猜测的地址是否和计算后的地址一致，决定预测是否成功
分支预测有两种算法：静态预测，动态预测，分支预测buffers

C.3讲了流水线是如何实现的，这里可以看到每个stage到底负责哪些工作，蛮有意思，如何实现流水线的控制逻辑，如何识别data hazards，如何去做bypassing

C.4举例说明为什么实现一个流水线很难，从如何处理异常来说明，以及指令集的兼容性上来说

C.5如何从integer pipeline扩展到fp pipeline，利用spec来分析fp pipeline的性能.

C.6拿MPIS R4000的流水线来做一个实例分析

3.2 用来利用ILP的编译技术
Basic pipeline scheduling and loop unrolling
保持一个pipeline尽可能满负荷运行，就是尽可能找到不存在依赖的指令，对于存在依赖的指令i和j，尽量安排j和i的距离为i的pipeline latency
编译器的指令调度基于代码中存在的ILP，以及流水线中function unit的latencies
循环展开 首先要看相邻迭代间有没有依赖，如果没有则可以展开，展开后可以去掉一些索引更新和branch判断，然后是调整指令排布，将没有依赖的指令排在一起

循环展开有两个限制，一个展开数太少，branch指令带来的开销比较大； 展开太多，代码size过大，会引起icache miss概率增大
展开太多还导致用了更多的register，导致可用的寄存器变少，理论上会认为会得到好的性能提升，而实际上提升并不理想，这种情况在多发射情况下，更明显。

3.3 利用高级的分支预测减少跳转指令带来的开销




intel i7 L1 miss带来的延时是10个cycle， L2 miss是L1 miss的3倍以上，  L3 miss是L1 miss的13倍以上
提升ILP的主要限制因素是memory system

Pentium 4的流水线深度是20， i7 920的深度是14， i7 920的性能是约是Pentium 4的3倍


