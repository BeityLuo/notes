# Metastable Failures in the Wild论文笔记

### 摘要

- 分布式系统中的某一类failure被称为**Metastable Failure（亚稳态故障）**
- 本文研究了这种failure，结果分为三方面
  1. 亚稳态故障普遍存在
  2. 亚稳态故障是设备故障发生时常见的一种模式
  3. 扩展了前人的模型，能够更好地反应亚稳态故障
     - 区分了两类trigger和两类amplification mechanism

### 1. Introduction

- **Metastable Failure State定义**：永久性的工作过载，但是实际吞吐量很低。permanent overload with an ultra-low goodput

- **stable state定义**：即使有暂时的工作过载，也可以成功恢复到正常状态

- **vulnerable state定义**：？？？

- **amplification**：一个小的工作负载经过一系列机制，放大到了整个系统的大负载

- **sustaining effect**：正常的优化导致放大了工作负载

- **系统转变为亚稳态的条件**：系统当前处于vulnerable state，同时一个**trigger**导致了暂时的工作过载，进而引发了持续性影响**sustaining effect**。sustaining effect会使得计算机始终处于亚稳态，及时trigger已经消失。

- 亚稳态故障并不是硬件/软件的bug，而是系统正常运行下存在的问题

- #### 剩余章节的内容简介：

  - **Section 2**：研究证实了实际生产环境中亚稳态故障的普遍性，并占了最严重的故障的很大比重。
  - **Section 3**：提出了一个模型能更好地解释亚稳态故障如何发生，区分了两类trigger和两类amplification mechanism。
  - **Section 4**：研究了Twitter出现的一个新故障：垃圾回收机制导致了amplification
  - **Section 5**：三个示例程序，能够在线亚稳态故障

### 2. Metastability in the Wild

- Trigger中有45%是由于工程师错误造成的（比如错误的配置或代码部署）。35%是load spike。有45%的事故存在多个trigger