# RDMA杂记

### 1. SGE

- RDMA编程中，SGL(Scatter/Gather List)是最基本的数据组织形式。 SGL是一个数组，该数组中的元素被称之为SGE(Scatter/Gather Element)，**每一个SGE就是一个Data Segment(数据段)**。RDMA支持Scatter/Gather操作，具体来讲就是RDMA可以支持一个连续的Buffer空间，进行Scatter分散到多个目的主机的不连续的Buffer空间。Gather指的就是多个不连续的Buffer空间，可以Gather到目的主机的一段连续的Buffer空间。

### 2. 用户态rdma工作流

- ```mermaid
  graph TD
  
  A((start))
  B(setup ib)
  C(resigter_memory_region)
  D(post_read/write/send/received/_request)
  E(poll_completion)
  F((end))
  
  A-->B-->C-->D-->E-->F
  
  BA(create_context)
  BB(ibv_alloc_pd)
  BC(ibv_create_cq)
  BD(create_queue_pair)
  BE(change_queue_pair_state_to_INT)
  BF(change_queue_pair_state_to_RTR)
  BG(change_queue_pair_state_to_RTS)
  B-.->BA-->BB-->BC-->BD-->BE-->BF-->BG-.->B
  
  DA(create work request)
  DB(create send sge list)
  DC(copy ibv_mr to sge list)
  DD(setup work request)
  DE(ibv_post_recv/send)
  D-.->DA-->DB-->DC-->DD-->DE-.->D
  ```

### 3. 内核态rdma工作流

```mermaid
graph TD

A((start))
B(init)
C


C(get_queue)
D(begin_read)
DD(write_queue_add)
Z((end))
A-->B-->C--"read"-->D-->Z
C--"write "-->DD-->Z

B1(kmem_cache_create)
B2(ib_register_client)
B3(create_ctrl)
B4(recv_remote_memory_region)
B-.->B1-->B2-->B3-->B4-.->B

B31(alloc ctrl)
B32(alloc and init queues)
B3-.->B31-->B32-.->B3

B41("create rdma_req(similar to work_request)")
B42("sswap_rdma_post_recv")
B43("sswap_rdma_wait_completion")
B44("sswap_rdma_recv_remotemr_done: unmap dma, complete_all")

B4-.->B41-->B42-->B43--"异步执行"-->B44-.->B4

```

```mermaid
graph TD

A(begin_read)
B()
```



- 内核态工作流特点

### 4. rdma_cm: rdma connection manager下的用户态工作流

- rdma_cm是一个主要用于管理rdma通信连接的过程，提供了建立、断开连接和一些基础操作的简单抽象，让rdma更加简单
- 避免了只使用ibverbs需要交换的global id等信息
- 详见：https://zhuanlan.zhihu.com/p/494826608
- CM不依赖于传统以太网卡，可以在infiniband网卡上实现信息交互（比如QPN、Virtual Address和Remote Key）。既是一种协议（Communication Management Protocol），也可以是编程时的API
- 兼容性不好，不怎么用

### 5. 一些坑

- 有时候qp无法切换到RTR状态，是因为没有用GID
- 申请GID的时候，需要注意参数ib_port和GID index的值，必须和切换到RTR状态时指定的`ah_attr.port_num, ah_attr.grh.sgid_index`相同
- 两台机器互相跑的时候，需要保证GID index相同
- 使用TCP多次传输数据时，要保证client端在server端listen之后再传输，不然就是connection refused
