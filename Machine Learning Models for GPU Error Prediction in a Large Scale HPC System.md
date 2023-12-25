---
Publisher：DSN
Year: 2018
CCF: B
Institution: 美国东北大学
---



# Machine Learning Models for GPU Error Prediction in a Large Scale HPC System

### 摘要

- 发表在2018DSN，B类会议，美国东北大学
- **本文不包含深度学习方法**
- 针对“**利用GPU进行高性能计算（HPC）时可能发生错误这一场景”**干了两件事：
  - 通过六个月的真实大规模HPC数据，分析了导致GPU错误的trigger
  - 利用数据的时间关系和**空间关系**，采用机器学习方法**预测GPU错误的出现**

### 1. Introduction

- 过去的工作表明，GPU的**电源和冷却系统**与GPU错误有密切关系，但是没有明确指出引发错误的条件
- **本文的主要方向：**研究 Titan 超级计算机上**工作负载/温度/功耗和 GPU soft error**之间的关系
- 通过将feature在时间和空间维度分类，系统性的选取feature
- 最好的GBDT模型的F1为0.81

### 2. Background

- Titan有18688个GPU
- NVIDIA给出了XID errors，**本文仅关注GPU soft error中的single bit error（SBE）**。因为这种错误在Titan中发生的最频繁
- 通过nvidia-smi收集GPU错误
- 对每个GPU节点，每分钟收集一次信息（如core-hours, maximum memory consumption, total memory consumption per application, temperature....)

### 3. GPU ERROR CHARACTERIZATION

- 多种导致GPU soft error的原因：宇宙射线撞击、电压波动、温度升高、制造缺陷和复杂的工作负载硬件交互等

#### SBE Offender Nodes

- Titan有25 * 8个cabinet，而**GPU error在每个cabinet的分布是不均匀的**，有的多有的少

#### Application

- **cabinet上application分布不均**：会被SBE错误影响的application，在cabinet的分布也是不均匀的
- **不同的application分布不均**：小部分app受到了大部分的SBE错误影响（受到运行时间、使用资源多少的影响）
- 

#### Temperature and Power Consumption

- 分析了温度和功耗对GPU错误的影响
- 不同的cabinet的温度和功耗分布不均匀
- **不考虑时间维度（取平均值）：cabinet的温度/功耗与是否容易出现GPU error的相关性较小**
- 考虑时间维度：容易出现SBE的cabinet，在发生SBE错误时温度较高、功耗较大
- 考虑空间维度：

#### 4. Overview of the OVERVIEW OF THE METHODOLOGY

- 本文接下来的工作分为：
  - Feature selection
  - Function discovery
  - Analysis of the learned function
- 本文的方法不包含深度学习

### 5. FEATURE SELECTION

- **本文使用的feature：**

  - 时间领域feature：

    - Application：

      

      - **application binary name**, **total execution time (from past runs)**
      - GPU resource utilization. includes the **aggregate GPU core time**, **aggregate GPU memory**, and **maximum GPU memory**
      - use **the application name that ran before this execution** to account for post-effects of an application run.
      - **有趣的点：使用在当前application之前的application name，从而将post-effect考虑在内**

    - Temperature, Power

      - 当前应用的温度的平均值和标准差
      - 当前应用两次测量之间温度差值的平均值和标准差
      - 在执行当前应用之前（5min, 15min, 30min, 60min)，温度序列的平均值和标准偏差以及同一节点上连续两分钟之间的温差。
      - 同理，对于功率采用同样方法

  - 空间领域feature

    - Node location
      - 
    - Temperature/Power consumption
      - 温度和功耗的平均值、标准差
      - 同一node CPU的温度的两次测量间差值的平均值、标准差
      - 同一slot GPU的温度的两次测量间差值的平均值、标准差
    - SBE history
      - 每个node的error总数
      - 整个机器的error总数
      - 某一application近期的SEB rate
      - 分配给某一application近期的node