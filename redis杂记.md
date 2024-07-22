# Redis杂记

### 1. 持久化

- 两条指令：SAVE与BGSAVE（background save）

- SAVE：同步持久化，父进程亲力亲为，在这个过程中redis无法再相应请求
- BGSAVE：异步持久化，通过fork创建子进程，父进程继续响应请求，子进程进行持久化。
- 位置：`rdb.c::saveCommand, rdb.c::bgsaveCommand`
- BGSAVE执行失败的情况：
  - 当前正在有一个BGSAVE在执行
  - 有一个不是BGSAVE的持久化操作在执行（如AOF）
  - 当使用`BGSAVE SCHEDULE`命令时，如果当前正有一个aof在执行，redis会立即返回ok，并将这次bgsave找个合适的时间再执行。


### 2. 命令的入口

- 在`server.c`中定义了所有命令的函数原型，格式为`xxxCommand`，例如`bgsaveCommand`
- 关于某条命令的其他属性可以在`src/commands/xxx.json`中找到
- ***疑问：如何通过json文件调用命令***
- 

### 3. bgsaveCommand流程梳理

伪代码如下：

```c
void bgsaveCommand(client* c) 
{
	// 判断是否是BGSAVE SCHEDULE命令
    int schedule = 0;
    if (c->argc > 1)
        schedule = 1;
    
    // 根据当前状态初始化rdbSaveInfo
    rdbSaveInfo rsi, *rsiptr;
    rsiptr = rdbPopulateSaveInfo(&rsi);	
    
    // 
    if (server.in_exec || (hasActiveChildProcess() && schedule)) {
        // 将BGSAVE schedule一下
        server.rdb_bgsave_scheduled = 1;
    } else if (server.child_type == CHILD_TYPE_RDB ||
              (hasActiveChildProcess() && schedule == 0)) {
        // bgsave已经在运行了，或者（有一个子进程是active的状态，且没有用BGSAVE SCHEDULE命令）
        // 返回错误. active
        return error;
    }
    
    // 执行
    rdbSaveBackground(SLAVE_REQ_NONE,server.rdb_filename,rsiptr,RDBFLAGS_NONE)
}

int rdbSaveBackground(int req, char *filename, rdbSaveInfo *rsi, int rdbflags) 
{
    pid_t childpid = redisFork(CHILD_TYPE_RDB);
    
    if (childpid == 0) {
        // 子进程
        set a few things;
        if (rdbSave(req, filename,rsi,rdbflags) == C_OK) {
            通知父进程
        }
        退出子进程;
    } else {
        // 父进程
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_type =  RDB_SCHILD_TYPE_DISK;
    }
}

int rdbSave(int req, char *filename, rdbSaveInfo* rsi, int rdbflags)
{
    //TODO: 搞清楚这个事件产生的影响
    startSaving(RDBFLAGS_NONE); // 启动一个persistence start事件
    
    // 先写入到临时文件中，再将文件改名为filename，这样保证了在完成持久化之前，
    // 都不会改动filename这个文件。
    char tmpfile[256] = 一个临时文件名;
    rdbSaveInternel(req, tempfile, rsi, rdbflags); // 
   	rename(tempfile, filename); 
    
    // TODO
    if (fsyncFileDir(filename) != 0) {
        // 失败了
        stopSaving(0);
        return C_ERR;
    }
   
    stopSaving(1); // 启动一个persistence stop事件
    return C_OK;
}

static int rdbSaveInternal(int req, const char *filename, 
                           rdbSaveInfo *rsi, int rdbflags)
{
    rio rdb; // rdb中read、write、tell、flush等函数指针，以及其他用于io的属性。
    FILE *fp = fopen(filename, "w");
   
    根据fp初始化rdb(&rdb, fp); // rdb中保存了fp指针
    
    // TODO
    if (server.rdb_save_incremental_fsync) {
        rioSetAutoSync(&rdb,REDIS_AUTOSYNC_BYTES);
        if (!(rdbflags & RDBFLAGS_KEEP_CACHE)) rioSetReclaimCache(&rdb,1);
    }
    
    rdbSaveRio(req,&rdb,&error,rdbflags,rsi);
    
    // 保证数据被写入硬盘，不会停留在操作系统的写出缓冲上
    fflush(fp);
    fsync(filenp(fp));
    if (!(rdbflags & RDBFLAGS_KEEP_CACHE) && reclaimFilePageCache(fileno(fp), 0, 0) == -1) {
        //TODO
        serverLog(LL_NOTICE,"Unable to reclaim cache after saving RDB: %s", strerror(errno));
    }
    
    fclose(fp);
    return C_OK;
}

// 根据rio中的io方法，将数据库保存下来
int rdbSaveRio(int req, rio *rdb, int *error, int rdbflags, rdbSaveInfo *rsi)
{
   	save magic string of redis(rdb);
    rdbSaveInfoAuxFields;
    rdbSaveModulesAux;
    rdbSaveFunctions;
    
    for (j = 0; j < server.dbnum; j++) {
        // 保存数据库数据
        rdbSaveDb(rdb, j, rdbflags, &key_counter);
    }
    
    rdbSaveModulesAux;
    rdbSaveType(rdb, RDB_OPCODE_EOF); // save EOF code to file.
    save check sum;
}
```



### 4. 子进程

- **hasActiveChildProcess**：获得当前是否有有一个active的子进程。根据代码注释，子进程仅可能由一下三种途径产生：
  - RDB backgroud save
  - AOF rewrite
  - 被其他加载的module唤醒的子进程

- **父子进程通信：**
  - `server.child_info_pip`：子进程可以通过`sendChildInfoGeneric`函数向父进程的该域写入`child_info_data`，传递一些数据
  - `int child_info_pip[2]`： `child_info_pip[0]`用来读数据，`child_info_pip[1]`用来写数据

### 5. redisFork()

- 为redis创建子进程，为代码如下

  ```C
  // 这里的purpose是CHILD_TYPE_NONE/RDB/AOF/LDB/MODULE 
  int redisFork(int purpose)
  {
      if (purpose是exclusive的) { // RDB, AOF, MODULE是exclusive的
          if (已经有了子进程) // 违反了exclusive
              return 错误;
          打开server的child_info_pipe; // 子进程可以向父进程传递信息
      }
      
      int child_pid = fork();
      if (child_pid == 0) {
          // 子进程，进行一系列设置
          
          server.in_fork_child = purpose; // 表明该进程为fork产生的子进程
          设置处理child signal的handler;
          setOOMScoreAdj(CONFIG_OOM_BGCHILD); // 仅在linux生效，略去
          updateDictResizePolicy(); // 禁止dict.c进行resize，这是为了避免在COW下大量调整内存，导致大量拷贝
          dismissMemoryInChild(); // 关闭子进程中不使用的buffer，避免COW
          closeChildUnusedResourceAfterFork(); // 关闭子进程不用的resource
          /* 关闭pipe的read端。如果父进程崩溃了，子进程会得到一个写错误，从而退出 */
          // 为什么是写错误？？？？？
          if (server.child_info_pipe[0] != -1)
              close(server.child_info_pipe[0]);
      } else {
          // 父进程
          记录一系列统计数据;
          server.child_pid = childpid;
          server.child_type = purpose;
          
          updateDictResizePolicy(); // 禁止dict.c进行resize，这是为了避免在COW下大量调整内存，导致大量拷贝
          
          //TODO: 搞清楚这个事件产生的影响
         	moduleFireServerEvent(REDISMODULE_EVENT_FORK_CHILD,
                                REDISMODULE_SUBEVENT_FORK_CHILD_BORN,
                                NULL); // 发送一个事件，通知子进程已经生成了
      }
  }
  ```


### 6. 事件机制

TODO，module.c::11248line

### 7. Resis Cluster集群机制

- 集群的情况下，redis只有一个db

### 8. Redis测试

- redis自己有一个`redis-benchmark.sh`脚本，使用方法如下↓

- > https://developer.aliyun.com/article/783643

### 9. Redis的DB

- redis默认建立16个db，默认使用第0个。redis的db应当理解为一种“命名空间”，而不是一个单独的数据库。
- 不同的db之间键值不通，需要用户自己记录哪个db对应什么功能
- save、flushall等命令会针对所有的db进行操作，这点与传统关系型数据库区别较大
- 不同的应用之间应该创建不同的redis实例，而不是使用同一个实例的不同db。
- 集群模式下，只有一个db0.

### 10. Rio的创建

- 例如，在rdb.c中，在`rdbSaveInternal`函数中调用`rio.c::rioInitWithFile`创建

- ```C
  void rio.c::rioInitWithFile(rio *r, FILE *fp) {
      *r = rioFileIO; // static的变量。
      r->io.file.fp = fp;
      r->io.file.buffered = 0;
      r->io.file.autosync = 0;
      r->io.file.reclaim_cache = 0;
  }
  static const rio rioFileIO = {
      rioFileRead,
      rioFileWrite,
      rioFileTell,
      rioFileFlush,
      NULL,           /* update_checksum */
      0,              /* current checksum */
      0,              /* flags */
      0,              /* bytes read or written */
      0,              /* read/write chunk size */
      { { NULL, 0 } } /* union for io-specific vars */
  };
  ```

- 可以看到有一个


  ### 11. 设置redis大小

  - 想要设置redis的总大小为2GB，需要修改`redis.conf`中的`maxmemory`字段
  - 同时可以指定相应的淘汰政策：当超出了maxmemory时，是按照某种政策淘汰一些键值对，还是直接报错？

### 12. autosync机制

- 通过在`redis.config`中设置`rdb-save-incremental-fsync`和`aof-rewrite-incremental-fsync`，可以为rdb与aof文件保存开启autosync机制
- 开启后，在写入文件时，redis将**每fwrite 4MB就fflush一次**，并且通过sync_file_range将本次的4MB异步写入磁盘，然后再调用sync_file_range等待上一个4MB写入完成，再重新开始fwrite新的4MB。
  - 在IO queue不满的情况下，第一个sync_file_range不会等待磁盘IO，而是发出IO请求后立即返回。如果满了，就还是要等待队列空出来再发IO请求。
  - 第二个sync_file_range则会等待磁盘IO完成对上一个4MB的写入才会返回。消耗了大约30%的时间。
- 这样保证了在fwrite的同时能够让IO设备异步的向磁盘写入，减少磁盘IO阻塞的时间。
- fwrite消耗大概23%左右（这个测不准，因为调用次数过多），fflush占比几乎为0，第一个sync_file_range消耗6.9%，第二个消耗30.1%。
- 因此，等待磁盘IO的时间大概就是这两个sync_file_range的时间，37%左右

### 13. server.c::main梳理

- 每次启动都要随机生成一个种子用来作为dict的哈希函数的种子

### 14. 字典结构

- 详解：[【深度解析】Redis 字典 dict数据结构 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/382322388)

- **字典`struct dict`**

  - ```C
    typedef struct dict {
        dictType *type; // 根据数据类型不同的一组函数指针
        void *privdata; // 根据数据类型不同的私有数据
        dictht ht[2];   // 哈希表
        int rehashidx;  // rehash 索引。 当 rehash 不在进行时，值为 -1
    } dict;
    ```

  - 

  - 每个字典有两个哈希表，平时只用0号哈希表

  - 当需要rehash时才会创建1号哈希表（大小翻倍或减半）

  - **渐进式rehash：**不会一次性rehash，而是每次对键值对操作时rehash，将rehash的开销分摊到每次指令中

- **哈希表`struct dictht`：**(新版redis已经没有这玩意了，整合到`struct dict`里了)

  - ```C
    struct dictht {
        dictEntry **table;      // 哈希表数组
        unsigned long size;     // 哈希表大小
        unsigned long sizemask; // 用于计算索引值，等于 size - 1
        unsigned long used;     // 该哈希表已有节点的数量
    };
    ```

  - 采用链式方法解决hash冲突

- **哈希表项：`struct dictEntry`**

  - ```C
    struct dictEntry {
        void *key;
        union {
            void *val;
            uint64_t u64;
            int64_t s64;
        } v;
        struct dictEntry *next; // 指向下个哈希表节点，形成链表
    };
    ```




### 15. 数据结构

- 全局有一个`struct redisServer server`，`server`中保存了一个`redisDb *db;`

- `redisDb *db`是一个数组，总共有16个`redisDb`

- ```C
  typedef struct redisDb {
      dict *dict;                 /* The keyspace for this DB */
      dict *expires;              /* Timeout of keys with a timeout set */
      dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
      dict *blocking_keys_unblock_on_nokey;   /* Keys with clients waiting for
                                               * data, and should be unblocked if key is deleted (XREADEDGROUP).
                                               * This is a subset of blocking_keys*/
      dict *ready_keys;           /* Blocked keys that received a PUSH */
      dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
      int id;                     /* Database ID */
      long long avg_ttl;          /* Average TTL, just for stats */
      unsigned long expires_cursor; /* Cursor of the active expire cycle. */
      list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
      clusterSlotToKeyMapping *slots_to_keys; /* Array of slots to keys. Only used in cluster mode (db 0). */
  } redisDb;
  ```



### 16. rdb文件结构

- 版本：7.2

- 最开头是一些magic number，之后的每一段数据都是data_type+data_value的形式

- ```
  ----------------------------------------
  magic number
  ----------------------------------------
  RDB_OPCODE_AUX // 此段重复多次，来自rdbSaveRio
  Auxiliary field value  // Redis version, creation time, used memory, ...
  ----------------------------------------
  RDB_OPCODE_MODULE_AUX // 此段可能不存在，也可能重复多次，来自rdbSaveModulesAux
  Modules Auxiliary field value // Redis module的一些信息
  ----------------------------------------
  RDB_OPCODE_FUNCTION2 // 此段重复多次，来自rdbSaveFunctions
  Functions Code
  ----------------------------------------
  RDB_OPCODE_SELECTDB // 代表着开始保存一个db
  DB ID
  ----------------------------------------
  RDB_OPCODE_RESIZEDB // 保存db_size和expire_size
  db_size
  expire_size
  ----------------------------------------
  // 开始保存一个键值对。先保存expiretime，lru，lfu信息（如果有）
  RDB_OPCODE_EXPIRETIME_MS, RDB_OPCODE_IDLE, RDB_OPCODE_FREQ
  value
  ----------------------------------------
  RDB_TYPE_XXXXXX // 保存这个键值对的类型
  Key String // 键值对的key
  Value // 每种类型的value有不同的形式
  ----------------------------------------
  ... // 其他的键值对
  ----------------------------------------
  RDB_OPCODE_SELECTDB // 下一个db
  DB ID
  ----------------------------------------
  ...
  ----------------------------------------
  RDB_OPCODE_MODULE_AUX // 此段可能不存在，也可能重复多次，来自rdbSaveModulesAux
  Modules Auxiliary field value // Redis module的一些信息
  ----------------------------------------
  RDB_OPCODE_EOF // 文件结束
  Checksum
  ----------------------------------------
  ```

- 

### 17. Redis的Bug

- 当数据包较大时，readQueryFromClient的最后一次，nread
