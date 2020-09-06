
> 本文内容基于 Redis 6.0.6 版本

最近重新读了《Redis设计与实现》，注意到了一些原来没在意的小细节。比如 `9.7 AOF、RDB和复制功能对过期键的处理` 这节中说到的从节点可能读到过期数据的问题。


## 单机过期实现方式

在了解从节点读到过期数据这个问题之前，不得不先了解在单机情况下Redis如何实现数据过期功能的。

首先，要明白一点，虽然看起来过期时间一到，过期的键就立即不可见了。但Redis实际上并没有做到实时的过期删除，而是采取了后台定期删除与惰性删除相结合的方式。

### 定期删除

Redis内部存在一个定时任务（默认每秒运行10次），每次执行时抽查一部分key判断是否过期，碰到过期的键则进行删除。

如果抽样中过期键的比例较高，则继续执行抽查删除，这样可以实现过期键越多，删的速度越快的效果。

当然，在过期键比例非常高时，也不能无止境的陷在抽样删除的循环里，Redis还设置了一次执行的最大时长限制，避免一直阻塞主线程，影响服务

以下是定期抽样删除功能的终止条件部分代码：（仅更改了注释，下文同）
```c++
// src/expire.c/activeExpireCycle()
do {
//执行抽样和删除（略）

// 过期键太多时不能永远阻塞下去，需要在指定时间内结束
if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
    elapsed = ustime()-start; //当前时间 - 开始时间 = 已用时长
    if (elapsed > timelimit) { //达到最大时长限制，break退出 do...while 循环
        timelimit_exit = 1;
        server.stat_expired_time_cap_reached_count++;
        break;
    }
}
//如果抽样中过期键的比例低于 config_cycle_acceptable_stale ，结束循环
//若过期键比例较高，则继续删
} while (sampled == 0 ||
        (expired*100/sampled) > config_cycle_acceptable_stale);
```


### 惰性删除

仅凭定期删除会存在删除不及时的问题，但客户端访问可不会等过期键删除完。

所以当访问任意key时，都会先判断它是否处于已经过期但还没来得及删除的状态，如果已经过期则立刻删除，并向上层返回该key不存在。


---
## 主从之间的过期

说完了单机情况下键过期的实现方式，再回到最开始的问题：从节点可能读到过期数据。


书里写的原因是： 

从节点不会主动删除访问到的过期数据，而是要等主节点数据过期后生成`DEL`命令发过来。由于定期删除机制不够及时，在到达过期时间点与实际收到`DEL`命令这段时间内，读取从节点将会获取到本应过期的数据，而不执行惰性删除。


这怎么看都是一个bug吧？ 网上搜了一圈，都说在Redis 3.2版本修复了该问题，但没有找到具体用什么方式修的。

在官方文档的 [Replication](https://redis.io/topics/replication#how-redis-replication-deals-with-expires-on-keys) 一节中，有部分关于主从之间过期键处理的说明。

大意如下：

---
实现复制功能不能依靠主从节点的墙上时钟，Redis使用了以下三个方式来处理：

1. 副本不会主动删除过期键，而是等主节点过期时生成 `DEL` 命令同步至从节点进行删除。


2. 但主节点过期删除不及时，可能会使从节点上存在逻辑上已经过期的键。为了处理该问题，副本采用它自己的 `逻辑时钟` 来判断读取时键是否应当过期，过期则返回不存在（即使数据仍然在内存中，等着主节点的 DEL 命令）

```
In order to deal with that the replica uses its logical clock in order to report that a key does not exist only for read operations that don't violate the consistency of the data set (as new commands from the master will arrive). 
```

3. 在Lua脚本运行时，服务器中的时间是 **冻结** 的，防止键在脚本运行的过程中过期。这是为了保持副本上执行的脚本能具有相同的效果。（注：不同机器，性能不一样，脚本执行时长也不同）

---

文档中只给出了一个模糊的 “根据 **逻辑时钟** 判断” 的描述，没有具体说明实现方式。我只能先来猜一下实现方式，然后去代码里试试能不能找到。


### 我猜的实现方式

先补充一下背景知识：分布式系统中，网络和本地系统时钟都是 **不可靠** 的。

既然文档中描述了**逻辑时钟**，那么我猜测的方式如下：

开始执行 `slaveof` 的时候，主从节点校对时钟，每个从节点维护一个与主节点的`时间差`，实现逻辑时钟。

主节点上执行的与过期相关的命令 `set k v ex time`、`expire`、`expireat`、`pexpire`、`pexpireat` 等在传播到从节点时通通根据主节点的时钟转成 `pexpireat`，使用基于主节点的绝对时间。

从节点接到命令时，对这些过期命令加减`时间差`，实现主从一致的过期。（也就是说一切时间以主节点为准）


### 实际的实现方式

然而事实证明我完全想多了，Redis代码中关于过期判断的部分完全没有计算`时间差`的影子。

这里是其中一处过期判断的代码：
```c++
// src/db.c
int keyIsExpired(redisDb *db, robj *key) {
    mstime_t when = getExpire(db,key); //从过期字典获取过期时间，没有返回-1
    mstime_t now;
    
    if (when < 0) return 0;// when=-1 永不过期
    if (server.loading) return 0;

    if (server.lua_caller) { //lua脚本中的命令总是使用脚本开始时间
        now = server.lua_time_start;
    } else if (server.fixed_time_expire > 0) { //如 RPOPLPUSH 这种操作多个key的命令也使用命令开始时间，fixed_time_expire 在调用call()时刷新
        now = server.mstime;
    } else {
        now = mstime();
    }
    return now > when;
}
```

这里没找到解决方式，没办法只能直接去找3.2版本那次修复的提交记录，看看他是怎么改的。

于是我从issues里找到了 [Improve expire consistency on slaves](https://github.com/redis/redis/issues/1768) 这个关于提高主从过期时间一致性的问题，并在其中发现了对应的 [提交记录](https://github.com/redis/redis/commit/06e76bc3e22dd72a30a8a614d367246b03ff1312)。

结果令人大跌眼镜，总共只有寥寥几行，在过期判断后增加了从节点和只读命令的判断条件，然后返回null。

这下更令人疑惑了，这么说来，主从节点的过期没做区分，都用了本地系统的墙上时钟吗？

只能再换个方向去找了，Redis 还有什么地方可能让主从之间的命令产生区别？没错，在主节点到从节点的命令传播那里说不定能有新发现！

```c++
// src/server.c
// 命令传播
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,
               int flags)
{
    if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
        feedAppendOnlyFile(cmd,dbid,argv,argc); // 传给AOF
    if (flags & PROPAGATE_REPL)
        replicationFeedSlaves(server.slaves,dbid,argv,argc); //传给从节点
}
```

先看了下传播到AOF文件的函数 `feedAppendOnlyFile()`，在内部它通过`catAppendOnlyExpireAtCommand`将`expire`、`expireat`等过期相关命令全转成了`PEXPIREAT`，看起来有门哈。

再看向从节点传播命令的 `replicationFeedSlaves()`，很遗憾找了一圈下来没发现有对过期命令的特殊处理。

没做特殊处理也就意味着 `expire`、`expireat` 它们在主从节点上都是按原样执行的。但是，由于判断过期使用了各自的本地系统时钟，会不会导致主从节点之间产生不一致的行为？



### 使用本地时钟导致的问题


下面分析当主从节点所在机器的系统时间不一致时，将导致什么样的问题。


#### expire

对于 `expire`命令，键存活一段指定的时间后过期。即使主从时间不一致，但时长是一样的。

表现出的结果是，客户端同一时刻访问主从节点能得到相同的TTL，要过期也是一起过期，没有问题。

#### expireat

而 `expireat` 命令则不同了，由于它指定了一个确定的过期时间点，对单个节点自身来说，确实是实现了到达指定时间点过期的效果，符合命令语义。

**但是**，站在客户端视角来看，景象就不这么美好了。


在客户端看来，由于主从系统时间存在的偏差，令主从之间一样的时间戳并非实际上的 “同时”。表现出的结果将是，客户端在同一时刻访问主从节点的同一个键，将得到不同的TTL，乃至于一个过期另一个没过期。

虽然仅从语义上看这不能算是错，错的是系统时间。但由此产生的主从表现不一致现象也是需要考虑考虑的。

### 实际验证

上面只是根据代码推测的结果，接下来实际验证一下是否如此。

- 准备：需要准备两台机器，时间调成不一样的，并搭建Redis主从复制
    - 这里我将本地时间用`date -s` 调快了1小时，然后执行`slaveof`复制了另一台机器的上的Redis

- 验证 expire
```sh
# 主节点
set a 1
expire a 1800  #半小时过期

# 主从分别执行TTL， 时间一样（忽略操作时间）
```

---
- 验证 expireat

```sh
> expireat a 两小时后
# 主节点
> ttl a
3595

# 从节点
> ttl a
1789

# 两边都没过期，但TTL不一致

#---------------------

> expireat a 半小时后
# 主节点
> ttl a
1797

# 从节点 不存在该键
> ttl a
-2

# 从节点系统时间调快了一小时，键已经被判断为过期不存在了
```


---
## 结论

当使用 `expireat` 命令时，如果主从时间不一致，分别读取主从可能得到不一致的响应。

可以得出以下两点结论：

1. 运行Redis的机器一定要做好对时 （只能压缩不一致现象持续的时长，而非完全避免）

2. `expireat` 存在主从表现不一致的可能，应尽量优先使用 `expire` 命令来替代


---
## 参考

- [《Redis设计与实现》9.7节  AOF、RDB和复制功能对过期键的处理](https://book.douban.com/subject/25900156/)
- [《数据密集型应用系统设计》P271. 不可靠的时钟 ](https://book.douban.com/subject/30329536/)

