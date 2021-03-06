- 持久化
    - RDB方式
        - Redis实现快照的过程
    - AOF方式
        - 操作系统缓存
        - RDB与AOF
- 复制
    - 主从数据库
    - 主从复制的意义
- 安全

## 持久化
Redis通过将数据存储在内存中而获得了极快的速度，但为了保证Redis在重启后数据不丢失，需要将数据从内存持久化到硬盘中。

持久化的方式有两种，二者可以只用一种，也可以组合使用：
- RDB方式
- AOF方式

### RDB方式
RDB是Redis默认采用的持久化方式，当符合一定条件时Redis会自动将内存中的所有数据进行快照（snapshotting）并存储在硬盘上。进行快照的条件可以由用户在配置文件中自定义，配置由两个参数构成：时间和改动的键的个数。当在指定的时间内被更改的键的个数大于指定的数值时就会进行快照。在配置文件redis.conf中已经预置了3个条件：
```
save 900 1
save 300 10
save 60 10000
```
这些条件直接是“或”的关系。如果需要禁用自动快照，可以将所有的save配置删除。

除了自动快照，还可以手动发送SAVE或BGSAVE命令让Redis执行快照，SAVE命令是由主进程进行快照操作，会阻塞住其他请求，而BGSAVE命令则会通过fork子进程进行快照操作。

Redis默认会将快照文件存储在当前目录的dump.rdb文件中，可以通过配置dir和dbfilename两个参数分别指定快照文件的存储路径和文件名。


#### Redis实现快照的过程
RDB快照的过程为：
1. Redis使用fork函数复制一份当前进程（父进程）的副本（子进程）；
2. 父进程继续接收并处理客户端发来的命令，子进程则开始将内存中的数据写入硬盘中的临时文件；
3. 当子进程写入完所有数据后，会用该临时文件替换旧的RDB文件，至此一次快照操作完成。
4. Redis重新启动时会读取RDB快照文件，将数据从硬盘载入到内存。（根据数据量大小与结构和服务器性能不同，这个时间也不同。通常将一个记录一千万个字符串类型键、大小为1GB的快照文件载入到内存中需要花费20～30秒钟）。

Redis在进行快照的过程中不会修改RDB文件，只有快照结束后才会将旧的文件替换成新的，也就是说任何时候RDB文件都是完整的，所以可以通过定时备份RDB文件来实现Redis数据库备份。RDB文件是经过压缩的二进制格式，占用的空间会小于内存中的数据大小。但压缩过程也会增加CPU占用，如有需要可以通过修改配置rdbcompression参数以禁用压缩。
```
config set rdbcompression no
```

### AOF方式
AOF全称为append only file，在这种持久化方式下，每执行一条会更改Redis中的数据的命令，Redis都会将该命令写入硬盘中的AOF文件。
AOF文件的保存位置和RDB文件的位置相同，都可以通过dir参数设置，默认的文件名是appendonly.aof，可以通过appendfilename参数修改。
AOF文件是纯文本文件，其内容是Redis客户端向Redis发送的原始通信协议的内容。设想执行这样几条命令：
```
SET key1 1
SET key1 2
SET key1 3
```
AOF文件中会记录这三次操作，但实际上前两条实际上是多余的，只需要记录最终一次的命令即可。随着执行的命令越来越多，AOF文件也会越来越大，而Redis可以通过去除这类多余命令记录，自动对AOF文件进行优化。

每当达到一定条件时Redis就会自动重写AOF文件，这个条件可以在配置文件中设置：
```
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```
auto-aof-rewrite-percentage设置当目前的AOF文件大小超过上一次重写时的AOF文件大小的百分之多少时会再次进行重写，如果之前没有重写过，则以启动时的AOF文件大小为依据。
auto-aof-rewrite-min-size则限制了允许重写的最小AOF文件大小，通常在AOF文件很小的情况下即使其中有很多冗余的命令也不需要重写。

除了让Redis自动执行重写外，还可以主动使用BGREWRITEAOF命令手动执行AOF重写。

在重启时Redis会逐个执行AOF文件中的命令来将硬盘中的数据载入到内存中，载入的速度相较RDB会慢一些。

#### 操作系统缓存
虽然每次执行更改数据库内容的操作时，AOF都会将命令记录在AOF文件中，但实际上由于操作系统的缓存机制的存在，数据并没有真正地写入硬盘，而是进入了系统的硬盘缓存。在默认情况下系统每30秒会执行一次同步操作，以便将硬盘缓存中的内容真正地写入硬盘，在这30秒的过程中如果系统异常退出则会导致硬盘缓存中的数据丢失。一般来讲启用AOF持久化的应用都无法容忍这样的损失，这就需要Redis在写入AOF文件后主动要求系统将缓存内容同步到硬盘中。
可以通过appendfsync参数设置同步的时机：
```
config set appendfsync always/eversec/no
```
- always 每次执行写入都会执行同步，这种模式最安全但也最慢
- everysec 每秒同步一次
- no 不主动进行同步操作，操作系统没30秒同步一次

#### RDB与AOF
如果选择RDB的持久化方式，一旦Redis异常退出，就会丢失最后一次快照以后更改的所有数据。这就需要开发者根据具体的应用场合，通过组合设置自动快照条件的方式来将可能发生的数据损失控制在能够接受的范围。如果数据很重要以至于无法承受任何损失，则可以考虑使用AOF方式进行持久化。
可以同时开启AOF和RDB，这样既保证了数据安全，又使得进行备份等操作十分容易。此时重新启动Redis后Redis会使用AOF文件来恢复数据，因为AOF方式的持久化可能丢失的数据更少。


## 复制
Redis的持久化功能可以保证在服务器重启的情况下不会损失（或少量损失）数据。但是由于数据只存储在一台服务器，如果这台服务器的硬盘出现故障，也会导致数据丢失。为了避免单点故障，将数据库复制多个副本以部署在不同的服务器上，组成集群，这样即使有一台服务器出现故障时，其他服务器依然可以继续提供服务，此外也提升了整体的性能。
Redis提供了复制（replication）功能用以保障构成集群的多台服务器之间数据的同步。

### 主从数据库
构成集群的数据库分为两类，主数据库（master）和从数据库（slave）。主数据库可以进行读写操作，当发生写操作时自动将数据同步给从数据库。而从数据库一般是只读的，并接受主数据库同步过来的数据。主从数据库是一对多的关系。

要把一个数据库作为从数据库，需要在启动参数或者配置文件中加入：
```
slaveof 主数据库的IP 端口
```
接下来使用redis的docker镜像试验主从复制，首先启动两个实例：
```
docker run --name redis-6379 -p 6379:6379 -d redis
docker run --name redis-6380 -p 6380:6379 -d redis
```
redis-6379将被作为主数据库，redis-6380为从库，首先获取redis-6379容器的内网IP地址：
```
docker inspect redis-6379
```
在"NetworkSettings"结点下可以看到IP和端口为172.17.0.2 6379

进入redis-6380，设置其为从数据库：
```
> docker exec -it redis-6380 /bin/bash
root@cd19cbbab6d9:/data# redis-cli
127.0.0.1:6379> SLAVEOF 172.17.0.2 6379
```
这样主从数据库就设置好了，可以测试下在主数据库插入一个键，然后在从数据库读取。

从数据库默认是只读的，尝试写入将提示：
```
SET KEY1 1
(error) READONLY You can't write against a read only replica.
```
可以通过设置从数据库的配置文件中的slave-read-only为no以使从数据库可写，但是对从数据库的任何更改都不会同步给其他数据库，并且一旦主数据库中更新了对应的数据就会覆盖从数据库中的改动。

在redis实例运行也可以使用SLAVEOF命令，如果该数据库已经是其他主数据库的从数据库，则SLAVEOF命令会停止和原来数据库的同步转而和新数据库同步。还可以使用SLAVEOF NO ONE来使当前数据库停止接收其他数据库的同步转成主数据库。

### 主从复制的意义
- 数据冗余:主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。

- 故障恢复:当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复;实际上是一种服务的冗余。

- 读写分离:可以用于实现读写分离，主库写、从库读，读写分离不仅可以提高服务器的负载能力，同时可根据需求的变化，改变从库的数量;

- 负载均衡:在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务分担服务器负载;尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量。


- 高可用基石:除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础。


## 安全
在安全方面，Redis提供了下面几种策略。

### 可信的环境
Redis安全设计都是建立在“Redis运行在可信环境”这个前提下的，在生产环境运行时不能允许外界直接连接到Redis服务器上，而应该通过应用程序进行中转，运行在可信的环境中是保证Redis安全的最重要方法。

Redis的默认会接受来自任何地址发送来的请求，要更改这一设置，可以在配置文件中修改bind参数，如只允许本机应用连接Redis，可以将bind参数改成：
```
bind 127.0.0.1
```

### 密码
要设置密码，可以修改配置文件中的requirepass参数，例如：
```
requirepass 123456
```
docker镜像也可以在启动容器的时候设置密码：
```
docker run --name redis-test -p 6379:6379 redis --requirepass 123456
```
添加密码后，客户端在连接时需要输入密码:
```
redis-cli -h 127.0.0.1 -p 6379 -a 123456
```
设置密码后，如果要搭建主从数据库，需要在从数据库配置文件中的masterauth添加主数据库的密码。

需要注意的是，由于Redis的性能极高，并且输入错误密码后Redis并不会进行主动延迟，所以攻击者可以通过穷举法破解Redis的密码（1秒内能够尝试十几万个密码），因此在设置时一定要选择复杂的密码。

### 命令重命名
Redis支持在配置文件中将命令重命名，比如将FLUSHALL命令重命名成一个比较复杂的名字，以保证只有自己的应用可以使用该命令：
```
rename-command FLUSHALL oiuyhjkghjgyutdfhbuhbnjinjbgyvtcrd
```
如果希望直接禁用某个命令，可以将命令重命名成空字符串。
