
- Lua基本语法
- 表类型
- 函数
- Redis执行脚本
- KEYS与ARGV 
- 沙盒与随机数 
- 脚本相关命令 
- 原子性和执行时间

Lua是一种高效的轻量级脚本语言，能够方便地嵌入到其他语言中使用。在Redis中，借助Lua脚本可以自定义扩展命令。

### Lua基本语法
#### 数据类型
- 空(nil)，没有赋值的变量或表的字段值都是nil
- 布尔(boolean)
- 数字(number)，整数或浮点数
- 字符串(string),字符串可以用单引号或双引号表示，可以包含转义字符如\n \r等
- 表(table),表类型是Lua语言中唯一的数据结构，既可以当数组又可以当字典，十分灵活
- 函数(function)，函数在Lua中是一等值(first-class-value)，可以存储在变量中、作为函数的参数或返回结果。

#### 变量
Lua的变量分为全局变量和局部变量，全局变量无需声明就可以直接使用，默认值是nil。
全局变量：
```
a=1 -- 为全局变量a赋值
print(b) -- 无需声明即可使用，默认值是nil
```

局部变量：
```
local c -- 声明一个局部变量c，默认值是nil
local d=1 -- 声明一个局部变量d并赋值为1
local e,f -- 可以同时声明多个局部变量
```
但在Redis中，为了防止脚本之间相互影响，只允许使用局部变量。

#### 赋值
Lua支持多重赋值，如：
```
local a,b=1,2 --a的值是1，b的值是2
local c,d=1,2,3 --c的值是1，d的值是2，3被舍弃了
local e,f =1 --e的值是1，f的值是nil
```

#### 操作符
1. 数学操作符，包括常见的+ - * \ %(取模) -(一元操作符，取负)和幂运算符号^。

2. 比较操作符，包括==  ~=(不等于) > < >= <=。
比较操作符不会对两边的操作数进行自动类型转换：
```
pring(1=='1') --结果为false
print({'a'}=={'a'}) -false,表类型比较的是二者的引用
```
3. 逻辑操作符
包括下面三个：
not，根据操作数的真和假相应地返回false和true；
and，a and b中如果a是真则返回b，否则返回a；
or，a or b中，如果a是真则返回a，否则返回b。
这些根据操作符短路的原理可以推断出。

```
print(1 and 5)  --5
print(1 or 5)  --1
print(not 0)  --false
print('' or 1)  --''
```
只要操作数不是nil或false，逻辑操作符就认为操作数是真，否则是假。而且即使是0或空字符串也被当作真，所以上面的代码中print(not 0)的结果为false,print('' or 1)的结果为''。

4. 连接操作符
Lua中的连接操作符为'..'，用来连接两个字符串。

5. 取长度操作符
```
 print(#'hello')  --5
```

#### if语句
Lua中if语句的格式为
```
if condition then
    ...
else if condition then
    ...
else
    ...
end
```
由于Lua中只有nil和false才认为是假，这里也需要注意避坑，比如Redis中EXISTS命令返回1和0分别表示存在或不存在,类似下面的写法if条件将始终为true：
```
if redis.call('EXISTS','key1') then
    ...
```
所以需要写成：
```
if redis.call('EXISTS','key1')==1 then
    ...
```

#### 循环语句
Lua中的循环语句有四种形式：
```
while condition do
    ...
end
```

```
repeat
    ...
until condition
```

```
for i=初值, 终值, 步长 do
    ...
end
```
其中步长为1时可以省略。

```
for 变量1,变量2,...,变量N in 迭代器 do
    ...
end
```

### 表类型
表是Lua中唯一的数据结构，可以理解为关联数组，除nil之外的任何类型的值都可以作为表的索引。
#### 表的定义和赋值
```
-- 表的定义
a={} --将变量a赋值为一个空表
-- 表的赋值
a['field']='value' --将field字段赋值为value
print(a.field) --a['field']可以简化为a.field

-- 定义的同时赋值
b={
    name='bom',
    age=7
}
-- 取值
print(b['age'])
print(b.age)
```

当索引为整数的时候表和传统的数组一样，但需要注意的是Lua的索引是从1开始的。
```
a={}
a[1]='bob'
a[2]='daffy'
```
上面的定义和赋值的过程可以直接简化为：
```
a={'bob','daffy'}
```
取值：
```
print(a[1])
```
#### 表的遍历
之前介绍的这种类型的for循环可以用于表的遍历：
```
for 变量1,变量2,...,变量N in 迭代器 do
    ...
end
```

```
a={'bob','daffy'}

for index,value in ipairs(a) do
    print(index) 
    print(value) 
end
```
ipairs用于数组的遍历，index和value分别为元素的索引和值，变量名不是必须为index和value，可以自定义。
或者：
```
for i=1, #a do
    print(i)
    print(a[i])
end
```
通过#a可以去到数组a的长度。

对于非数组的遍历，可以使用pairs
```
b={
    name='bom',
    age=7
}

for key,value in pairs(b) do
    print(key) 
    print(value) 
end
```
变量名不是必须为key和value，可以自定义。


### 函数
函数的定义为：
```
function(参数列表)
    ...
end
```
实际使用中可以将其赋值给一个局部变量，如：
```
local square=function(num)
    return num * num
end
```
还可以简化为：
```
local function square(num)
    return num * num
end
```
如果实参的个数小于形参的个数，则没有匹配到的形参的值为nil；如果实参的个数大于形参的个数，则多出的实参会被忽略。如果希望参数可变，可以用...表示形参。

### 在脚本中调用Redis命令
在脚本中使用redis.call可以调用Redis命令
```
redis.call('SET','foo','bar')
```
redis.call的返回值就是Redis命令的执行结果。针对Redis的不同返回类型，redis.call会将其转换为对应的Lua的数据类型，两者的对应关系为：
Redis返回类型 | Lua数据类型
- | -
整数回复 | 数字类型
字符串回复 | 字符串类型
多行字符串回复 | 表类型(数组形式)
状态回复 | 表类型(只有一个ok字段存储状态信息)
错误回复 | 表类型(只有一个err字段存储错误信息)

Redis的nil回复会被转换为false。

Lua脚本执行完毕后可以通过return将结果返回给Redis客户端，这是又会将Lua的数据类型转换为Redis的返回类型，过程与上面的表格相反。

redis.pcall函数与redis.call的功能相同，但redis.pcall在执行出错时会记录错误并继续执行，而redis.call则会中断执行。



### Redis执行脚本
#### EVAL
在Redis客户端通过EVAL命令可以调用脚本，其格式为：
```
EVAL 脚本内容 key参数的数量 [key...] [arg...]
```
例如用脚本来设置键的值，就是这样的：
```
EVAL "return redis.call('SET',KEYS[1],ARGV[1])" 1 foo bar
```
通过key和arg这两类参数向脚本传递数据，它们的值可以在脚本中分别使用KEYS和ARGV两个表类型的全局变量访问。key参数的数量是必须指定的，没有key参数时必须设为0，EVAL会依据这个数值将传入的参数分别存入KEYS和ARGV两个表类型的全局变量。

#### EVALSHA
如果脚本比较长，每次调用脚本都将整个脚本传给Redis会占用较多的带宽。而使用EVALSHA命令可以脚本内容的SHA1摘要来执行脚本，该命令的用法和EVAL一样，只不过是将脚本内容替换成脚本内容的SHA1摘要。Redis在执行EVAL命令时会计算脚本的SHA1摘要并记录在脚本缓存中，执行EVALSHA命令时Redis会根据提供的摘要从脚本缓存中查找对应的脚本内容，如果找到了则执行脚本，否则会返回错误：“NOSCRIPT No matching script. Please use EVAL.”。

具体使用时，可以先计算脚本的SHA1摘要，并用EVALSHA命令执行脚本，如果返回NOSCRIPT错误，就用EVAL重新执行脚本。

### KEYS与ARGV 
前面提到过向脚本传递的参数分为KEYS和ARGV两类，前者表示要操作的键名，后者表示非键名参数。但这一要求并不输强制的，比如设置键值的脚本：
```
EVAL "return redis.call('SET',KEYS[1],ARGV[1])" 1 foo bar
```
也可以写成：
```
EVAL "return redis.call('SET',ARGV[1],ARGV[2])" 0 foo bar
```
虽然规则不是强制的，但不遵守这样的规则可能会为后续带来不必要的麻烦。比如Redis 3.0之后支持集群功能，开启集群后会将键发布到不同的节点上，所以在脚本执行前就需要知道脚本会操作哪些键以便找到对应的节点，而如果脚本中的键名没有使用KEYS参数传递则无法兼容集群。

### 沙盒与随机数 
Redis限制脚本只能在沙盒中运行，只允许脚本对Redis的数据进行处理，而禁止使用Lua标准库中与文件或系统调用相关的函数，Redis还通过禁用脚本的全局变量的方式保证每个脚本都是相对隔离、不会互相干扰的。

使用沙盒一方面可保证服务器的安全性，还可确保可以重现（脚本执行的结果只和脚本本身以及传递的参数有关）。

Redis还替换了math.random和math.randomseed函数，使得每次执行脚本时生成的随机数列都相同。如果希望获得不同的随机数序列，可以采用提前生成随机数并通过参数传递给脚本，或者提前生成随机数种子的方式。

集合类型和散列类型的字段是无序的，所以SMEMBERS和HKEYS命令原本会返回随机结果，但在脚本中调用这些命令时，Redis会对结果按照字典顺序排序。

对于会产生随机结果但无法排序的命令，比如SPOP，SRANDMEMBER, RANDOMKEY, TIME，Redis会在这类命令执行后将该脚本状态标记为lua_random_dirty，此后只允许调用只读命令，不允许修改数据库的值，否则会返回错误：“Write commands not allowed after non deterministic commands.”

### 脚本相关命令 
#### SCRIPT LOAD
EVAL命令会执行脚本，并将脚本计算SHA1、加入到脚本缓存中，如果只是希望缓存脚本而不执行，就可以使用SCRIPT LOAD，返回值是脚本的SHA1结果：
```
> SCRIPT LOAD "return redis.call('SET',KEYS[1],ARGV[1])"
"cf63a54c34e159e75e5a3fe4794bb2ea636ee005"
```

#### SCRIPT EXISTS
通过SHA1查询某个脚本是否被缓存，可以查询多个SHA1。参数必须是完整的SHA1，而不能像docker只输前几位。返回结果1表示存在。

#### SCRIPT FLUSH
Redis将脚本加入到缓存后会永久保留，如果要清空缓存可以使用SCRIPT FLUSH。

#### SCRIPT KILL
用于终止正在执行的脚本

### 原子性和执行时间
Redis的脚本执行是原子的，脚本执行期间其他命令不会被执行，必须等待上一个脚本执行完成。

但为了防止某个脚本执行时间过长导致Redis无法提供服务（比如陷入死循环），Redis提供了lua-time-limit参数限制脚本的最长运行时间，默认为5秒钟。当脚本运行时间超过这一限制后，Redis将开始接受其他命令，但为了确保脚本的原子性，新的脚本仍然不会执行，而是会返回“BUSY”错误。

可以打开两个redis-cli实例A和B来验证，首先在A执行一个死循环脚本：
```
EVAL "while true do end" 0
```
这时在实例B执行GET key1会返回：
(error) BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.

如果按照错误提示，在B执行SCRIPT KILL，这时在实例A的脚本会被终止，并返回：
(error) ERR Error running script (call to f_694a5fe1ddb97a4c6a1bf299d9537c7d3d0f84e7): @user_script:1: Script killed by user with SCRIPT KILL...

但如果A已经对Redis的数据做了修改，则SCRIPT KILL无法将其终止，A执行：
```
EVAL "redis.call('SET','foo','bar') while true do end" 0
```
如果在B尝试KILL脚本，会返回错误：
(error) UNKILLABLE Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.

这时就只能通过SHUTDOWN NOSAVE命令强行终止Redis。SHUTDOWN NOSAVE与SHUTDOWN命令的区别在于，SHUTDOWN NOSAVE将不会进行持久化操作，所有发生在上一次快照后的数据库修改都会丢失！
