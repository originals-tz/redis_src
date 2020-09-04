# Learn Redis

阅读工具: SourceTrail

看redis的源码，首先需要明白redis的大致流程

redis是一个缓存服务器，既然是服务器就必然会有以下的流程

- 启动事件循环
- 接受连接，处理客户端请求
- 根据特定的协议解析请求，然后进行处理
- 将结果返回

那么只需要按照这些流程一步一步往下走，就能够摸清redis的大部分源码

其中，比较想要了解的就是

- redis的协议
- redis吞吐为何如此之高以至于说“网络带宽就是redis最大的瓶颈”
- 使用一个map来做一个kv缓存服务器是否能够做到和redis一样的性能？如果不能，是为什么，redis的数据结构性能有多高

## 目录

- [eventloop](eventloop.md)
- [command](cmd.md)