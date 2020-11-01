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

## Accept

服务器的入口就是接受客户端连接的accept，accept绑定的是`acceptTcpHandler`函数，当客户端请求建立连接，就会在那里被accept然后绑定对应的read handler(`void readQueryFromClient(connection *conn)`)来处理客户端发送过来的命令/请求

accpet之后，连接会对应一个client结构，里面绑定了一个redisDb，客户端的请求都会在那个db上执行

## readQueryFromClient

该函数是客户端所绑定的回调，当数据到达之后就会调用该函数，首先是读，然后是解析协议，执行命令，最后是返回结果，所对应的函数是

- 读：readQueryFromClient
- 解析：processCommand
- 执行：call
- 返回结果：会在执行指令的过程中调用addReply等函数

## call

抛开其他的流程，最关键的就是执行指令的call，如下

```c
    /* Call the command. */
    dirty = server.dirty;
    updateCachedTime(0);
    start = server.ustime;
    c->cmd->proc(c); //在这里调用cmd所绑定的handler
    duration = ustime()-start;
    dirty = server.dirty-dirty;
    if (dirty < 0) dirty = 0;
```

在call里面，使用`c->cmd->proc`进行指令的处理，那么这个proc又是什么时候进行绑定的呢，答案是在ProcessCommand的时候

```c
c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr); //找到该函数所绑定的handler

// lookupCommand函数
func lookupCommand(...)
{
	dictEntry *he;
	...
	he = dictFind(server.commands,key); //handler是从全局的server里面找的
	return he ? dictGetVal(he) : NULL;    
}
```

从上面看到，cmd是从server里面找到的，那么回头看就会发现，一开始就先在initServer初始化了servier.commands这个表

```c
    server.commands = dictCreate(&commandTableDictType,NULL);
    server.orig_commands = dictCreate(&commandTableDictType,NULL);
    populateCommandTable();
```

其中`populateCommandTable`就是将硬编码的指令解析出来放到commands中，硬编码的表如下，该表位于`server.c`的开头

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

可以看到，第二个参数就是该命令所绑定的handler

例如，客户端发送了一个get指令，那么就进入下面的handler中

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

在命令结束后，会将结果通过addReply放入对应的缓冲区中

那么什么时候将数据发送到客户端呢？在redis中，有一个beforesleep，每次进入eventloop前都会调用

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

当客户端的命令处理完后将结果写入缓冲区，redis会在beforesleep里面调用`handleClientsWithPendingWritesUsingThreads`等函数，将结果写到客户端中

那么整个大致的流程就是

- 服务器生成 cmd-handler的表
- 新建一些事件，如超时，accept等，然后进入等待
- 当连接到达，accept客户端连接，然后绑定对应的handler
- 当数据到来，使用那个handler读取客户端的请求，接着解析请求，得到cmd，然后从表中找到cmd所绑定的handler，进行处理
- 处理完毕后，进入beforesleep，将请求返回给客户端
