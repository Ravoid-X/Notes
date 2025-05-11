在官网 https://redis.io/commands 查看
## 通用指令
1. KEYS：查看符合模板的所有key
2. DEL：删除一个指定的key
3. EXISTS：判断key是否存在
4. EXPIRE：给一个key设置有效期，有效期到期时该key会被自动删除
5. TTL：查看一个KEY的剩余有效期
## 单数据操作
1. 添加/修改数据
```
set key value
```
2. 获取数据
```
get key
```
3. 删除数据
```
del key
```
# 多数据操作
1. 添加/修改数据\
mset相对于set操作能够节省数据发送返回所用时间，但是需要考虑数据量，如果过大可能影响性能，阻塞其他命令执行。
```
mset key1 value1 key2 value2 …
```
2. 获取数据
```
mget key1 key2 …
```
3. 追加信息到原始信息后部（如果原始信息存在就追加，否则新建）
```
append key value
```
4. 获取数据字符个数
```
strlen key
```
## 应用举例
1. 应用场景：大型企业级应用中，分表操作是基本操作，使用多张表存储同类型数据，但是对应的主键 id 必须保证统一性，不能重复。 Oracle 数据库具有 sequence 设定，可以解决该问题，但是 MySQL数据库并不具有类似的机制，该如何解决？\
解决方案： 设置数值数据增加指定范围的值
```
incr key
incrby key increment
incrbyfloat key increment
设置数值数据减少指定范围的值
decr key
decrby key increment
```
2. 应用场景： “中国好声音”启动海选投票，只能通过微信投票，每个微信号每 4 小时只能投1票。电商商家开启热门商品推荐，热门商品不能一直处于热门期，每种商品热门期维持3天， 3天后自动取消热门。新闻网站会出现热点新闻，热点新闻最大的特征是时效性，如何自动控制热点新闻的时效性。\
解决方案： 设置数据具有指定的生命周期
```
setex key seconds value
psetex key milliseconds value
```
3. 应用场景：主页高频访问信息显示控制，例如新浪微博大V主页显示粉丝数与微博数量。\
```
在redis中为大V用户设定用户信息，以用户主键和属性值作为key，后台设定定时刷新策略即可
eg: user:id:3506728370:fans → 12210947
eg: user:id:3506728370:blogs → 6164
eg: user:id:3506728370:focuss → 83
在redis中以json格式存储大V用户信息，定时刷新（也可以使用hash类型）
eg: user:id:3506728370 →
{"id":3506728370,"name":"中国新闻网","fans":12210862,"blogs":6164, "focus":83}

key 的设置约定
数据库中的热点数据key命名惯例
     表名  ：主键名 ： 主键值：     字段名
eg1：order ：id     ： 29437595    ：name
eg2：equip ：id     ： 390472345   ：type
eg3：news  ：id     ： 202004150   ：title
```