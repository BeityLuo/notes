# ptmalloc3

`ptmalloc3`在内存效率上略逊于`ptmalloc2`，因此`glib2.3.x`使用的是`ptmalloc2`.

### 初始化过程

- 第一次调用malloc时，会调用`malloc_hook_ini`函数

- ```C
  /* main_arena.buf_内容（对齐的情况下,）：
       | struct malloc_chunk | struct malloc_state(also "mspace"): top = top chunk | top chunk: head = 8 | chunk: head = TOP_FOT_SIZE |
       |        8 bytes      |                                                     |         8 bytes     |    TOP_FOT_SIZE btes       |
       | segment->base指向此，segment->size是buf的长度 |
    */
  static void* malloc_hook_ini(size_t sz, const void * caller) // sz是第一次调用malloc传递的参数size
  {
    __malloc_hook = NULL; // 这样第二次就不会再调用了。
    ptmalloc_init();
    return public_mALLOc(sz);
  }
  
  static void ptmalloc_init(void)
  {
    const char* s;
    int secure = 0;
    void *mspace;
    ...
    main_arena.next = &main_arena;
    // arena的从MSPACE_OFFSET开始的地方是mspace
    // MSAPCE_OFFSET是main_arena.buf_的offset对MALLOC_ALIGN(8)向上对齐
    //   实际情况中，一般都是对齐的，所以就是main_arena.buf_的地址
    mspace = create_mspace_with_base((char*)&main_arena + MSPACE_OFFSET,
  				   sizeof(main_arena) - MSPACE_OFFSET,
  				   0);
    assert(mspace == arena_to_mspace(&main_arena));
    ...
  }
  
  mspace create_mspace_with_base(void* base, size_t capacity, int locked) {
    // base是arena.buf_，capacity是arena.buf_的大小（buf_地址对齐时）
    mstate m = 0;
    size_t msize = pad_request(sizeof(struct malloc_state));
    init_mparams(); /* Ensure pagesize etc initialized */
  
    if (capacity > msize + TOP_FOOT_SIZE &&
        capacity < (size_t) -(msize + TOP_FOOT_SIZE + mparams.page_size)) {
      m = init_user_mstate((char*)base, capacity);
      m->seg.sflags = EXTERN_BIT; // 将这块内存设置为不可释放
      set_lock(m, locked);
    }
    return (mspace)m; // 返回值是init_user_mstate的返回值
  }
  
  static mstate init_user_mstate(char* tbase, size_t tsize) {
    size_t msize = pad_request(sizeof(struct malloc_state));
    mchunkptr mn;
    mchunkptr msp = align_as_chunk(tbase); // msp指向第一个chunk. 对齐的情况下msp == tbase
    mstate m = (mstate)(chunk2mem(msp)); // m是第一个chunk的mstate
    memset(m, 0, msize);
    INITIAL_LOCK(&m->mutex);
    msp->head = (msize|PINUSE_BIT|CINUSE_BIT); // head是说这个chunk的总大小（包括chunk的四个变量），是8bytes对齐的。因此空了三个bit可以存信息
                                               // PINUSE_BIT是上一个chunk已被用，CINUSE_BIT是本chunk已被用
    m->seg.base = m->least_addr = tbase;
    m->seg.size = m->footprint = m->max_footprint = tsize;
    ...
    init_bins(m);
    mn = next_chunk(mem2chunk(m));
    init_top(m, mn, (size_t)((tbase + tsize) - (char*)mn) - TOP_FOOT_SIZE); //TODO: 在arena的最后空间初始化top
    check_top_chunk(m, m->top);
    return m;
  }
  
  ```

- 

### 申请新的内存块流程

- 当`mspace_malloc`中判断，small_bin，tree_bin，top chunk中的空间都不够了，就调用`sys_alloc`分配空间

  - #### 对于过大的分配请求：

    - “过大”的临界值是`mparams.mmap_threshold`
    - 调用`mmap_alloc`函数分配一大块空间，设置chunk信息后直接返回
    - 不会记录这块空间的管理信息，不会与其他的chunk产生任何联系
    - 即：分配与释放都是独立的，不对其他chunk产生影响

  - #### 如果定义了`MORECORE_CONTIGUOUS`

    - ...

  - #### 如果定义了`HAVE_MMAP`

    - 先将需要分配的空间大小对齐`mparams.granularity`（向上取整），然后用`mmap`分配
    - 调用`init_top`将分配的空间设置为新的top

### 释放内存流程

- 



### 内存结构

```
| main arena:  | chunk head   | struct malloc_state(also "mspace") | top chunk head                              | chunk head                 | some data |
|   ...        |   prev_size  |  small_bin, tree_bin               |  prev_size: headsize + sizeof(malloc_state) |   prev_size = headsize + 8 |           |
|   next arena |   size       |  top_chunk, top chunk size         |  size: 8                                    |   size: TOP_FOOT_SIZE      |           |
                              |  | segment_addr, size, next segment|                                                                            ^
                                 | |                   |                                                                                        | 
                                 | |                   |________________________________________________________________________________________|
_________________________________|_|
|
v
| chunk head |    data   |
|                        |
|                        |
|        1 TB            |


```

