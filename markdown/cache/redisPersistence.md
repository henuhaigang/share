## 持久化方式
Redis中数据存储模式有2种
- cache-only

    即只做为缓存服务，不持久数据，数据在服务终止后将消失，此模式下也将不存在数据恢复的手段，是一种安全性低/效率高/容易扩展的方式
- persistence

    即为内存中的数据持久备份到磁盘文件，在服务重启后可以恢复，此模式下数据相对安全。
### 一 RDB(redis database)
RDB是在某个时间点将数据写入一个临时文件，持久化结束后，用这个临时文件替换上次持久化的文件，通过配置redis在n秒内如果超过m个key被修改这执行一次RDB操作。
这个操作就类似于在这个时间点来二进制的方式保存一次Redis的一次快照数据。所有这个持久化方法也通常叫做snapshots
 
- 优点

    使用单独子进程来进行持久化，主进程不会进行任何IO操作，保证了redis的高性能 
- 缺点

    RDB是间隔一段时间进行持久化，如果持久化之间redis发生故障，会发生数据丢失。所以这种方式更适合数据要求不严谨的时候

redis默认开启RDB    
#### redis.config配置

```text
# The filename where to dump the DB（持久化数据存储在本地的文件）
dbfilename dump.rdb

# The working directory.（持久化数据存储在本地的路径）
#
# The DB will be written inside this directory, with the filename specified
# above using the 'dbfilename' configuration directive.
#
# The Append Only File will also be created inside this directory.
#
# Note that you must specify a directory here, not a file name.
dir ./

## snapshot触发的时机，save <seconds> <changes>  
## 900秒后，至少有一个变更操作，才会触发snapshot 
save 900 1
## 300秒10次变更触发
save 300 10
## 60秒 10000次变更触发
save 60 10000
## 可以通过save ""来关闭snapshot功能  

## 当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”，“错误”可能因为磁盘已满/磁盘故障/OS级别异常等  
stop-writes-on-bgsave-error yes 
 
## 是否启用rdb文件压缩，默认为“yes”，压缩往往意味着“额外的cpu消耗”，同时也意味这较小的文件尺寸以及较短的网络传输时间  
rdbcompression yes  

```
#### RDB数据恢复 
- 实际只要重启redis服务即可完成（启动redis的server时会从dump.rdb中先同步数据）

#### 客户端使用命令进行持久化save存储
在前台进行存储，由于redis是用一个主线程来处理所有 client的请求，这种方式会阻塞所有client请求。所以不推荐使用。
```text
./redis-cli -h ip -p port save
```
在后台进行存储。我的client就在server这台服务器上，所以不需要连其他机器，直接./redis-cli bgsave
```text
./redis-cli -h ip -p port bgsave
```
每次快照持久化都是将内存数据完整写入到磁盘一次，并不是增量的只同步脏数据。如果数据量大的话，而且写操作比较多，必然会引起大量的磁盘io操作，可能会严重影响性能。
### 二 AOF(append-only file)
AOF 则以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到 AOF 文件，以此达到记录数据库状态的目的，
当server需要数据恢复时，可以直接replay此日志文件，即可还原所有的操作过程。AOF相对可靠，它和mysql中bin.log、apache.log、zookeeper中txn-log简直异曲同工

- 优点

    可以保持更高的数据完整性，如果设置追加file的时间是1s，如果redis发生故障，最多会丢失1s的数据；且如果日志写入不完整支持redis-check-aof来进行日志修复；AOF文件没被rewrite之前（文件过大时会对命令进行合并重写），可以删除其中的某些命令（比如误操作的flushall）。 
- 缺点

    AOF文件比RDB文件大，且恢复速度慢
#### AOF rewrite
- 使用rewrite原因

    只会记录“变更操作”(例如：set/del等)，如果server中持续的大量变更操作，将会导致AOF文件非常的庞大，意味着server失效后，数据恢复的过程将会很长；事实上，一条数据经过多次变更，将会产生多条AOF记录，其实只要保存当前的状态，历史的操作记录是可以抛弃的；因为AOF持久化模式还伴生了“AOF rewrite” 
- rewrite原理

    AOF rewrite操作就是“压缩”AOF文件的过程，当然redis并没有采用“基于原aof文件”来重写的方式，而是采取了类似snapshot的方式：基于copy-on-write，全量遍历内存中数据，然后逐个序列到aof文件中。因此AOF rewrite能够正确反应当前内存数据的状态，这正是我们所需要的；*rewrite过程中，对于新的变更操作将仍然被写入到原AOF文件中，同时这些新的变更操作也会被redis收集起来(buffer，copy-on-write方式下，最极端的可能是所有的key都在此期间被修改，将会耗费2倍内存)，当内存数据被全部写入到新的aof文件之后，收集的新的变更操作也将会一并追加到新的aof文件中，此后将会重命名新的aof文件为appendonly.aof,此后所有的操作都将被写入新的aof文件  
#### redis.config配置
```text
AOF默认关闭，此选项为aof功能的开关，默认为“no”，可以通过“yes”来开启aof功能，只有在“yes”下，aof重写/文件同步等特性才会生效  
appendonly yes  
##指定aof文件名称  
appendfilename appendonly.aof  
##指定aof操作中文件同步策略，有三个合法值：always everysec no,默认为everysec  
appendfsync everysec  
##在aof-rewrite期间，appendfsync是否暂缓文件同步，"no"表示“不暂缓”，“yes”表示“暂缓”，默认为“no”  
no-appendfsync-on-rewrite no  
##aof文件rewrite触发的最小文件尺寸(mb,gb),只有大于此aof文件大于此尺寸是才会触发rewrite，默认“64mb”，建议“512mb”  
auto-aof-rewrite-min-size 64mb  
##相对于“上一次”rewrite，本次rewrite触发时aof文件应该增长的百分比。  
##每一次rewrite之后，redis都会记录下此时“新aof”文件的大小(例如A)，那么当aof文件增长到A*(1 + p)之后  
##触发下一次rewrite，每一次aof记录的添加，都会检测当前aof文件的尺寸。  
auto-aof-rewrite-percentage 100  
```
### 三 总结
OF和RDB各有优缺点，这是有它们各自的特点所决定：
- AOF更加安全，可以将数据更加及时的同步到文件中，但是AOF需要较多的磁盘IO开支，AOF文件尺寸较大，文件内容恢复数度相对较慢。 
- RDB安全性较差，它是“正常时期”数据备份以及master-slave数据同步的最佳手段，文件尺寸较小，恢复数度较快

在架构良好的环境中，master通常使用AOF，slave使用snapshot，主要原因是master需要首先确保数据完整性，它作为数据备份的第一选择；slave提供只读服务(目前slave只能提供读取服务)，它的主要目的就是快速响应客户端read请求；但是如果你的redis运行在网络稳定性差/物理环境糟糕情况下，建议你master和slave均采取AOF，这个在master和slave角色切换时，可以减少“人工数据备份”/“人工引导数据恢复”的时间成本