
- 使用列表实现任务队列
- 优先级队列
- 按照规则订阅

Redis也可以作为任务队列。任务队列顾名思义，就是“传递任务的队列”。任务队列与消息队列什么区别呢？任务队列是逻辑模型，而消息队列是通信模型，两者是不同层次的抽象，用消息队列可以实现任务队列。

与任务队列进行交互的实体有两类，一类是生产者（producer），一类是消费者（consumer）。生产者会将需要处理的任务放入任务队列中，而消费者则不断地从任务队列中读入任务信息并执行。

使用任务队列可以达到松耦合的效果，生产者和消费者无需知道彼此的实现细节，只需要约定好任务的描述格式。这使得生产者和消费者可以由不同的团队使用不同的编程语言编写。

而且这种模型下，消费者可以很容易地进行扩展，并可以分布在不同的服务器，降低单台服务器的负载。

### 使用列表实现任务队列
Redis的列表类型支持从一边PUSH、从另一边POP的操作，用作队列非常合适。只需生产者通过LPUSH命令将任务加入列表中，另一边让消费者不断使用RPOP命令从列表中取出即可。但不希望消费者不停RPOP，可以用BRPOP改进。BRPOP在列表中没有元素时BRPOP命令会一直阻塞住连接，直到有新元素加入。

BRPOP命令接收两个参数，第一个是键名，第二个是超时时间，单位是秒。当超过了此时间仍然没有获得新元素的话就会返回nil。超时时间设置为0时，如果没有新元素加入列表就会永远阻塞下去。

#### 开启多个客户端
为了测试消息通知，可以开启多个redis-cli实例。
如果使用的是redis的docker镜像的话，可以先启动一个redis容器作为redis server：
```
docker run --name redisserver -d -p 6379:6379 redis redis-server
```
redis-server命令可以启动redis server。
然后可以多开几个终端，都执行下面的命令以启动redis容器，并连接之前启动的redisserver，这样就可以作为redis client使用了：
```
docker run -it --link redisserver:redis --rm redis redis-cli -h redisserver -p 6379
```

在实例A中输入：
```
 BRPOP queue 0
```
接下来实例A会一直处于阻塞状态。然后在实例B中向队列加入元素：
```
 LPUSH queue taskk
```
这时马上就可以在实例A中看到返回结果，同时可以验证queue中刚刚加入的元素已经被取走了。
与BRPOP对应，还有BLPOP命令。

### 优先级队列
BRPOP命令可以同时接收多个键，如：
```
 BRPOP queue:1 queue:2 queue:3 0
```
这时它会同时检测这几个键，如果所有键都没有元素则阻塞，如果其中有一个键有元素则会从该键中弹出元素。
如果多个键都有元素则按照从左到右的顺序取第一个键中的一个元素。

借助这些特性可以实现优先级队列，queue:1 queue:2 queue:3三个队列中，queue:1的优先级最高，这个队列中的任务会优先被消费。

比如在一个网站中，向用户发送的邮件中账号激活邮件的优先级比活动推广邮件的优先级高，这时就可以将这两种邮件放在不同的队列，用BRPOP来优先发送激活邮件。

### “发布/订阅”模式
除了实现任务队列外，Redis还提供了一组命令可以让开发者实现“发布/订阅”（publish/subscribe）模式。“发布/订阅”模式同样可以实现进程间的消息传递。
用列表类型实现的任务队列中，一个任务只能被一个消费者消费，多个生产者都可以向队列推送任务；
在“发布/订阅”模式下，一条消息可以被所有订阅这个频道订阅者接收，多个发布者也都可以向指定的频道推送消息。

在redis-cli实例A发布消息：
```
 PUBLISH channel.1 msg1
```
PUBLISH命令的返回值表示接收到这条消息的订阅者数量。发出去的消息不会被持久化，客户端订阅channel.1后只能收到后续发布到该频道的消息，之前发送的是收不到的。

在redis-cli实例B订阅channel.1
```
 SUBSCRIBE channel.1
```
SUBSCRIBE命令可以同时订阅多个频道。

执行SUBSCRIBE命令后客户端会进入订阅状态，处于此状态下客户端只能使用属于“发布/订阅”模式的命令，这些命令有SUBSCRIBE/UNSUBSCRIBE/PSUBSCRIBE/PUNSUBSCRIBE。

但在redis-cli中进入订阅状态后是无法退出到非订阅状态的，只能关闭cli。

实例B已经订阅了channel.1，这时如果再从实例A向channel.1发布消息msg1，实例B就会收到下面的响应：
```
1) "message"
2) "channel.1"
3) "msg1"
```
第一行是消息类型，消息类型的取值有subscribe/message/unsubscribe。
当消息类型为message时，第二行表示频道名称，第三行是消息的内容；
当消息类型为subscribe时，表示订阅成功的反馈信息。第二行是订阅成功的频道名称，第三行是当前客户端订阅的频道数量。
当消息类型为unsubscribe时，第二行是对应的频道名称，第三行是当前客户端订阅的频道数量，当此值为0时客户端会退出订阅状态。

### 按照规则订阅
除了可以使用SUBSCRIBE命令订阅指定名称的频道外，还可以使用PSUBSCRIBE命令按照按照指定的规则订阅多个频道。如：
```
> PSUBSCRIBE channel.*
 Reading messages... (press Ctrl-C to quit)
 1) "psubscribe"
 2) "channel.*"
 3) (integer)1"
```
订阅规则支持glob风格通配符格式，channel.*中*可以匹配任意数量的字符。
使用PSUBSCRIBE命令可以重复订阅一个频道，如某客户端执行了PSUBSCRIBE channel.? channel.?*，这时向channel.2发布消息后该客户端会收到两条消息。

PUNSUBSCRIBE命令可以退订指定的规则
```
PUNSUBSCRIBE [pattern ...]
```
如果没有指定pattern参数，则会退订所有由PSUBSCRIBE订阅的规则。

使用PUNSUBSCRIBE命令只能退订通过PSUBSCRIBE命令订阅的规则，不会影响直接通过SUBSCRIBE命令订阅的频道；同样UNSUBSCRIBE命令也不会影响通过PSUBSCRIBE命令订阅的规则。

