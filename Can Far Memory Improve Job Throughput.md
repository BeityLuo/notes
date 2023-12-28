# Can Far Memory Improve Job Throughput?

### 关键词

Fastswap

### 摘要

近年来，内存的需求快速增长，而内存管理技术的增长技术却较慢，因此，充分的可用内存正逐渐成为大计算机集群的瓶颈。一种解决方案是memory diaggregation，这种方案下，计算任务可以访问其他计算机上的内存。
本论文提出了faster swapping机制，以及一个基于远端内存的集群调度模块。

### 主要思想

- 设置一个或多个“内存服务器”，其他服务器可以使用内存服务器的内存。

- 设置了两个部分：
  - Fastswap：
  - Far memory-aware cluster scheduler==key==
  - 这两个合起来支持CFM(Cluster-wide Far Memory)的使用

### CFM: Cluster-wide Far Memory

- #### 实现方式

  - swapping：主动使用被动使用都行
  - cgroup：使用Linux的cgroup机制控制分配给一组进程的内存大小，当超过cgroup的限制，则将部分内存换出到far memory
  - RDMA：使用RDMA实现

### Fastswap机制

- 在内核态实现，改动了当页错误发生的时候Linux的处理流程

- #### 问题：

  1. linux在发生页错误的时候会预取好几个附近的页，而Infiniswap等可能会先拿到预取的页，再拿到想要的页
  2. RDMA完成操作会用中断通知CPU，引入额外的时延
  3. 当页错误的页被读入内存后，如果超过了cgroup的最大限制，就需要换出页面，这一过程带来时延。

- #### RDMA的设计

  - 提出了**Frontswap interface**，来区分一个一个页请求是否是**重要的**（预取的页是不重要的）
  - 为每个CPU分配两个queue pair，其中一个用来取最重要的页，另一个用来预取其他页
  - 

- #### Page Fault Handler的改动

  - 使得交换系统能够分别处理错误页和预取页
  2. 让交换系统先读错误的页并返回，再读其他预取窗口里的页。

- #### 对内存回收（memory reclaimer）的改动

  - 传统的cgroup中，一个页错误读入新的内存页后，如果超过了cgroup的内存限制，就立即执行内存回收，增加了延时
  - 改进：不立即执行内存回收，而是异步地让memory reclaimer执行。
  - 指定一个CPU进行memory reclaim工作，这个CPU并不是固定的
  - 为了避免reclaimer积压过多memory reclaim，当cgroup的memory超过一个阈值，就立即执行reclaim

### Far Memory-Aware Scheduler

- 如何调度任务的执行，需要考察所需内存、CPU等情况。当中央调度器将任务分配给一个守护程序，守护程序就创建一个cgroup并根据memory policy设置一个内存限制给他。

- 采用传统的调度策略：
  1. 当一个任务到来，被添加到一个等待队列
  2. 调度器尝试分配队首任务
  3. 随机顺序遍历节点，找到一个CPU数和内存满足任务要求的节点，分配给他
- 解决了两个问题：
  - 决定一个任务能否fit一个节点：CPU数满足，且剩余本地内存能载入（可能吧）任务，且有足够的远程内存保证任务执行
  2. 当一个任务fit一个节点，我们需要决定给他多少本地内存。

- 方法：
  - 设定每个任务远端内存占总内存的比例是个相对值（25%），而不是定值（2GB）
  - 设计了一个算法决定占多少比例