---
title: geohash
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["redis"]
tags: ["redis"]
---

# geohash
#技术/数据库/redis

## 介绍
GeoHash是一种地址编码方法。他能够把二维的空间经纬度数据编码成一个二进制字符串，然后base32后成为一个短字符串。

1. GeoHash将二维的经纬度转换成字符串，比如下图展示了北京9个区域的GeoHash字符串，分别是WX4ER，WX4G2、WX4G3等等，**每一个字符串代表了某一矩形区域**。也就是说，这个矩形区域内所有的点（经纬度坐标）都共享相同的GeoHash字符串，这样既可以保护隐私（只表示大概区域位置而不是具体的点），又比较容易做缓存，比如左上角这个区域内的用户不断发送位置信息请求餐馆数据，由于这些用户的GeoHash字符串都是WX4ER，所以可以把WX4ER当作key，把该区域的餐馆信息当作value来进行缓存，而如果不使用GeoHash的话，由于区域内的用户传来的经纬度是各不相同的，很难做缓存
2. 字符串越长，表示的范围越精确。5位的编码能表示10平方千米范围的矩形区域，而6位编码能表示更精细的区域（约0.34平方千米）
3. **字符串相似的表示距离相近**，这样可以利用字符串的前缀匹配来查询附近的POI信息

## 常见操作
### geoadd
```
geoadd 指令携带集合名称以及多个经纬度名称三元组，注意这里可以加入多个三元组
127.0.0.1:6379> geoadd company 116.48105 39.996794 juejin
(integer) 1
127.0.0.1:6379> geoadd company 116.514203 39.905409 ireader
(integer) 1
127.0.0.1:6379> geoadd company 116.489033 40.007669 meituan
(integer) 1
127.0.0.1:6379> geoadd company 116.562108 39.787602 jd 116.334255 40.027400 xiaomi
(integer) 2
```
### geodist
```
geodist 指令可以用来计算两个元素之间的距离，携带集合名称、2 个名称和距离单位。
127.0.0.1:6379> geodist company juejin ireader km
"10.5501"
127.0.0.1:6379> geodist company juejin meituan km
"1.3878"
127.0.0.1:6379> geodist company juejin jd km
"24.2739"
127.0.0.1:6379> geodist company juejin xiaomi km
"12.9606"
127.0.0.1:6379> geodist company juejin juejin km
"0.0000"
我们可以看到掘金离美团最近，因为它们都在望京。距离单位可以是 m、km、ml、ft，
分别代表米、千米、英里和尺
```
### geopos
```
geopos 指令可以获取集合中任意元素的经纬度坐标，可以一次获取多个。
127.0.0.1:6379> geopos company juejin
1) 1) "116.48104995489120483"
2) "39.99679348858259686"
127.0.0.1:6379> geopos company ireader
1) 1) "116.5142020583152771"
2) "39.90540918662494363"
127.0.0.1:6379> geopos company juejin ireader
1) 1) "116.48104995489120483"
2) "39.99679348858259686"
2) 1) "116.5142020583152771"
2) "39.90540918662494363"
```
### geohash
```
geohash 可以获取元素的经纬度编码字符串，上面已经提到，它是 base32 编码。 你可
以使用这个编码值去 http://geohash.org/${hash}中进行直接定位，它是 geohash 的标准编码
值。
127.0.0.1:6379> geohash company ireader
1) "wx4g52e1ce0"
127.0.0.1:6379> geohash company juejin
1) "wx4gd94yjn0"
```
### georadius
```
GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [ASC|DESC] [COUNT count]
```
georadius，以给定的经纬度为中心， 返回键包含的位置元素当中， 与中心的距离不超过给定最大距离的所有位置元素。
```
redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

redis> GEORADIUS Sicily 15 37 200 km WITHDIST
1) 1) "Palermo"
   2) "190.4424"
2) 1) "Catania"
   2) "56.4413"

redis> GEORADIUS Sicily 15 37 200 km WITHCOORD
1) 1) "Palermo"
   2) 1) "13.361389338970184"
      2) "38.115556395496299"
2) 1) "Catania"
   2) 1) "15.087267458438873"
      2) "37.50266842333162"

redis> GEORADIUS Sicily 15 37 200 km WITHDIST WITHCOORD
1) 1) "Palermo"
   2) "190.4424"
   3) 1) "13.361389338970184"
      2) "38.115556395496299"
2) 1) "Catania"
   2) "56.4413"
   3) 1) "15.087267458438873"
      2) "37.50266842333162"
```
### georadiusbymember
```
GEORADIUSBYMEMBER key member radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [ASC|DESC] [COUNT count]
```
这个命令和 GEORADIUS 命令一样， 都可以找出位于指定范围内的元素， 但是 GEORADIUSBYMEMBER 的中心点是由给定的位置元素决定的， 而不是像 GEORADIUS 那样， 使用输入的经度和纬度来决定中心点。
```
redis> GEOADD Sicily 13.583333 37.316667 "Agrigento"
(integer) 1

redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

redis> GEORADIUSBYMEMBER Sicily Agrigento 100 km
1) "Agrigento"
2) "Palermo"
```
## 计算过程
首先，分别针对经度和纬度，求取当前区间（对于纬度而言，开始的区间就是[-90, 90], 对于经度而言，开始区间就是[-180, 180]）的平均值，将当前区间分为两个区间。然后用用户的经、纬度和区间平均值进行比较，用户经、纬度必然落在两个区间中的一个，如果大于平均值，那么取 1，如果小于平均值，那么取 0。
继续求取当前区间的平均值，进一步将当前区间分为两个区间。如此不断重复，可以在经度和纬度方向上，得到两个二进制数。这个二进制数越长，其所在的区间越小，精度越高。

以(123.15488794512, 39.6584212421)为例计算geohash：

1. 第一步，经纬度分别计算出一个二进制数，通过二分法不断查找最小区间。
以经度123.15488794512为例，计算过程如下：
```
精度|左边界|平均值|右边界|结果
1|-180|0|180|1|
2|0|90|180|1|
3|90|135|180|0|
4|90|112.5|135|1
5|112.5|123.75|135|0
6|112.5|118.125|123.75|1
7|118.125|120.9375|123.75|1
8|120.9375|122.3438|123.75|1
...
20|123.1547|123.155|123.1554|0
```

#经度转换结果
11010111100100111010
#维度转换结果
10111000011001110011

2. 两个二进制组合，经度占偶数位，纬度占奇数位
11100 11101 10101 01001 01100 00111 11100 01101
 
3. 每5位一组，进行base32编码，得到geohash字符串
```
8个10进制数
28 29 21 9 12 7 28 13
wxp9d7we
```

## 应用
交友软件，获取用户当前位置附近的所有用户
### 实现方式
- 使用 Hash 表存储的 GeoHash 算法，经度采用 13bit 编码，纬度采用 12bit 编码，即最后的 GeoHash 编码 5 个字符。
- 采用 Hash 表存储。Hash 表的 key 是 GeoHash 编码，value 是一个 List，其中包含了所有相同 GeoHash 编码的用户 ID
	- 用户打开App的时候，更新数据：删除旧的位置hash key的用户，添加到新的位置hash key。所以应用还需要记录键值对：用户->位置
- 查找邻近好友的时候，先计算用户当前位置的 GeoHash 值（5 个字符），然后从 Hash 表中读取该 Hash 值对应的所有用户，即在同一个网格内的用户，进行匹配，将满足匹配条件的对象返回给用户
- 如果一个网格内匹配的对象数量不足，计算周围 8 个网格的 GeoHash 值，读取这些 Hash 值对应的用户列表，继续匹配
