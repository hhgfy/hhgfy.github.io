Redis提供了Lua脚本功能来让用户实现自己的原子命令，但也存在着风险，编写不当的脚本可能阻塞线程导致整个Redis服务不可用。

本文将介绍Redis中Lua脚本的基本用法，以及脚本超时导致的问题和处理方式。


## EVAL命令简介

### eval格式

Redis 提供了命令`EVAL`来执行Lua脚本，格式如下

```
EVAL script numkeys key [key …] arg [arg …]
```

其中 `script` 是将要执行的脚本内容，至于后面的脚本参数部分与本文无关，在此不做赘述。


### 特性

由于Redis对数据集单线程读写的特性，Lua脚本执行时会阻塞所有对数据集的读写操作，这给它带来了下面两个特性：

- 原子性：可以通过Lua脚本实现对数据集的原子读写操作，这和Redis的事务功能`MULTI / EXEC`类似

- 长时间阻塞风险：如果Lua脚本执行时间过长，导致整个Redis不可用


### 执行流程

已 `eval "return 'hello world'" 0`为例，脚本执行步骤如下

1. 定义脚本函数
   -  执行过的脚本可以根据hash值找到函数重新使用

Redis会根据传入的脚本内容生成函数，函数名由 `f_` + 脚本内容的sha1摘要组成。 

```Lua
function f_5332031c6b470dc5a0dd9b4bf2030dea6d65de91()
	return 'hello world'
end
```

2. 函数保存到 `Lua_scripts`字典，便于 `evalsha`使用

3. 执行脚本函数
   1. 将KEYS和ARGV两个参数数组传入Lua执行环境
   2. 装载超时处理钩子
   3. 执行脚本
   4. 移除超时钩子
   5. 结果保存到客户端输出缓冲区，等待服务器将结果返回客户端
   6. Lua环境垃圾回收

---

## 关于脚本超时

介绍完EVAL命令，下面来关注Lua脚本长时间阻塞的风险。

Redis的配置文件中提供了如下配置项来规定最大执行时长

- `lua-time-limit 5000` Lua脚本最大执行时间，默认5秒



但这里有个坑，当一个脚本达到最大执行时长的时候，Redis并不会强制停止脚本的运行，仅仅在日志里打印个警告，告知有脚本超时。

```sh
Lua slow script detected: still in execution after 5000 milliseconds. You can try killing the script using the SCRIPT KILL command. Script SHA1 is: 2531e4edc1a1e2a9bac3c52e99466f9ccabf12c0
```

为什么不能直接停掉呢？

因为 Redis 必须保证脚本执行的原子性，中途停止可能导致内存的数据集上只修改了部分数据。

>（只读的脚本应该是可以自动停的，没自动停的原因我猜测是：脚本超时严重可以肯定出现了编码错误，作者可能希望在测试中尽早发现这种问题，而不是靠自动停止导致bug被忽略？）


如果时长达到 `Lua-time-limit` 规定的最大执行时间，Redis只会做这几件事情：

- 日志记录有脚本运行超时
- 开始允许接受其他客户端请求，但仅限于 `SCRIPT KILL` 和 `SHUTDOWN NOSAVE` 两个命令
  - 其他请求仍返回busy错误


### SCRIPT KILL 命令

如果Lua只是读取数据而没做修改的话，执行 `SCRIPT KILL` 就可以直接终止脚本执行，不用担心数据被修改。

但是，如果脚本已经改写了数据内容，`SCRIPT KILL`将报出以下错误，因为它破坏数据集的内容。

```
(error) UNKILLABLE Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.
```


### SHUTDOWN NOSAVE 命令

如上所述，如果脚本已经执行了写命令，`SCRIPT KILL`将无法执行。那我们就只剩以下两种选择了：

1. 继续等待脚本执行完成
2. 使用 `SHUTDOWN NOSAVE` 来直接停掉 Redis，并避免脏数据持久化到磁盘


最后，不知道你有没有疑问，从开始执行脚本到 SHUTDOWN 之间的写命令会把日志写到AOF里吗？Lua脚本中的命令什么时候会写AOF里？

讲道理，既然 Redis 为了不破坏脚本的原子性而不让`SCRIPT KILL`执行，那么脚本中写命令的 **“提交”** 也应当是原子执行的，而不是执行一句就向AOF里写一句。

> “提交”：借用数据库中 commit 的概念，这里指写入AOF文件中

下面就来验证这个猜测：

先执行 `tail -f appendonly.aof` 实时查看AOF文件变化


再开一个redis-cli 命令行执行一个内容如下的Lua脚本

```Lua
redis.call('set','a','aaaa') --先执行写命令
local count = 1 
while( 999999999 > count ) -- 阻塞几秒
do  
   count = count+1   
end
```

```
127.0.0.1:6379> eval "redis.call('set','a','aaaa') local count = 1 while( 999999999 > count ) do  count = count+1   end" 0
(nil)
(8.65s)
```

现象是，脚本刚开始执行，AOF文件毫无反应，一直等到8秒后脚本完成，命令才追加写入到AOF中。

这就验证了Redis脚本里的写命令是等到执行完成后再一次性写入AOF的。




## 参考

- [Redis设计与实现](https://book.douban.com/subject/25900156/)


