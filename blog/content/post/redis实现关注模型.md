---
title: redis实现关注模型
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["系统设计"]
tags: ["系统设计"]
---

# redis实现关注模型
#技术/系统设计

## 共同关注
a 关注了 b, c, d用户
b 关注了 c, d, e, f用户
a和b共同关注的用户是：c, d

将a和b关注的用户都存入 set，使用SINTER算出交集即可

```sh
SINTER set_a set_b -> {c, d}
```

## 我关注的人也关注了他
a 关注了 b, c, d用户
b 关注了 c, d, e, f用户
c 关注了 e, f, g, h用户
d 关注了 c, f, g

a访问c的首页，显示a关注的人也关注了c
即b, d也关注了c

遍历a关注的所有人（除c以外）的set
```sh
SISMEMBER b_set, c
SISMEMBER d_set, c
...
```
依次判断即可。若关注了几十万人，也无所谓，因为首页只需展示部分，通常是个位数，若查看全部通常是翻页接口，一批一批返回即可

## 我可能认识的人
a可能认识的人e, f
通常使用a和b用户的差集即可
```sh
SDIFF a_set, b_set -> {b,e,f}，刨掉b和a自身，即为可能认识的人e,f
```

# redis实现排行榜模型
每天有一个当日所有新闻的有序集合，每点击一次，给有序集合内部的key的分值+1
```sh
ZINCRBY hot_news_2021_01_02 1 news_name
```

当日的热搜榜前10即
```sh
ZREVRANGE hot_news_2021_01_02 0 9 WITHSCORES
```

七日排行榜
```sh
ZUNIONSTORE hot_news_2021_week_1 7 hot_news_2021_01_01, hot_news_2021_01_02, ... , hot_news_2021_01_07
```

```sh
ZREVRANGE hot_news_2021_week_1 0 9 WITHSCORES
```

# redis分布式锁
## setnx
setnx 返回1，加锁成功
setnx 返回0，锁已存在，加锁失败
问题：当加锁线程挂掉，锁永远无法释放
## setnx + expire
setnx的同时，加上过期时间，防止死锁问题
问题：若加锁线程的业务处理过程阻塞，导致过了超时时间还未完成业务处理，另一线程又可以获取到锁并加锁，加锁线程释放锁时释放了另一线程加上的锁，这样锁其实就永久失效了

## setnx + expire + uuid
setnx，value为一个uuid，只能由加锁线程释放
问题：对于expire时间仍然难以把控，若加锁线程业务处理过程卡住，那么也会导致后续线程可以加锁成功，锁永久失效

## setnx + expire，后台线程刷新锁
setnx，过期时间设置得稍微小一点，后台启动一个线程每隔expire/n的时间监控锁key，若发现该锁存在，则为其延长锁的过期时间
问题：若redis是主从/集群架构，加锁后主节点挂了，从节点还未同步该key，会导致别的线程可以重新加锁，锁失效

## zookeeper，cp
redis保证了ap（可用性+分区一致性），zookeeper保证cp（一致性+分区一致性）
zk可以保证写入了半数以上节点成功后才返回，即使master挂掉，可以让leader顶上重新工作

# redis存储对象数据场景
## SET 
`SET user:1 value(json)`

## MGET,MSET
`MSET user:1:name name1 user:1:school school1`
`MGET user:1:name user:1:school`
使用场景：若某一对象字段非常多时，json编码解码比较耗费时间，网络传输的压力也较大，可以通过这种方式只操作部分字段

## HMSET, HMGET
`HMSET user 1:name name1 1:age age1`
`HMGET user 1:name 1:age`
问题：大key
解决方式：对hash key进行拆分

# 分布式全局序列号
`INCRBY order_id`
每次自增，保证序列号全局唯一；问题：对redis压力较大

优化方式：
`INCRBY order_id 1000`
每次取1000个，一个实例内部缓存住这1000个，然后在内部进行分配

# 电商购物车
hash结构
以用户id为key，商品id为field，商品数量为value

购物车操作：
```
添加商品：hset cart:1001 10088 1
增加商品数量：hincrby cart:1001 10088 1
商品种类总数：hlen cart:1001
删除商品：hdel cart:1001 10088
获取购物车所有商品：hgetall cart:1001
```
问题：在redis cluster架构下不宜大规模使用，大key问题：
数据倾斜，某个节点存储了大key，压力较大

# 阻塞队列
`lpush + brpop`
brpop，若队列里没有元素则一直阻塞，知道有元素被放入

# 微信消息推送
a关注了b和c等大v
b发了微博，消息id为10018
`LPUSH msg:a 10018`
c发了微博，消息id为10086
`LPUSH msg:a 10086`
查看最新消息
`LRANGE msg:a 0 4`

# 微信抽奖小程序
1. 加入抽奖集合
`SADD key userid`
2. 查看所有抽奖用户
`SMEMBERS key`
3. 抽取count名中奖者
`SRANDMEMBER key count`
4. 抽过奖的用户不允许重复抽奖
`SPOP key count`

# 点赞
1. 点赞
`SADD like:msgId userId`
2. 取消点赞
`SREM like:msgId userId`
3. 检查用户是否点过赞
`SISMEMBER like:msgId userId`
4. 获取点赞用户列表
`SMEMBERS like:msgId`
5. 获取点赞用户数
`SCARD like:msgId`