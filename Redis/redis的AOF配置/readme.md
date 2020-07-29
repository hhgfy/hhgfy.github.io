
最近在看[redis默认配置文件](https://github.com/redis/redis/blob/unstable/redis.conf) AOF部分时候发现了两个没怎么了解过的配置项`aof-load-truncated`和`aof-use-rdb-preamble`，把这部分的配置文件的说明翻译了一下，做个记录。


### 基本配置
```sh
appendonly no # 是否开启aof
appendfilename "appendonly.aof" # 文件名

#磁盘同步策略 默认每秒一次  
# appendfsync always  # 每次
appendfsync everysec # 每秒一次
# appendfsync no # 由操作系统执行，默认Linux配置最多丢失30秒
```

---
### no-appendfsync-on-rewrite

- 作用： 后台执行（RDB的save | aof重写）时appendfsync设为no

```sh
no-appendfsync-on-rewrite no
```

当AOF 的appendfsync配置为 `everysec` 或 `always` ，并且后台运行着RDB的save或者AOF重写时，（由于save和rewrite会）消耗大量的磁盘性能。在某些Linux配置下，aof同步到磁盘执行 `fsync()` 将被阻塞很长时间。

注意：这个问题目前还没有解决办法。因为即使在不同的线程中执行fsync，也会阻塞同步 `write()` 调用。

为了减轻这个问题，可以使用 `no-appendfsync-on-rewrite` 选项防止执行`BGSAVE`或`BGREWRITEAOF`时，在主进程中调用fsync()。

这意味着当有另一个子进程执行`BGSAVE`或`BGREWRITEAOF` 时，磁盘同步策略相当于 `appendfsync no`。在最坏的情况下(使用默认的Linux设置)可能会丢失最多30秒的日志。

如果你有延迟问题可以将该项设置为“yes”。否则，从数据完整性的角度看，使用“no”更安全。


---
###  auto-aof-rewrite-percentage
- 作用： 自动触发AOF重写
```sh
auto-aof-rewrite-percentage 100 # 触发重写百分比 （指定百分比为0，将禁用aof自动重写功能）
auto-aof-rewrite-min-size 64mb # 触发自动重写的最低文件体积（小于64mb不自动重写）
```

Redis能够在AOF文件大小增长了指定百分比时，自动隐式调用 `BGREWRITEAOF` 命令进行重写。

这里是它如何工作的说明：

1. Redis记录上一次执行AOF重写后的文件大小作为基准。（如果启动后没有发生过重写，则使用启动时的AOF文件大小）。
2. 将该基准值与当前文件大小进行比较，如果当前体积超出基准值的指定百分比，将触发重写。

另外你还需要指定要重写的AOF文件的最小体积`（auto-aof-rewrite-min-size）`，这可以避免在文件体积较小时多次重写。

如果指定百分比为0，将禁用AOF自动重写功能


---
### aof-load-truncated
- 作用：指定当发生AOF文件末尾截断时，加载文件还是报错退出

```sh
aof-load-truncated yes 
```

Redis启动并加载AOF时，可能发现AOF文件的末尾被截断了。

如果Redis所在的机器运行崩溃，就可能导致该现象。特别是在不使用 `data=ordered` 选项挂载ext4文件系统时。（但是Redis本身崩溃而操作系统正常运行则不会出现该情况）

当发生了末尾截断，Redis可以选择直接报错退出，或者继续执行并恢复尽量多的数据（默认选项）。配置项 `aof-load-truncated` 用于控制此行为。

- yes ：末尾被截断的 AOF 文件将会被加载，并打印日志通知用户。

- no ：服务器将报错并拒绝启动。
    - 这时用户需要使用`redis-check-aof` 工具修复AOF文件，再重新启动。


注意：如果AOF文件在中间（而不是末尾）发生了截断，仍然会报错退出。 `aof-load-truncated`只适用于当Redis将尝试从AOF文件读取更多的数据，但发现没有足够的字节时。

---
###  aof-use-rdb-preamble
- 作用： 开启混合持久化，更快的AOF重写和启动时数据恢复

```sh
aof-use-rdb-preamble yes
```

#### AOF重写时
当开启该选项时，触发AOF重写将不再是根据当前内容生成写命令。而是先生成RDB文件写到开头，再将RDB生成期间的发生的增量写命令附加到文件末尾。

重写完成后，继续像普通AOF一样追加内容。

文件格式： `[RBD文件内容][追加的AOF日志]`

#### 重启加载AOF时

启动时加载AOF如果碰到RDB的开头前缀 `REDIS`，先按RDB恢复数据。再将附加在末尾的写命令重放，恢复完整数据
