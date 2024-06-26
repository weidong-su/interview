# 在线redis环境：https://try.redis.io/


# Redis怎么实现分布式锁？

Redis实现分布式锁主要依赖于其提供的setnx（set if not exist）命令，这个命令可以在指定的key不存在时，为key设置指定的值。如果key已经存在，则setnx不做任何动作。利用这个特性，我们可以将key视为锁，value视为锁的持有者。

以下是一个简单的Redis分布式锁的实现步骤
1. 获取锁：
使用setnx命令尝试获取锁。具体做法是，将锁的key设置为一个随机值（通常是UUID），并设置过期时间（防止锁持有者崩溃导致锁无法释放）。如果设置成功，则获取到锁；如果设置失败（即key已经存在），则表示锁已被其他客户端持有，此时需要等待或重试。

```
SET lock_key random_value NX PX 30000
```

其中，lock_key是锁的key，random_value是锁的value（通常是UUID），NX表示只有在key不存在时才设置，PX 30000表示锁的过期时间为30秒。
2. 操作共享资源：

获取到锁后，客户端可以安全地操作共享资源。需要注意的是，由于网络延迟或客户端崩溃等原因，锁的持有者可能无法及时释放锁。因此，在设置锁时，我们需要为其设置一个合理的过期时间，以避免死锁。
3. 释放锁：

当客户端完成对共享资源的操作后，需要释放锁。释放锁时，需要确保只释放自己持有的锁，防止误解锁。具体的做法是，使用Lua脚本先判断锁的value是否为自己的随机值，如果是，则删除该锁。

```
if redis.call("get", KEYS[1]) == ARGV[1] then  
    return redis.call("del", KEYS[1])  
else  
    return 0  
end
```
其中，KEYS[1]是锁的key，ARGV[1]是锁的value（即获取锁时设置的随机值）。这个Lua脚本保证了原子性，即判断锁和删除锁的操作是一个不可分割的整体。

这就是Redis实现分布式锁的基本思路。在实际应用中，还需要考虑一些其他因素，如锁的重入性、锁的公平性、锁的粒度等。同时，由于Redis的分布式特性，还需要考虑如何保证锁的正确性和一致性。例如，当Redis集群发生脑裂时，可能会出现多个客户端同时持有同一个锁的情况，这时就需要使用RedLock等算法来确保锁的正确性。



# 如何保证缓存与数据库双写时的数据一致性

> https://xiaolincoding.com/redis/architecture/mysql_redis_consistency.html

在Redis缓存与数据库双写场景中，数据一致性是一个重要的问题。以下是一些策略和方法，用于确保两者之间的数据一致性：

1. 延时双删策略：

当在数据库中更新一个数据项时，首先删除Redis中对应的缓存项。
然后更新数据库。
等待一段短暂的时间（比如几百毫秒），再次删除Redis中的缓存项。
这个策略的目的是为了解决在更新数据库和删除Redis缓存之间的时间差中，如果Redis缓存被其他请求读取并设置了新的缓存值，那么这个新值将不会被后续的数据库更新所覆盖。通过再次删除Redis缓存，可以确保后续请求会重新从数据库中读取数据。

2. 先更新数据库，再删除缓存（Cache Aside 旁路缓存策略）：

当需要更新数据时，首先更新数据库。
然后删除Redis中对应的缓存项。
这种策略的一个潜在问题是，如果在更新数据库和删除Redis缓存之间的时间差中，其他请求读取了旧的数据库值并设置了Redis缓存，那么这个旧值将一直存在于Redis中，直到它因为某种原因（如过期）被删除。但是，在大多数情况下，这种策略已经足够好，因为新数据最终会被读取并覆盖旧的缓存值。

3. 使用消息队列确保顺序：

当需要更新数据时，首先将更新操作放入消息队列。
消费者从消息队列中取出更新操作，先更新数据库，再删除Redis缓存。
这种方法可以确保更新和删除操作的顺序性，从而避免潜在的数据不一致问题。

4. 分布式锁：

在更新数据库和删除Redis缓存时，使用分布式锁来确保操作的原子性。
这可以防止多个请求同时更新数据库和Redis缓存，从而避免数据不一致。但需要注意的是，分布式锁的使用可能会引入性能瓶颈和死锁问题，因此需要谨慎使用。

5. 读时修复：

当从Redis读取数据时发现数据不一致，重新从数据库读取数据并更新Redis缓存。
这种方法适用于对一致性要求不是特别严格的应用场景。它可以在读取数据时自动修复不一致问题，但可能会增加一些额外的读取成本。

6. 合理设置缓存过期时间：

根据业务需求和数据更新频率，合理设置Redis缓存的过期时间。
这可以确保即使出现数据不一致的情况，不一致的数据也不会在Redis中停留太长时间。

综上所述，确保Redis缓存与数据库双写时的数据一致性需要根据具体业务场景和需求来选择合适的策略和方法。在实际应用中，可能需要结合多种策略来达到最佳效果。


# Redis是什么？有哪些功能？
Redis是一个数据库，不过与传统数据库不同的是Redis的数据库是存在内存中，所以读写速度非常快，因此 Redis被广泛应用于缓存方向。

除此之外，Redis也经常用来做分布式锁，Redis提供了多种数据类型来支持不同的业务场景。除此之外，Redis 支持事务持久化、LUA脚本、LRU驱动事件、多种集群方案。

# Redis持久化机制可以说一说吗？
> https://xiaolincoding.com/redis/storage/rdb.html#%E5%BF%AB%E7%85%A7%E6%80%8E%E4%B9%88%E7%94%A8
Redis持久化机制是指将数据从Redis的内存中持久化保存到硬盘中的过程，以防止数据丢失，并确保在Redis重启后能够恢复数据。Redis提供了两种主要的持久化方式：RDB（Redis DataBase）持久化和AOF（Append Only File）持久化。

Redis的持久化机制主要包括RDB（Redis DataBase）和AOF（Append Only File）两种方式，下面我将详细介绍这两种持久化方式的过程。

RDB持久化过程：

1. 手动触发或自动触发：RDB持久化可以通过手动触发（如使用save或bgsave命令）或自动触发（如配置文件中设置的定时任务）来执行。手动触发中，save命令会阻塞Redis主线程直到持久化完成，而bgsave命令通过`fork()` 创建一个子进程进行持久化操作，主线程可以继续处理其他请求。具体是用到了写时复制的技术。fork子进程时，此时子进程和父进程是共享同一片内存数据的，因为创建子进程的时候，会复制父进程的页表，但是页表指向的物理内存还是一个。如果主线程（父进程）要修改共享数据里的某一块数据（比如键值对 A）时，就会发生写时复制，于是这块数据的物理内存就会被复制一份（键值对 A'），然后主线程在这个数据副本（键值对 A'）进行修改操作。与此同时，bgsave 子进程可以继续把原来的数据（键值对 A）写入到 RDB 文件。因此bgsave 快照过程中，如果主线程修改了共享数据，发生了写时复制后，RDB 快照保存的是原本的内存数据。

2. 生成快照文件：当触发持久化后，Redis会开始将内存中的数据生成一个快照文件（RDB文件）。这个过程对于Redis主线程来说几乎是无感知的，因为大部分工作都由子进程完成。子进程会遍历Redis内存中的数据结构，将其转化为二进制格式，并写入到一个临时文件中。

3. 替换旧文件：当持久化过程结束后，子进程会使用这个临时文件替换上一次的持久化文件（即旧的RDB文件）。

4. 启动恢复：当Redis重启时，它会读取RDB文件，并将文件中的数据加载到内存中，实现数据的恢复。

AOF持久化过程：

1. 记录写操作：与RDB不同，AOF持久化是通过记录Redis的写操作命令来实现的。每当Redis执行一个写操作（如SET、DEL等），这个操作命令就会被追加到AOF文件的末尾。
2. 文件同步：为了确保数据的持久化，Redis提供了多种AOF文件同步策略，如每秒同步一次（appendfsync everysec）、每执行一个写操作就同步一次（appendfsync always）等。这些策略可以根据实际需求进行配置。
3. 重写AOF文件：随着时间的推移，AOF文件可能会变得非常大，因为它记录了所有的写操作。为了解决这个问题，Redis提供了AOF重写功能。重写过程会创建一个新的AOF文件，只包含恢复当前数据所需的最小命令集合。重写完成后，Redis会用新的AOF文件替换旧的文件。
4. 启动恢复：当Redis重启时，它会读取AOF文件，并逐条执行文件中的写操作命令，以恢复数据。这个过程可能会比RDB恢复慢一些，尤其是当AOF文件很大时。

总的来说，RDB和AOF持久化方式各有优缺点。RDB适用于对数据一致性要求不高但希望快速恢复的场景，而AOF则提供了更高的数据安全性但可能会牺牲一些性能。在实际应用中，可以根据具体需求选择适合的持久化方式或结合使用。


# AOF重写了解吗？可以简单说说吗？

AOF重写机制是Redis持久化中的一个重要环节，主要用于解决AOF文件随着时间增长而不断膨胀的问题，从而节省磁盘空间并提高数据恢复的效率。下面是AOF重写机制的详细过程：

1. 触发重写：
+ 当AOF文件的大小超过预设的阈值比如64MB，或者手动触发AOF重写命令时，Redis会启动AOF重写机制。
+ 重写机制的触发时机通常配置在Redis的配置文件中，可以通过aof-rewrite-min-size和aof-rewrite-percentage两个参数来控制。

2. 创建子进程：
Redis通过fork命令创建一个子进程来进行AOF重写操作。
子进程会持有主进程在fork时刻的内存数据快照。也就是子进程会持有主进程的数据副本。主进程在通过 fork 系统调用生成 bgrewriteaof 子进程时，操作系统会把主进程的「页表」复制一份给子进程，这个页表记录着虚拟地址和物理地址映射关系，而不会复制物理内存，也就是说，两者的虚拟空间不同，但其对应的物理空间是同一个。当父进程或者子进程在向共享内存发起写操作时，会触发写时复制，复制物理内存。主进程修改了已经存在 key-value，就会发生写时复制，注意这里只会复制主进程修改的物理内存数据，没修改物理内存还是与子进程共享的。为了解决这种数据不一致问题，Redis 设置了一个 AOF 重写缓冲区，在重写 AOF 期间，当 Redis 执行完一个写命令之后，它会同时将这个写命令写入到 「AOF 缓冲区」和 「AOF 重写缓冲区」。
当子进程完成 AOF 重写工作（扫描数据库中所有数据，逐一把内存数据的键值对转换成一条命令，再将命令记录到重写日志）后，会向主进程发送一条信号，信号是进程间通讯的一种方式，且是异步的。
主进程收到该信号后，会调用一个信号处理函数，该函数主要做以下工作：

+ 将 AOF 重写缓冲区中的所有内容追加到新的 AOF 的文件中，使得新旧两个 AOF 文件所保存的数据库状态一致；
+ 新的 AOF 的文件进行改名，覆盖现有的 AOF 文件。

信号函数执行完后，主进程就可以继续像往常一样处理命令了。

AOF重写机制通过压缩AOF文件，减少了磁盘空间的占用，并提高了数据恢复的效率。同时，由于重写操作是在子进程中进行的，主进程可以继续处理客户端的请求，因此不会对Redis的性能产生太大影响。然而，需要注意的是，在AOF重写期间，如果Redis的写入流量非常大，重写缓存可能会堆积较多的数据，这可能会导致在重写完成后，子进程无法完全消费重写缓存中的所有数据，因此需要合理配置Redis的内存和磁盘资源，以确保AOF重写过程的顺利进行。

# Redis的五种底层数据结构整理
> https://interviewguide.cn/notes/03-hunting_job/02-interview/04-02-01-Redis.html
> http://zhangtielei.com/posts/blog-redis-dict.html

# 单线程的Redis为什么这么快？
主要是有三个原因：

1、Redis的全部操作都是纯内存的操作
2、Redis采用单线程，有效避免了频繁的上下文切换；
3，采用了非阻塞I/O多路复用机制。

展开说，
1. Redis 的大部分操作都在内存中完成，并且采用了高效的数据结构，因此 Redis 瓶颈可能是机器的内存或者网络带宽，而并非 CPU，既然 CPU 不是瓶颈，那么自然就采用单线程的解决方案了；
2. Redis 采用单线程模型可以避免了多线程之间的竞争，省去了多线程切换带来的时间和性能上的开销，而且也不会导致死锁问题。
3. Redis 采用了 I/O 多路复用机制处理大量的客户端 Socket 请求，IO 多路复用机制是指一个线程处理多个 IO 流，就是我们经常听到的 select/epoll 机制。简单来说，在 Redis 只运行单线程的情况下，该机制允许内核中，同时存在多个监听 Socket 和已连接 Socket。内核会一直监听这些 Socket 上的连接请求或数据请求。一旦有请求到达，就会交给 Redis 线程处理，这就实现了一个 Redis 线程处理多个 IO 流的效果。

# 缓存雪崩、缓存穿透、缓存预热、缓存更新、缓存击穿、缓存降级全搞定

> https://top.interviewguide.cn/issue/234

布隆过滤器的基本原理是通过一系列哈希函数将集合中的元素映射到位阵列（Bit array）的某些位上，并将这些位设置为1。在查询某个元素是否存在于集合中时，布隆过滤器会检查所有相关的位是否都为1。如果所有相关位都为1，布隆过滤器会报告该元素可能存在于集合中；如果有任何一位为0，则可以确定该元素不在集合中。

# emdstr 编码

Redis中的embstr编码全称是Embedded Simple Dynamic String，简称embstr。embstr编码是Redis中String类型数据的一种编码方式，专门用于保存短字符串。以下是embstr编码的详细解释：

用途：embstr编码用于存储小于44个字节（在Redis 3.2版本之前为39字节）的字符串。
内存分配：embstr编码通过调用一次内存分配函数来分配一块连续的空间，空间中依次包含redisObject和SDS（Simple Dynamic String）结构。这使得embstr编码的数据结构更为紧凑，且只需要一次内存分配操作，从而提高了效率。

与RAW编码的区别：RAW编码在存储大于44个字节的字符串时会被使用，它需要调用两次内存分配函数来分别创建redisObject结构和SDS结构。而embstr编码则只需要一次内存分配。

只读性：embstr编码的字符串是只读的，因为Redis没有为embstr编码提供修改的函数。如果需要对embstr编码的字符串进行修改，Redis会将其转换为RAW编码，然后再执行修改操作。

综上所述，embstr编码是Redis中用于存储短字符串的一种高效、紧凑的编码方式。

# redis实现服务发现

Redis在服务发现中的应用主要依赖于其高性能的键值存储能力和灵活的数据结构。以下是一个基于Redis实现服务发现的清晰步骤：

定义服务注册与发现的规则：
使用Redis的键值对结构来存储服务信息。常见的命名规则如“service:name:port”，其中“name”是服务的名称，“port”是服务的端口号。
创建一个Hash类型的数据结构在Redis中，用于保存已注册的服务。服务的名称作为Hash的键，而服务的主机地址和端口号作为键值对存储。

服务注册：
当一个服务启动时，它需要将自身的信息注册到Redis中。这通常通过执行Redis的HSET命令来完成，如HSET service:Shopservice ip 192.168.0.1 port 8080，将Shopservice服务的主机地址和端口号注册到Redis。

服务发现：
当一个客户端需要调用某个服务时，它会从Redis中查询该服务的信息。这通常通过执行Redis的HGET命令来完成，如HGET service:Shopservice ip和HGET service:Shopservice port，从Redis中获取Shopservice服务的主机地址和端口号。
客户端获得服务信息后，就可以通过网络协议建立连接并进行通信了。

负载均衡：
虽然Redis本身不提供直接的负载均衡功能，但可以通过结合其他策略或工具来实现。例如，客户端可以根据从Redis获取的服务列表，结合负载均衡算法（如轮询、随机、最少连接数等）来选择合适的服务进行调用。

服务更新与注销：
当服务发生变化（如主机地址或端口号改变）时，需要更新Redis中的服务信息。这可以通过执行Redis的HSET命令来完成。
当服务停止时，需要从Redis中注销该服务。这可以通过执行Redis的HDEL命令来删除对应的服务信息。

扩展与集群：
随着服务数量的增加，可能需要扩展Redis的容量或使用Redis集群来提供更高的可用性和容错性。Redis提供了多种集群解决方案，如Redis Sentinel和Redis Cluster，可以根据实际需求进行选择。

监控与告警：
为了确保服务发现的正常运行，需要对Redis进行监控，并设置告警机制。监控指标包括Redis的性能、连接数、内存使用情况等。当指标出现异常时，需要及时告警并采取相应的措施。

通过以上步骤，Redis可以作为一个简单高效的服务注册与发现中心，为分布式系统中的服务提供快速、可靠的信息查询和通信支持。
