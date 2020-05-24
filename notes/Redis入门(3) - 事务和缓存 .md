- 事务的使用方式
- 事务的错误处理
- WATCH命令
- 生存时间
- 缓存策略

Redis中的事务（transaction）是一组命令的集合。事务同命令一样都是Redis的最小执行单位，一个事务中的命令要么都执行，要么都不执行。

事务的原理是先将属于一个事务的命令发送给Redis，然后再让Redis依次执行这些命令。

### 事务的使用方式
```
> MULTI
OK
> SADD key1 1
QUEUED
> SADD key2 2
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```
首先用MULTI开启事务，redis会返回"OK"。
接下来输入的命令都会被加入到等待执行的事务队列中，而不是像通常一样立即执行，redis会返回"QUEUED"表示成功加入到队列中了。

最终需要执行时，使用EXEC命令告诉Redis将等待执行的事务队列中的所有命令按照发送的顺序依次执行。EXEC命令的返回值就是这些命令的返回值组成的列表，返回值顺序和命令的顺序相同。

通过这种方式，Redis可以保证一个事务中的所有命令要么都执行，要么都不执行。如果在发送EXEC命令前客户端断线了，则Redis会清空事务队列，事务中的所有命令都不会执行。而一旦客户端发送了EXEC命令，所有的命令就都会被执行，即使此后客户端断线也没关系，因为Redis中已经记录了所有要执行的命令。

除此之外，Redis的事务还能保证一个事务内的命令依次执行而不被其他命令插入。比如在客户端A执行多条命令的时候，客户端B也恰好发送了一条命令，如果不使用事务，则客户端B的命令可能会插入到客户端A的命令中执行。而使用事务也可以避免这种情况的发生。

### 事务的错误处理
如果事务中的某个命令执行错误，Redis怎么处理呢？分两种情况：
1. 语法错误。命令的使用方式不对，或输入了不存在的命令，这类情况在加入事务队列前就会被识别出来，发送EXEC后Redis会直接返回错误提示，事务中的命令都不会被执行。
```
 > MULTI
 OK
 > SET key value
 QUEUED
 > SET key
 (error)ERR wrong number of arguments for 'set' command
 > ERRORCOMMAND key
 (error)ERR unknown command 'ERRORCOMMAND
 > EXEC
 (error) EXECABORT Transaction discarded because
 of previous errors.
```
这里一共三条命令，第一条被加入到事务队列中，但后面两条都报错，最终事务会被取消执行。

2. 运行错误。运行错误是指在命令执行时出现的错误，比如使用散列类型的命令操作集合类型的键，运行错误在实际执行之前是无法被Redis识别的，所以在事务里这样的命令是会被Redis接受并执行的。如果事务里的一条命令出现了运行错误，事务里其他的命令依然会继续执行，包括出错命令之后的命令！例如：
```；/
 > MULTI
 OK
 > SET key 1
 QUEUED
 > SADD key 2
 QUEUED
 > SET key 3
 QUEUED
 > EXEC
 1)OK
 2) (error) ERR Operation against a key holding
 the wrong kind of value
 3)OK
 > GET key
 "3"
```
虽然SADD那一步执行错误，但接下来的SET key 3仍然执行了，key的最终结果为3。
遇到这种情况怎么办呢？可能第一反应是如何回滚，但Redis并没有提供事务的回滚功能，一旦出现这类运行错误，需要使用者自行处理，手动将数据库修复为事务执行前的状态。

但换个角度，正是由于Redis舍弃了事务的回滚功能，使得Redis在的事务简洁、快速。这里的第一种错误可以被Redis识别，第二种错误也是可以提前避免的。

### WATCH命令
在一个事务中只有当所有命令都依次执行完后才能得到每个结果的返回值，可是有些情况下需要先获得一条命令的返回值，然后再根据这个值执行下一条命令。如果直接获取，可能之后这个值已经被别的客户端更改过了，那么后面都是基于过时的值在做计算了。
为了避免这种情况，就可以使用WATCH。

WATCH命令可以监控一个或多个键，一旦其中有一个键被修改（或删除），之后的事务就不会执行。监控一直持续到EXEC命令（事务中的命令是在EXEC之后才执行的，所以在MULTI命令后可以修改WATCH监控的键值），如：
```
 > SET key 1
 OK
 > WATCH key
 OK
 > SET key 2
 OK
 > MULTI
 OK
 > SET key 3
 QUEUED
 > EXEC
 (nil)
 > GET key
 "2"
```
这个例子中，在执行WATCH命令后、事务执行前修改了key的值（即SET key 2），所以最后事务中的命令SET key 3没有执行，EXEC命令返回空结果。

执行EXEC命令后会取消对所有键的监控，如果不想执行事务中的命令也可以使用UNWATCH命令来取消监控。

### 生存时间
Redis中可以为键设置生存时间，可以利用这一特性，实现缓存、验证码等功能。
```
 EXPIRE key seconds
 PEXPIRE key milliseconds
```
其中seconds/milliseconds参数表示键的生存时间，单位是秒/毫秒。返回1表示设置成功，返回0则表示键不存在或设置失败。
再次执行EXPIRE/PEXPIRE会重置键的生存时间。

还可以通过设置截至时间的方式让键失效：
```
 EXPIREAT key unixtimespan
 PEXPIREAT key unixtimespan
```
同样的，PEXPIREAT的单位是毫秒。

如果想取消键的生存时间设置，即将键恢复成永久有效，可以使用PERSIST命令：
```
 PERSIST key
```
如果生存时间被成功清除则返回1，如果键不存在或键本来就是永久的则返回0。
除了这种方法，用SET命令为键复制也会清除键的生存时间。


查看一个键剩余的生存时间：
```
 TTL key
 PTTL key
```
TTL和PTTL返回值的单位分别是秒和毫秒。
但如果键不存在，会返回-2。如果键没有设置生存时间，或生存时间设置被清除，会返回-1。



基于生存时间可以将Redis用作缓存。
当服务器内存有限时，如果大量地使用缓存键且生存时间设置得过长就会导致Redis占满内存；另一方面如果为了防止Redis占用内存过大而将缓存键的生存时间设得太短，就可能导致缓存命中率过低并且大量内存白白地闲置。实际开发中会发现很难为缓存键设置合理的生存时间，为此可以限制Redis能够使用的最大内存，并让Redis按照一定的规则淘汰不需要的缓存键，这种方式在只将Redis用作缓存系统时非常实用。

修改配置文件的maxmemory参数可以限制Redis最大可用内存大小，当超出了这个限制时Redis会依据maxmemory-policy参数指定的策略来删除不需要的键，直到Redis占用的内存小于指定内存。

maxmemory-policy可选的规则如下：
规则 | 说明
- | -
 volatile-lru | 使用LRU算法删除一个键(只对设置了生存时间的键)
 allkeys-lru | 使用LRU算法删除一个键
 volatile-random | 随机删除一个键(只对设置了生存时间的键)
 allkeys-random | 随机删除一个键
 volatile-ttl | 删除生存时间最近的一个键
 noeviction | 不删除键，只返回错误

LRU（LeastRecently Used）算法即“最近最少使用算法”，认为最近最少使用的键在未来一段时间内也不会被用到，所以当需要空间时这些键会首先被删除。
