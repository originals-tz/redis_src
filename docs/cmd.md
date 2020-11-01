---
title : Redis Command
---
# Redis Command

在eventloop的那节，我们已经看到，所有命令的handler都存放在`server.c`中的`redisCommandTable`中

因此，可以照着那里面所绑定的handler看下去，这样一来，就能知道每一个命令是如何处理的

## db

redis的命令都是操作在`redisDb`这个结构上的，该结构在`initServer`的时候被初始化到全局的server上，每次有新连接过来，就会绑定该db

```c
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```

可以看到，存储数据的结构就是`dict`，也就是一个map，类似于`std::map<std::string, void*>`,知道怎么存之后，一切都简单起来，我们先抛开一些高级的特性，例如超时等，简单看下各个操作

## Get

get所绑定的是`getCommand`，整个流程也很简单，查找，返回

## Set

set则比较复杂点，从顶端的调用开始，首先要判断这个key是否存在，如果不存在，则添加，如果存在，则覆盖

```c
void genericSetKey(client *c, redisDb *db, robj *key, robj *val, int keepttl, int signal) {
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val);
    } else {
        dbOverwrite(db,key,val);
    }
    incrRefCount(val);
    if (!keepttl) removeExpire(db,key);
    if (signal) signalModifiedKey(c,db,key);
}
```

那为何要判断是否存在呢，不管三七二十一直接覆盖不就完事了吗？

当一个结果一样的行为被分成了两个：add和overwrite，那么就说明这两个操作可能会存在性能上的差异抑或是，前提(key是否存在)对整个系统存在一个整体的影响，因此需要两种不同的行为来分别处理

首先看判断函数`lookupKeyWrite`

```c
robj *lookupKeyWriteWithFlags(redisDb *db, robj *key, int flags) {
    expireIfNeeded(db,key);
    return lookupKey(db,key,flags);
}

robj *lookupKeyWrite(redisDb *db, robj *key) {
    return lookupKeyWriteWithFlags(db, key, LOOKUP_NONE);
}
```

可以看到，在访问key的时候，会有一个`expireIfNeeded`的行为，这个是一个惰性删除过期key的行为，意图是让lookupKey不会接触到过期的key

当key不存在的时候，直接添加一个key，也就是直接往dict里面添加东西

当key存在的时候，则会先拿到当前存在的key，将其value进行替换，而旧的value，则由策略来决定是否删除，因为，删除一个占用空间很大的value是非常消耗性能的(在eventloop里面我们也看到，redis处理客户端请求是单线程的，因此如果一个指令执行很久，那么就会阻塞到其他的客户端请求)，**这就是add和overwrite的不同之处**

```c
void dbOverwrite(redisDb *db, robj *key, robj *val) {
    dictEntry *de = dictFind(db->dict,key->ptr);

    serverAssertWithInfo(NULL,key,de != NULL);
    dictEntry auxentry = *de;
    robj *old = dictGetVal(de);
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        val->lru = old->lru;
    }
    dictSetVal(db->dict, de, val);

    // 惰性释放
    if (server.lazyfree_lazy_server_del) {
        freeObjAsync(old);
        dictSetVal(db->dict, &auxentry, NULL);
    }

    dictFreeVal(db->dict, &auxentry);
}

```
