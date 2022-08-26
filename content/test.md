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
