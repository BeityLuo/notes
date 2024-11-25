# Redis `setCommand`与rehash流程梳理

- 本文基于Redis 7.2编写
- 因Redis 7.4及之后，为了适配cluster模式，在db.c和dict.c之间增加了一层kvstore.c，为理解流程增加了不必要的代码

```C

// set指令的入口
t_string.c:
void setCommand(client *c) {
	robj *expire = NULL;
    int unit = UNIT_SECONDS;
    int flags = OBJ_NO_FLAGS;
	
    // 用于分析set指令的”GET，NX|XX，EX|PX”等参数，不常用
    if (parseExtendedStringArgumentsOrReply(c,&flags,&unit,&expire,COMMAND_SET) != C_OK) {
        return;
    }
	
    // 尝试压缩value
    c->argv[2] = tryObjectEncoding(c->argv[2]);
    setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
}

void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
    long long milliseconds = 0;
    int found = 0;
    int setkey_flags = 0;
    ...;
    // 搜索一次key对应的键值对，更新access time，ttl等信息
    found = (lookupKeyWrite(c->db,key) != NULL);
 	setkey_flags |= found ? SETKEY_ALREADY_EXIST : SETKEY_DOESNT_EXIST;
    ...;
    // setKey是重要函数，所有db中的键值对都应该通过setKey添加
    setKey(c,c->db,key,val,setkey_flags);
	...;
    // 更新expire字典
    if (expire) {
        setExpire(c,c->db,key,milliseconds);
        ...;
    }
    ...;
}

db.c:
void setKey(client *c, redisDb *db, robj *key, robj *val, int flags) {	
    if (键值对不存在) {
        dbAddInternal(db,key,val,0);
    } else if (是msetnx命令) {
        dbAddInternal(db,key,val,1);
    } else {
        // 键值对已存在
        dbSetValue(db,key,val,1,NULL);
    }
    incrRefCount(val);
    if (!(flags & SETKEY_KEEPTTL)) removeExpire(db,key);
    if (!(flags & SETKEY_NO_SIGNAL)) signalModifiedKey(c,db,key);
}

static dictEntry *dbAddInternal(redisDb *db, robj *key, robj *val, int update_if_existing) {
 	dictEntry *existing;
    // 根据key的哈希值，创建一个entry
    // 可能触发rehash
    dictEntry *de = dictAddRaw(db->dict, key->ptr, &existing);
    
    if (update_if_existing && existing) {
        dbSetValue(db, key, val, 1, existing);
        return;
    }
	
    // 将key拷贝一份，设置为entry的key
    dictSetKey(db->dict, de, sdsdup(key->ptr));
    initObjectLRUOrLFU(val);
    // 设置entry的val
    dictSetVal(db->dict, de, val);
    ...;
}
```



## dict.c的关键入口：``dictAddRaW()``

- dictAddRaw用来在键值对不存在的基础上，在dict中创建一个新的entry
- 可能会触发rehash操作

```C
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    /* 获取要插入的位置。如果已经存在，返回NULL */
    void *position = dictFindPositionForInsert(d, key, existing);
    if (!position) return NULL;

    /* Dup the key if necessary. */
    if (d->type->keyDup) key = d->type->keyDup(d, key);

    return dictInsertAtPosition(d, key, position);
}

void *dictFindPositionForInsert(dict *d, const void *key, dictEntry **existing) {
    unsigned long idx, table;
    dictEntry *he;
    uint64_t hash = dictHashKey(d, key);
    if (existing) *existing = NULL;
    
    // 如果正在做rehash，就往下做一步
    // dict.c的许多函数都会有这一句，用来逐渐将旧hashtable迁移到新hashtable
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 如果dict的负载过高，就触发rehash
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return NULL;
    
    // 搜索两张hash table
    for (table = 0; table <= 1; table++) {
        idx = hash & DICTHT_SIZE_MASK(d->ht_size_exp[table]); // 计算当前table下的hash
        he = d->ht_table[table][idx];
        while(he) { // Redis采用链表法处理冲突，因此遍历此位置下的链表
            void *he_key = dictGetKey(he);
            if (key == he_key || dictCompareKeys(d, key, he_key)) {
                // 已经存在这个key，直接返回
                if (existing) *existing = he;
                return NULL;
            }
            he = dictGetNext(he);
        }
        // 如果正在rehash，则扫描table[1]
        if (!dictIsRehashing(d)) break;
    }

    // 如果正在rehash，为了将新的键值对存到table[1]中，则一定返回table[1]中的位置
    dictEntry **bucket = &d->ht_table[dictIsRehashing(d) ? 1 : 0][idx];
    return bucket;
}
// 根据dictFindPositionForInsert的返回值，插入新的entry
dictEntry *dictInsertAtPosition(dict *d, void *key, void *position) {
    dictEntry **bucket = position; /* It's a bucket, but the API hides that. */
    dictEntry *entry;
    
    int htidx = dictIsRehashing(d) ? 1 : 0;
    assert(bucket >= &d->ht_table[htidx][0] &&
           bucket <= &d->ht_table[htidx][DICTHT_SIZE_MASK(d->ht_size_exp[htidx])]);
    size_t metasize = dictEntryMetadataSize(d);
    if (d->type->no_value) {
        // 有的键值对可能会没有value：未找到什么时候会走到这一分支
        assert(!metasize); /* Entry metadata + no value not supported. */
        if (d->type->keys_are_odd && !*bucket) {
            entry = key;
            assert(entryIsKey(entry));
        } else {
            entry = createEntryNoValue(key, *bucket);
        }
    } else {
        entry = zmalloc(sizeof(*entry) + metasize);
        assert(entryIsNormal(entry)); /* Check alignment of allocation */
        if (metasize > 0) {
            memset(dictEntryMetadata(entry), 0, metasize);
        }
        entry->key = key;
        entry->next = *bucket; // 将新的键值对放到链表开始
    }
    *bucket = entry;
    d->ht_used[htidx]++;
    
    return entry;
}

```

