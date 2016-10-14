+++
title = "redis string operations"
date = "2016-10-14T16:20:41+08:00"
tags = ["redis","string","rate limit"]
categories = ["programming","cache"]
menu = ""
banner = "banners/redis.jpg"
+++
Notes about the commands of Redis String operations.Especially some scenarios that would be much useful.
<!--more-->


## Redis 

## String Operation.

## SET
 - 用法: `SET key value [EX seconds] [PX milliseconds] [NX|XX]`,整个set系列里功能最完整的一个命令,`setnx`,`setex`能完成的事情，都可以用`set`完成.
 - 对于某个原本带有生存时间（TTL）的键来说， 当 SET 命令成功在这个键上执行时， 这个键原有的 TTL 将被清除。
 - 即: **set是一个覆盖式写.如果写发生，则对于value和ttl都是覆盖式的**.
 - 解释:
 
 	|key word|说明|
 	|---|---|
 	| EX second|设置键的过期时间为 second 秒。 SET key value EX second 效果等同于 SETEX key second value.|
 	|PX millisecond|设置键的过期时间为 millisecond 毫秒。 SET key value PX millisecond 效果等同于 PSETEX key millisecond value.|
 	| NX|只在键不存在时，才对键进行设置操作。 SET key value NX 效果等同于 SETNX key value .|
 	| XX|只在键已经存在时，才对键进行设置操作.|



## GETBIT/SETBIT/BITCOUNT
 - `Redis`针对位做操作的支持.
 - 用法 : `setbit key offset 1`,设置某个key的某个offset为1(当然也可以设置为0,不过因为默认为0.为0的时候不去管理就好了。不过如果是设置为1设置错了或者要重新从1变为0,就可以设置).`getbit key offset`,获取key的offset位置的bit值是0还是1.`bitcount key [start] [end]`,给出key在指定区间(不指定区间则表示所有),所有bit值为1的个数.
 - 模式:
 	1. **Bitmap** 对于一些特定类型的计算非常有效。假设现在我们希望记录自己网站上的用户的**上线频率**，比如说，计算用户 A 上线了多少天，用户 B 上线了多少天，诸如此类，以此作为数据，从而决定让哪些用户参加 beta 测试等活动 —— 这个模式可以使用 SETBIT 和 BITCOUNT 来实现。
 	2. 以此想到了一个场景,比如每个用户每天只能操作一次的情况.每个月最多31号.维护一个31bit长的redis key即可.每天如果操作了就是1.没操作就是0.每个月做一次数据的删除(keys+del,此时keys跟del非原子,要保证原子性还是得用`redis+lua`),不然会可能有人在删除的间隙操作了,这时候由于整个删除机制被干掉了.BUT更好的方式是每天重置之前天的数据就好了.这样所有数据重置的时候没有写入,也就不需要考虑资源征用的问题(**通过避免并发才是最好的性能**).但是这里最重要的是,一定周期之后要获得用户每天的是否操作的数据,如果不获得则仅仅就用（setnx来做就行,这样每天的数据是一个key.这个key第二次就会由于存在而setnx会返回失败).
 - BIT操作的性能
 	1. 前面的上线次数统计例子，即使运行 10 年，占用的空间也只是每个用户 10*365 比特位(bit)，也即是每个用户 456 字节。对于这种大小的数据来说， BITCOUNT 的处理速度就像 GET 和 INCR 这种 O(1) 复杂度的操作一样快。

	2. 如果你的 bitmap 数据非常大，那么可以考虑使用以下两种方法：

	将一个大的 bitmap 分散到不同的 key 中，作为小的 bitmap 来处理。使用 Lua 脚本可以很方便地完成这一工作。
	使用 BITCOUNT 的 start 和 end 参数，每次只对所需的部分位进行计算，将位的累积工作(accumulating)放到客户端进行，并且对结果进行缓存 (caching).
 
## GETSET
 - 用法 `getset key value`,给key设置一个新值,并返回key的旧值(没有旧值返回nil).
 - `set`命令成功返回的是OK.
 - 模式:
	1. GETSET 可以和 INCR 组合使用，实现一个有原子性(atomic)复位操作的计数器(counter)。
	2. 举例来说，每次当某个事件发生时，进程可能对一个名为 mycount 的 key 调用 INCR 操作，通常我们还要在一个原子时间内同时完成获得计数器的值和将计数器值复位为 0 两个操作.实际上在实际过程中,需要在`incr`之后还要对`incr`的操作结果做判断,然后再复位为0.这时候`incr`和`getset`之间已经有时间消耗，不再是原子的.这在高并发场景如果不能接受.这时候应该考虑`redis+lua`的实现.
	
 		```
 		redis> INCR mycount
		(integer) 11

		redis> GETSET mycount 0  # 一个原子内完成 GET mycount 和 SET mycount 0 操作
		"11"

		redis> GET mycount       # 计数器被重置
		"0"
 	```

## MGET
 - 用法 `mget key1 [key2] [key3...]`,一次返回多个key的值,返回值的顺序和key值的顺序一致.
 - 如果某个key不存在,该key对应返回的值是nil,该命令永不失败.
## MSET
 - 用法: `MSET key value [key value ...]`,一次设置多个key的值.
 - 如果某个给定 key 已经存在，那么 **MSET 会用新值覆盖原来的旧值**，如果这不是你所希望的效果，请考虑使用 **MSETNX 命令：它只会在所有给定 key 都不存在的情况下进行设置操作s**。
 - **MSET是一个原子性(atomic)操作**，所有给定 key 都会在同一时间内被设置，某些给定 key 被更新而另一些给定 key 没有改变的情况，不可能发生.
 - **命令总是返回OK,因为mset不会失败**.

## MSETNX
- 用法同mset,但是不同的是仅仅在所有key不存在的时候才设置值.如果有任何一个key已经存在,整个msetnx是不执行的.
- **MSETNX是一个原子性(atomic)操作**


## SETEX
 - 用法: `setex key seconds value`,含义: set expire.
 - 最终结果等价于

 	```
 	set key value
	expire key seconds
 	```
 - 与以上命令的区别:
 	1. setex是一个原子操作,关联key和value的同时设置了超时时间.但是一旦用set+expire组合来实现的时候,至少是两个命令.这时候中间至少有 `set返回+expire命令发送的时间间隔`,如果set和expire在发送到redis之间有别的操作,间隔时间就会更长.

## PSETEX 
 - 用法: `psetex key milliseconds value`.
 - 这个命令和 SETEX 命令相似，但**它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位**。



## SETNX
 - 用法: `setnx key value`,含义:set if not exist.
 - 注意: **setnx不支持设置超时时间**，仅仅能给key设置一个不带超时的值.
 - 模式: 可以用在只能操作一次的场景.比如 每天只能签到一次.这时候第二次来接口重复性验证就可以redis直接完成(速度更快).每次操作如果不是永远有效.往往还需要设置一个expire的时间.这时候如果用`setnx+expire`不能保证原子性.则需要用`set+NX`命令来实现
## SETRANGE
 - 用法:`setrange key offset value`.
 - 最简总结:**override从offset开始的数据,如果offset之前无内容用"\x00"填充,key不存在,相当于整个内容最初就是空的**.
 - SETRANGE 命令会确保字符串足够长以便将 value 设置在指定的偏移量上，如果给定 key 原来储存的字符串长度比偏移量小(比如字符串只有 5 个字符长，但你设置的 offset 是 10 )，那么原字符和偏移量之间的空白将用零字节(zerobytes, "\x00" )来填充。

## GETRANGE
- 用法: `getrange key start end`,返回key对应的value的从start到end的内容(包含start和end).
- 如果key不存在的时候，返回的是空字符串.

## TTL 
 - 用法: `ttl key`,得到key的超时时间.返回的单位为秒.
 - 不同返回值的含义
 	
 	```
 	 -1 -- 如果key没有超时时间,则返回-1.
 	 -2 -- 如果key不存在,返回-2.
 	 否则以秒为大内，返回key的超时剩余时间.
 	```
 
## INCR/DECR/INCRBY/DECRBY
- 用法 : `incr/decr key`对key做加/减1.`incrby/decrby key incrementVal/decrementVal`
- 如果 key 不存在，那么 key 的值会先被初始化为 0,然后再执行操作.

### STRLEN
- 用法： `strlen key`,返回一个string value的长度. 
