
- Redis中的数据类型
- 字符串
- 散列
- 列表
- 集合
- 有序集合

### Redis中的数据类型
Redis定义了这几种数据类型：
- string（字符串）
- hash（散列）
- list（列表）
- set（集合）
- zset（有序集合）
后面会介绍它们各自的特点和使用场景。可以用TYPE命令来获取键的类型。

### 字符串
字符串类型是Redis中最基本的数据类型，它能存储任何形式的字符串，包括二进制数据。可以用其存储用户的邮箱、JSON化的对象甚至是一张图片。一个字符串类型键允许存储的数据的最大容量是512 MB。
字符串类型是其他4种数据类型的基础，其他数据类型和字符串类型的差别从某种角度来说只是组织字符串的形式不同。例如，列表类型是以列表的形式组织字符串，而集合类型是以集合的形式组织字符串。

#### 赋值、取值
```
SET key value
GET key
```
SET成功返回OK，GET键不存在时返回空(nil)。

#### 增加、减少数字
当操作的字符串是数字形式时，可以用这些命令方便地增加、减少数字
递增、递减操作分别为：
```
INCR key
DECR key
```
当操作的键不存在时，会先创建一个默认为0的值，所以使用INCR得到的结果为1。
如果被操作的键值不是整数，会给出错误提示： (error) ERR value is not an integer or out of range

增加、减少指定整数：
```
INCRBY key increment
DECRBY key decrement
```
INCRBY key1 3就相当于DECRBY key1 -3

增加、减少指定浮点数，支持双精度浮点数。
```
INCRBYFLOAT key increment
DECRBYFLOAT key decrement
```
这些命令都是原子操作，可以保证多客户端连接的并发场景下读取写入不会出错。

#### 向尾部追加值
```
APPEND key value
```
如果键不存在则将该键的值设置为value。返回值是追加后字符串的总长度。
```
> SET key1 hello
OK
> APPEND key " world!"
(integer) 12
```
上面的操作得到的就是“hello world”，由于需要在单词之间有空格，append时的值用双引号包裹。

#### 获取字符串长度
```
STRLEN key
```
如果键不存在则返回0。

#### 同时获取/设置多个键值
```
MGET key1 key2 ...
MSET key value key1 value1 ...
```

### 散列
Redis是采用字典结构以键值对的形式存储数据，而散列类型（hash）的键值也是一种字典结构，其存储了字段（field）和字段值的映射，但字段值只能是字符串，不支持其他数据类型，所以不能嵌套存储其他数据类型。Redis的其他数据类型也不支持数据类型的嵌套，比如集合类型的每个元素都只能是字符串。
一个散列类型的键可以最多包含2^32-1个字段。
散列类型适合存储对象，使用类名和ID构成键名，使用字段表示对象的属性，而字段值则存储属性的值，这样散列类型就与最简单的DTO类构成了映射关系。

#### 赋值与取值
```
 HSET key field value
 HGET key field
```
这组命令用来设置、获取单个字段的值。HSET命令不区分插入和更新操作，修改数据时不用事先判断字段是否存在来决定要执行的是插入还是更新，可以通过命令的返回结果来判断插入还是更新，插入时返回1，更新时会返回0。当键或者字段不存在时，HSET命令还会自动建立它们。

HMSET、HMGET则可以一次操作多个字段：
```
 HMSET key field value [field value ...]
 HMGET key field [field value  ...]
```

获取键的所有字段使用HGETALL：
```
 HGETALL key
```
这个命令返回的结果是字段和字段值组成的列表：
```
1)"price"
2)"500"
3)"name"
4)"BMW"
```

#### 判断字段是否存在
```
 HEXISTS key field
```
如果存在则返回1，否则返回0，如果键不存在也会返回0。

#### 当字段不存在时赋值
```
HSETNX key field value
```
HSETNX命令只有当字段不存在时才进行赋值，如果字段已经存在，将不执行任何操作。

#### 增加、减少数字
```
HINCRBY key field increment
```
需要减少值时增量设为负值。如果键或者值不存在时将初始化并设置为1。

#### 删除字段
```
HDEL key field1 field2 ...
```
可以删除一个或多个字段，被删除的字段个数作为返回值。

#### 只获取字段名或字段值
```
HKEYS key
HVALS key
```
这组命令会获取键中所有字段的名称或值。

### 列表
列表类型可以存储一个有序的字符串列表，常用的操作是向列表两端添加元素，或者获得列表的某一个片段。列表类型内部是使用双向链表实现的，向列表两端添加元素的时间复杂度为O(1)，获取越接近两端的元素速度就越快，但通过索引访问元素的速度却比较慢。

所以列表类型最适合于访问前某几个元素的场景，比如热点新闻。一个列表类型键最多能容纳的元素数量也是2^32-1个。

#### 向列表两端增加元素
```
 LPUSH key value [value ...]
 RPUSH key value [value ...]
```
LPUSH和RPUSH分别用来先列表的左边和右边增加元素，返回值表示增加元素后列表的长度。可以同时增加多个元素。

#### 从列表两端弹出元素
```
 LPOP key
 RPOP key
```
LPOP和RPOP分别从列表的左边和右边弹出一个元素，返回值元素的内容，如果列表为空，返回(nil)。

#### 获取列表中元素的个数
```
 LLEN key
```
如果列表为空或不存在，则返回0。

#### 获得列表片段
```
 LRANGE key start stop
```
返回索引从start到stop之间的所有元素（包含两端的元素），与大多数人的直觉相同，Redis的列表起始索引，左边为从0开始，右边为从-1开始，-2表示右边第二个元素，以此类推。
如果start的索引位置比stop的索引位置靠后，则会返回空列表；如果stop大于实际的索引范围，则会返回到列表最右边的元素。

#### 删除列表中指定的值
```
LREM key count value
```
删除列表中前count个值为value的元素，返回值是实际删除的元素个数。
count值的正负会决定从删除开始的方向：
- 当count>0时LREM命令会从列表左边开始删除前count个值为value的元素；
- 当count<0时LREM命令会从列表右边开始删除前|count|个值为value的元素；
- 当count=0时LREM命令会删除列表中所有值为value的元素。

#### 获得/设置指定索引的元素值
```
LINDEX key index
LSET key index value
```
与LREM类似，index >= 0表示从左边开始计算索引，index < 0则从右边开始。

#### 只保留列表指定片段
```
LTRIM key start end
```
删除指定索引范围之外的所有元素，其指定列表范围的方法和LRANGE命令相同。

#### 向列表中插入元素
```
LINSERT key BEFORE|AFTER privot value
```
LINSERT命令首先会在列表中从左到右查找值为pivot的元素，然后根据第二个参数是BEFORE还是AFTER来决定将value插入到该元素的前面还是后面。命令的返回值是插入后列表的元素个数。

#### 将元素从一个列表转到另一个列表
```
RPOPLPUSH source destination
```
这个命令会先对source执行RPOP命令，从它的右边弹出一个元素，再对destination执行LPUSH命令，将元素加入到这个列表的左边，并返回这个元素的值。整个过程是原子的。
RPOPLPUSH命令可以很直观地在多个队列中传递数据。当source和destination相同时，RPOPLPUSH命令会不断地将队尾的元素移到队首，同时列表中仍然可以增加、删除元素。

### 集合
同列表类型相比，集合类型中的元素不能相同，同时元素没有顺序。
一个集合类型键可以存储的字符串数量最多也是2^32 -1个。
集合类型的常用操作是向集合中加入或删除元素、判断某个元素是否存在等，由于集合类型在Redis内部是使用值为空的散列表（hashtable）实现的，所以这些操作的时间复杂度都是O(1)。最方便的是多个集合类型键之间还可以进行并集、交集和差集运算。

#### 增加/删除元素
```
 SADD key member [member ...]
 SREM key member [member ...]
```
这组命令用来向集合中增加或删除一个或多个元素，增加时如果键不存在则会自动创建。因为在一个集合中不能有相同的元素，所以增加时如果要加入的元素已经存在于集合中，就会忽略这个元素。命令的返回值是成功加入或删除的元素数量。

#### 获得集合中的所有元素
```
 SMEMBERS key
```

#### 判断元素是否在集合中
```
 SISMEMBER key mumber
```
这项操作的时间复杂度为O(1)，无论集合中有多少个元素，SISMEMBER命令都可以非常快地返回结果。当值存在时返回1，不存在或键不存在时返回0。

#### 获得集合中元素个数
```
SCARD key
```

#### 集合间运算
```
 SDIFF key [key...]
 SINTER key [key...]
 SUNION key [key...]
```
这组命令用来对多个集合进行差集、交集、并集运算。

```
 SDIFFSTORE destination key [key...]
 SINTERSTORE destination key [key...]
 SUNIONSTORE destination key [key...]
```
这组命令会把集合间运算的结果存储到destination中。

### 有序集合
有序集合类型与集合类型相比，它为其中的每个元素都关联了一个分数，这使得可以对元素进行取前N个、获得指定分数范围内的元素等操作。
有序集合中的元素也是各不相同的，但元素的分数可以相同。

有序集合类型与列表类型相比，相似的地方在于二者都是有序的，也都可以获得指定范围的元素；但他们的区别大于共性：
- 在存储结构上，列表类型是通过链表实现的，获取靠近两端的数据速度极快，而当元素增多后，访问中间数据的速度会较慢，所以它更适合很少访问中间元素的场景；有序集合类型则是使用散列表和跳跃表实现的，所以即使读取位于中间部分的数据速度也很快。

- 列表中不能简单地调整某个元素的位置，但是在有序集合中通过更改元素的分数就可以做到。

- 有序集合要比列表类型更耗费内存。

#### 增加元素
```
 ZADD key score member [score member ...]
```
ZADD命令用来向有序集合中加入若干元素和该元素的分数，如果该元素已经存在则会用新的分数替换原有的分数。命令的返回值是新加入到集合中的元素个数（不包含之前已经存在的元素）。
这里设置的分数可以是整数，也可以是双精度浮点数，还可以设置为+inf和-inf来分别表示正无穷和负无穷。

#### 获得元素的分数
```
 ZSCORE key member
```

#### 获得排名在某个范围的元素列表
```
 ZRANGE key start stop [WITHSCORES]
 ZREVRANGE key start stop [WITHSCORES]
```
ZRANGE和ZREVRANGE命令分别按照元素分数从小到大和从大到小的的顺序返回索引从start到stop之间的所有元素（包含两端的元素）。
在命令的尾部加上WITHSCORES可以同时获得元素的分数。
如果两个元素的分数相同，Redis会按照字典顺序（即"0" ＜ "9" ＜"A" ＜ "Z" ＜ "a" ＜ "z"这样的顺序）来进行排列，如果元素是中文，会按照中文编码来排序。

索引>=0表示从前往后查找，为<0表示从后往前查找。
```
 ZRANGE key1 0 4
 ZRANGE key1 1 -1
```

#### 获得指定分数范围的元素
```
 ZRANGEBYSCORE key min max [WITHSCORE] [LIMIT offset count]
```
按照元素分数从小到大的顺序返回分数在min和max之间的元素，默认包含min和max，如果希望分数范围不包含端点值，可以在分数前加上'('符号
```
 ZRANGEBYSCORE key1 (60 80
```
这样写获取的范围就是 60< score <= 80。

这里也同样可以用-inf和+inf分别表示负无穷和正无穷。

用WITHSCORES可以同时获得元素的分数。

[LIMIT offset count]表示在获得的元素列表的基础上向后偏移offset个元素，并且只获取前 count个元素。
比如想获得分数高于60分的从第二个人开始的3个人：
```
 ZRANGEBYSCORE key1 60 +inf LIMIT 1 3
```
与ZRANGEBYSCORE对应的，还有ZREVRANGEBYSCORE，它会按照元素分数从大到小的顺序返回分数在max和min之间的元素
```
 ZREVRANGEBYSCORE key max min [WITHSCORE] [LIMIT offset count]
```

#### 增加某个元素的分数
```
 ZINCRBY key increment member
```
返回值是更改后的分数，这个操作也是原子的。

#### 获得指定分数范围内的元素个数
```
 ZCOUNT key min max
```
min max可以指定为+inf -inf，也可以用(来设置边界条件。

#### 删除一个或多个元素
```
 ZREM key member [member ...]
```
返回值是成功删除的元素数量（不包含本来就不存在的元素）。

#### 按照排名范围删除元素
```
 ZREMRANGEBYRANK key start stop
```
按照元素分数从小到大的顺序（即索引0表示最小的值）删除处在指定排名范围内的所有元素，并返回删除的元素数量。索引负值表示从后向前查找。

#### 按照分数范围删除元素
```
 ZREMRANGEBYSCORE key min max
```
删除指定分数范围内的所有元素，参数min和max的特性和ZRANGEBYSCORE命令中的一样。返回值是删除的元素数量。

#### 获得元素的排名
```
 ZRANK key member
 ZREVRANK key member
```
这组命令会分别按照元素分数从小到大、从大到小的顺序获得指定的元素的排名（从0开始，即分数最小或最大的元素排名为0）

