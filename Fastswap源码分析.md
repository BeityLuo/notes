## Fastswap源码分析

### 1. Kernel修改的文件

- `include/linux/frontswap.h`
- `include/linux/swap.h`
- `mm/frontswap.c`
- `mm/memcontrol.c`
- `mm/page_io.c`
- `mm/swap_state.c`

- 主要工作：

  - 在`frontswap.c`中加入了`__frontswap_load_async, __frontswap_poll_load`，并修改了`frontswap.h`
  - 在`page_io.c`中加入了`swap_readpage_sync`，与`swap_readpage`进行区分，并修改了`swap.h`
  - 修改了`memcontrol.c`中的`high_work_func`与`memory_high_write`

  - 在`swap_state.c`加入了`read_swap_cache_sync`，与原来的`read_swap_cache_async`进行区分
  - 修改了`swap_state.c`的`swapin_readahead`，使用新加入的`read_swap_cache_sync`

- 调用

- ```mermaid
  graph TD
  
  AA(swap_state.c: read_swap_cache_async)
  A(page_io.c: swap_readpage)
  B(__frontswap_load_async,\n 通过EXPORT_SYMBOL暴露给内核\n但似乎没用到)
  
  
  H(frontswap.c: frontswap_shrink,\n 通过EXPORT_SYMBOL暴露给内核)
  I(swapfile.c: try_to_unuse)
  J(上接内核页错误处理)
  K(memory.c: do_swap_page)
  L(swap_state.c: swapin_readahead)
  M(swap_state.c: read_swap_cache_sync)
  N(page_io.c: swap_readpage_sync)
  O(frontswap.c: __frontswap_load,\n 通过EXPORT_SYMBOL暴露给内核\n但似乎没用到)
  L-->AA-->A-->B
  J-->K-->L-->M-->N-->O
  H-->I-->AA
  ```

- `frontswap_shrink`：

  - 外部程序调用`frontswap_shrink`，可以强制让frontswap管理的页面换出到可以被内核通过地址访问的内存中。	

- ```mermaid
  graph TD
  
  
  A("schedule_work_on(FASTSWAP_RECLAIM_CPU, &memcg->high_work);")
  B("memcontrol.c: try_charge")
  C("memcontrol.c: mem_cgroup_try_charge, mem_cgroup_charge_skmem, memcg_kmem_charge_memcg")
  C-->B-->A
  
  D("memcontrol.c: memory_high_show")
  D-->A
  
  
  Z(high_work_func)
  ```

- `high_work_func`在初始化`mem_cgroup_alloc`中被赋值给`memcg->high_work`

  - `memory_high_show`函数指针记录在`memcontrol.c: memory_files`这个列表中的一个元素
  - `memory_file`赋值给了`memory_cgrp_subsys.dfl_cftypes`
  - `memory_cgrp_subsys`用到的地方太多了


### 2. 内核页错误处理调用栈

```mermaid	
graph TD
O(do_page_fault, handle_page_fault等等)
A(memory.c: handle_mm_fault)
B(memory.c: __handle_mm_fault)
C(memory.c: handle_pte_fault)
O-->A-->B-->C

D(memory.c: do_swap_page)
C--!pte_present-->D-->return


```

### 3. driver调用方式

- 通过`frontswap_register_ops`函数，为frontswap注册了一个后端，frontswap根据注册时提供的函数指针完成内存块的操作。
- 使用了两个driver：fastswap.ko和fastswap_rdma.ko。
  - `fastswap_rdma.ko`：用来与远端内存进行交互
  - `fastswap.ko`：将rdma操作包装成能够被frontswap直接调用的接口。
- 为什么要用两个driver：因为还可以用`fastswap_dram.ko`。rdma与dram版本的driver对外暴露的符号是相同的，因此可以用fastswap进行复用，只需要选择想要的版本加载即可。

### 4. fastswap_rdma.c分析

- **初始化调用栈：**

```mermaid
graph TD

A(sswap_rdma_init_module)
B(sswap_rdma_create_ctrl)
C(sswap_rdma_init_queues)
D(sswap_rdma_init_queue)
E(init_completion\nrdma_create_id\nrdma_resolve_addr\nswwap_rdma_wait_for_cm)

A-->B-->C-->D-->E

F(sswap_rdma_recv_remotemr)
G(get_req_for_buf)
H(sswap_rdma_post_recv\nsswap_rdma_wait_completion)
A-->F-->G
F-->H




```

- **读写调用栈：**
  - sswap_rdma_read与sswap_rdma_write的工作流类似，都是先获取queue，然后poll_target，在创建req，最后用sswap_rdma_post_rdma发布req。

```mermaid
graph TD

A(sswap_rdma_read_async\nsswap_rdma_read_sync)

C(sswap_rdma_get_queue)
D(begin_read)
E(poll_target)
F(get_req_for_page)
G(sswap_rdma_post_rdma)

A-->C
A-->D-->E
D-->F
D-->G

I(sswap_rdma_write)
J(write_queue_add)
K(drain_queue)
I-->C
I-->J
I-->K

J-->E
J-->F
J-->G
```

- # **poll_load: TODO: 这是干嘛的？？**

- **sswap_rdma_cm_handler**: TODO: 这是干嘛的？

- 在`sswap_rdma_init_queue`中通过`queue->cm_id = rdma_create_id(&init_net, sswap_rdma_cm_handler, queue, RDMA_PS_TCP, IB_QPT_RC);`传递给queue->cm_id作为callback函数

```mermaid
    graph TD
    
    A(sswap_rdma_cm_handler)
    B1(sswap_rdma_addr_resolved)
    B2(sswap_rdma_route_resolved)
    B3(swap_rdma_conn_established)
    
    A-->B1
    A-->B2
    A-->B3
    
    C1(sswap_rdma_get_device)
    C2(sswap_rdma_create_queue_ib)
    C3("rdma_resolve_route(cma.c)")
    C4(sswap_rdma_destroy_queue_ib)
    D1("rdma_connect(cma.c)")
    B1-->C1
    B1-->C2
    B1-->C3
    B1-->C4
    B2-->D1
    B2-->C4
    
```

  - 

