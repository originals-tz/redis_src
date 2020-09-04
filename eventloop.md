# EventLoop

eventloop是Linux服务器的核心，redis也不例外

在源码之前，可以根据经验猜测一下redis是如何使用eventloop的

- 创建一个监听事件，绑定对应的处理函数，用于处理客户端连接
- 当连接建立，就为客户端生成一个新的事件，然后绑定对应的处理函数，由于redis是一个缓存服务器，那么客户端所做的事情无非就两类，修改与读取



redis的eventloop就在server.c中的main函数里面被启动了，如下

```c
int main()
{
	...
    aeMain(server.el);
    aeDeleteEventLoop(server.el);
    return 0;
}

void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```

从入参可以知道，一些服务器的信息，例如ip，端口之类的，肯定是绑定在里面，果不其然，在启动之前，会调用一个initServer来初始化一些基本的信息

## InitServer

在redis中，server是作为全局变量定义的

```c
/* Global vars */
struct redisServer server; /* Server global state */
```

在函数InitServer中，会找到服务器的标准套路：创建eventloop，创建事件，绑定处理函数，注册到eventloop中，如下

```c
if (aeApiCreate(eventLoop) == -1) goto err;   
...
    /* Create an event handler for accepting new connections in TCP and Unix
     * domain sockets. */
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }

```

accept绑定的是`acceptTcpHandler`函数，当客户端请求建立连接，就会在那里被accept然后绑定对应的read handler(`void readQueryFromClient(connection *conn)`)来处理客户端发送过来的命令/请求

## readQueryFromClient

当数据到达之后，首先是读，然后是解析，执行，最后是返回结果

- 读：readQueryFromClient
- 解析：processCommand
- 执行：call
- 返回结果

## call

```c
    /* Call the command. */
    dirty = server.dirty;
    updateCachedTime(0);
    start = server.ustime;
    c->cmd->proc(c);
    duration = ustime()-start;
    dirty = server.dirty-dirty;
    if (dirty < 0) dirty = 0;
```

在call里面，使用`c->cmd->proc`进行指令的执行，那么这个proc又是什么时候进行绑定的呢，答案是在ProcessCommand的时候

```c
c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);

// lookupCommand函数
func lookupCommand(...)
{
	dictEntry *he;
	...
	he = dictFind(server.commands,key);
	return he ? dictGetVal(he) : NULL;    
}
```

一开始，先在initServer初始化servier.commands这个表

```c
    server.commands = dictCreate(&commandTableDictType,NULL);
    server.orig_commands = dictCreate(&commandTableDictType,NULL);
    populateCommandTable();
```

其中populateCommandTable就是将硬编码的指令解析出来放到commands中，硬编码的表如下

```c

struct redisCommand redisCommandTable[] = {
    {"module",moduleCommand,-2,
     "admin no-script",
     0,NULL,0,0,0,0,0,0},

    {"get",getCommand,2,
     "read-only fast @string",
     0,NULL,1,1,1,0,0,0},

    /* Note that we can't flag set as fast, since it may perform an
     * implicit DEL of a large key. */
    {"set",setCommand,-3,
     "write use-memory @string",
     0,NULL,1,1,1,0,0,0},

    {"setnx",setnxCommand,3,
     "write use-memory fast @string",
     0,NULL,1,1,1,0,0,0},

    {"setex",setexCommand,4,
     "write use-memory @string",
     0,NULL,1,1,1,0,0,0},
     ...
};
```

其中redisCommand的结构如下

```c
struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;
    char *sflags;   /* Flags as string representation, one char per flag. */
    uint64_t flags; /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    long long microseconds, calls;
    int id;     /* Command ID. This is a progressive ID starting from 0 that
                   is assigned at runtime, and is used in order to check
                   ACLs. A connection is able to execute a given command if
                   the user associated to the connection has this command
                   bit set in the bitmap of allowed commands. */
};
```

可以看到，第二个参数就是该命令所绑定的handler，整个流程就是

服务器生成 cmd-handler的表，accept客户端连接，读取客户端的请求，解析请求，然后找到对应的cmd所绑定的handler，进行处理，例如，客户端发送了一个get指令，那么就进入下面的handler中

```c
int getGenericCommand(client *c) {
    robj *o;

    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.null[c->resp])) == NULL)
        return C_OK;

    if (o->type != OBJ_STRING) {
        addReply(c,shared.wrongtypeerr);
        return C_ERR;
    } else {
        addReplyBulk(c,o);
        return C_OK;
    }
}
```

在命令结束后，会将结果通关addReply放入对应的缓冲区中

在redis中，有一个beforesleep，每次进入eventloop前都会调用

```
/* This function gets called every time Redis is entering the
 * main loop of the event driven library, that is, before to sleep
 * for ready file descriptors.
 *
 * Note: This function is (currently) called from two functions:
 * 1. aeMain - The main server loop
 * 2. processEventsWhileBlocked - Process clients during RDB/AOF load
 *
 * If it was called from processEventsWhileBlocked we don't want
 * to perform all actions (For example, we don't want to expire
 * keys), but we do need to perform some actions.
 *
 * The most important is freeClientsInAsyncFreeQueue but we also
 * call some other low-risk functions. */
void beforeSleep(struct aeEventLoop *eventLoop);
```

当命令处理完，将结果写入缓冲区，redis会在before sleep的时候将数据返回给客户端，在beforesleep里面会调用handleClientsWithPendingWritesUsingThreads，将结果写到客户端中