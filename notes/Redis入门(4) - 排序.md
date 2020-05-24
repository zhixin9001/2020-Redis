- SORT命令
- LIMIT参数
- BY参数
- GET参数 
- STORE参数
- 排序性能优化

很多场合需要对元素进行排序，这时除了使用有序集合外，还可以借助Redis提供的SORT命令来排序。

### SORT命令
SORT命令可以对列表类型、集合类型和有序集合类型的键进行排序。
```
 SORT key
 SORT key DESC
 SORT key ALPHA
```
SORT命令会根据元素自身的值进行排序，在对有序集合类型排序时会忽略元素的分数。
默认按从小到大的顺序排列，增加DESC参数可以按照倒序排列。
如果元素为非数字，增加ALPHA参数可以按照字典顺序排列。如果直接对非数字元素排序会报错：
```
 (error)ERR One or more scores can't be converted into double
```
如果没有加ALPHA参数的话，SORT命令会尝试将所有元素转换成双精度浮点数来比较，如果无法转换则会提示错误。

### LIMIT参数
如果返回结果数量较多需要分页，可以使用LIMIT参数
```
SORT key DESC LIMIT offset count
```
表示在排序结果中，跳过前offset个元素，获取之后的count个元素。

### BY参数
很多情况下列表（或集合、有序集合）中存储的元素值代表的是对象的ID，单纯对这些ID自身排序有时意义并不大。更多的时候会希望根据ID对应的对象的某个属性进行排序。
这种情况下可以使用BY参数：
```
 SORT key BY reference
```
其中reference表示排序的参考键，会根据参考键的值来排序，而不再是列表或集合中元素自身的值。
比如：
```
SORT tag:ruby:posts BY post:* -> time DESC
```
这里tag:ruby:posts存储了文章的ID，post:\*为散列类型，其中的time字段为文章的发布时间，这样就可以将文章ID根据发布时间排序了。执行的时候，对每个元素使用元素的值替换参考键中的第一个“\*”并获取其值，然后依据该值对元素排序。

上面是基于散列类型排序的写法，基于字符串排序更简单：
```
SORT sortbylist BY itemscore:* -> time DESC
```

BY参数排序有下面几种特殊情况：
- 当参考键名不包含“\*”时（即常量键名，与元素值无关），SORT命令将不会执行排序操作，因为Redis认为这种情况是没有意义的(所有要比较的值都一样）。
- 如果几个元素的参考键值相同，则SORT命令会再比较元素本身的值来决定元素的顺序。
- 当某个元素的参考键不存在时，会默认参考键的值为0。

### GET参数 
SORT命令默认返回的是键本身的元素被排序后的结果，而使用GET参数可以指定返回键值。
比如前面按照文章发布时间排序后，并不仅仅获得文章ID，而是更进一步获取文章的标题，可以这样写：
```
SORT tag:ruby:posts BY post:* -> time DESC GET post:*->title
```

而且，在一个SORT命令中可以使用多个GET参数（BY参数只能有一个）：
```
SORT tag:ruby:posts BY post:* -> time DESC GET post:*-> title GET post:* -> time
```

如果仍然需要文章的ID，可以使用GET #：
```
SORT tag:ruby:posts BY post:* -> time DESC GET post:*-> title GET post:* -> time GET #
```
这样最终的结果就包含了文章的标题、发布时间和ID。


### STORE参数
默认情况下SORT会直接返回排序结果，如果希望保存排序结果，可以使用STORE参数，比如要把排序的结果保存到sort.result键中：
```
SORT tag:ruby:posts BY post:* -> time DESC STORE sort.result
```
保存后的键的类型为列表类型，如果键已经存在则会覆盖它。加上STORE参数后SORT命令的返回值为结果的个数。

实际使用中，常常将STORE命令与之前学过的EXPIRE结合，来缓存排序的结果。

### 排序性能优化
SORT是Redis中最强大最复杂的命令之一，但如果使用不好也很容易成为性能的瓶颈。
SORT命令的时间复杂度是O(n+mLog m)，其中:
- n表示要排序的列表（集合或有序集合）中的元素个数
- m表示要返回的元素个数
当n较大的时候SORT命令的性能相对较低，并且Redis在排序前会建立一个长度为n的容器来存储待排序的元素，虽然是一个临时的过程，但如果同时进行较多的大数据量排序操作则会严重影响性能。

所以在使用SORT命令时要注意这几点：
- 减小n，尽量减少待排序键中元素的数量；
- 减少m，使用LIMIT参数只获取需要的数据；
- 如果排序的数据量较大，经常需要排序，可以使用STORE和EXPIRE将结果缓存起来。
