	* [在Docker中使用Redis]
	* [Redis-Cli]
	* [命令的返回值类型]
	* [Redis中的多数据库]
	* [基础命令]



Redis是REmote DIctionary Server（远程字典服务器）的缩写，它以字典结构存储数据，并允许其他应用通过TCP协议读写字典中的内容。

Redis数据库中的所有数据都存储在内存中。由于内存的读写速度远快于硬盘，因此Redis在性能上对比其他基于硬盘存储的数据库有非常明显的优势，在一台普通的笔记本电脑上，Redis可以在一秒内读写超过十万个键值。同时Redis也支持持久化数据到硬盘。

### <a name='DockerRedis'></a>在Docker中使用Redis
在Docker中学习和使用Redis非常方便，免去了直接在机器上安装：
```
$ docker run -itd --name redis-test -p 6379:6379 redis:latest
```
redis的默认监听的端口为6379，然后就可以进入redis容器了：
```
$ docker exec -it redis-test /bin/bash
```

### <a name='Redis-Cli'></a>Redis-Cli
redis-cli是Redis的命令行客户端，可以通过cli向Redis发送一系列命令。
在前面docker exec进入redis容器后，就可以使用redis-cli了，可以将命令作为redis-cli的参数，比如用于测试客户端与Redis连接是否正常的PING命令，可以直接这样输入：
```
$ redis-cli PING
```
也可以不附带参数运行redis-cli，这样会进入交互模式，然后直接输入命令：
```
$ redis-cli
redis 127.0.0.1:6379 > PING
```
两种方式下，只要连接正常，都会受到PONG回复。
Redis中的命令是不区分大小写的，但这里为了直观，用大写来表示。

### <a name=''></a>命令的返回值类型
redis执行命令后的返回值有下面几类：
1. 状态回复(status reply)
状态回复是最简单的一种回复，比如向Redis发送SET命令设置某个键的值时，Redis会回复状态OK表示设置成功。之前执行PING命令收到的PONG也属于状态回复。

2. 错误回复(error reply)
命令执行失败会返回错误回复,这类回复以(error)开头。

3. 整数回复(integer reply)
对于类似增加键值、获取键数量等命令会返回整数结果，整数回复与(integer)开头。

4. 字符串回复(bulk reply)
字符串回复是最常见的一种回复类型，当请求一个字符串类型键的键值或一个其他类型键中的某个元素时就会得到一个字符串回复。字符串回复以双引号包裹。但如果键不存在时会返回空，用(nil)表示。

5. 多行字符串回复(multi-bulk reply)
这种回复也比较常见，比如当请求一个非字符串类型键的元素列表时就会收到多行字符串回复，多行字符串回复中的每行字符串都以一个序号开头，如获取所有的键：
```
redis 127.0.0.1:6379 > KEYS *
1) "k1"
2) "k2"
```

### <a name='Redis'></a>Redis中的多数据库 
一个Redis实例提供了多个用来存储数据的字典，客户端可以指定将数据存储在哪个字典中。这与在一个关系数据库实例中可以创建多个数据库类似，所以可以将其中的每个字典都理解成一个独立的数据库。

Redis默认支持16个数据库，用数字命名，分别为0-15。数据库的数量可以通过配置来修改。客户端与Redis建立连接后会自动选择0号数据库，可以自行切换，切换到1号数据库的命令为：
```
SELECT 1
```

Redis中的数据库与我们常规理解的关系型数据库有很大的区别：
- Redis不支持自定义数据库的名字;
- Redis也不支持为每个数据库设置不同的访问密码，所以一个客户端要么可以访问全部数据库，要么无法访问所有数据库；
- 多个数据库之间并不是完全隔离的，比如使用FLUSHALL命令可以清空一个Redis实例中所有数据库中的数据。

鉴于上述区别，将Redis理解为命名空间可能更为合适。不同的redis数据库并不适宜存储不同应用程序的数据，对于多应用的使用，推荐的方式是分别使用不同的Redis实例，由于Redis非常轻量级，一个空Redis实例占用的内存只有1MB左右，所以不用担心多个Redis实例会额外占用很多内存。

### <a name='-1'></a>基础命令
##### KEYS 获取符合规则的键名列表
```
KEYS pattern
```
pattern支持通配符：
- ?，匹配一个字符
- *，匹配任意个字符
- []，匹配方括号间的任一字符，可以使用"-"表示范围，比如a[b-d]可以匹配ab ac ad
- \x，\为转义符

##### EXISTS 判断一个键是否存在
```
EXISTS key
```
如果存在返回(integer) 1，不存在返回(integer) 0。

##### DEL 删除一个或多个键
```
DEL key1
DEL key1 key2 ...
```
返回整数类型表示被删除的个数。如果键不存在，返回0。
DEL命令不支持通配符，但可以组合KEYS命令来实行：
- 可以结合Linux的管道和xargs命令： redis-cli KEYS "user:*" | xargs redis-cli DEL
- 直接用KEYS的输出作为DEL的参数：redis-cli DEL 'redis-cli KEYS "user:*"'


