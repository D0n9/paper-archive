# 【实战企业级Java二】渐进式理解Redis分布式锁-阿里云开发者社区
渐进式理解Redis分布式锁
--------------

> 并发场景下，由于修改和保存数据的过程不是原子性的，部分操作可能会丢失，在单服务中我们常用本地锁来避免并发带来的问题。但是本地锁无法在多服务器之间生效。

#### 1\. 分布式锁需要满足的条件

*   互斥性：任意时刻，只能有一个客户端获取锁。
*   同一性：锁只能被持有该锁的客户端删除。
*   可重入性：持有锁的客户端可继续对该锁加锁，实现锁的续租。
*   容错性：持有锁的客户端下线，到期释放锁，防止死锁。

#### 2\. 如何实现Redis分布式锁？

##### 2.1 如何使用Redis加锁❓

> 最直白的做法：SETNX
> 
> `SETNX` is short for "**SET** if **N**ot e**X**ists"，即设置KEY如果不存在的话，value我们可以暂定设置1。

```null
SETNX lockName 1
```

返回1说明key不存在设置成功，即获取到了锁，返回0则加锁失败。

##### 2.2 加锁就需要解锁，使用Redis解锁❗️

> 删除命令：DEL

```null
DEL lockName
```

删除了该key，此时其他线程就可以通过SETNX获取锁了。

##### 2.3 为了保证容错性，需要设置锁的超时时间❗️

> 设置key的过期时间：EXPIRE

```null
EXPIRE lockName 20
```

为key设置一个超时时间，以保证即使锁没有被显示的释放时，在到达过期时间后也能自动释放锁，防止死锁的产生。

##### 2.4 即第一版的分布式锁伪代码为：⁉️

```null
if（setnx（key，1） == 1）{
    expire（key，30）
    try {
        work....
    } finally {
        del（key）
    }
}
```

##### 2.5 问题1：加锁和设置过期时间是非原子操作❗️

在极端情况下，当线程执行完`SETNX`还未执行`EXPIRE`时服务挂掉。

此时该锁既不会被显示的解锁，也不会自动过期，其他线程再也无法获取到该锁了，game over。

##### 2.6 如何解决死锁的问题呢❓

> SET命令加锁

```null
SET lockName 1 EX 30
```

`SETNX`命令是不支持传入超时时间的，不过幸好Redis2.6.12以后为SET指令增加了可选参数EX、PX属性，这样加锁和设置超时时间就是原子操作了。

##### 2.7 问题2：锁到期，任务未完成❗️

回忆一下我们实现的锁机制，如果锁到期了任务未完成将产生两个严重问题。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/e6fb81a0-7baf-47fe-b3f8-73e39d752d8f.png?raw=true)

1.  将其他线程的锁释放（不满足同一性）。
2.  其他线程提前获取到了锁，即本不应该同时执行的任务同事执行（不满足互斥性）。

##### 2.8 如何解决释放其他线程锁的问题❓

解决这个问题，我们只需要在删除之前验证key对应的value是不是自己的线程。

我们可以把线程ID作为key对应的value，在删除之前验证一下锁是不是自己的锁。

**伪代码：** 

```null
加锁：
String threadId = Thread.currentThread().getId()
set（key，threadId ，30，EX）
解锁：
if（threadId .equals(redisClient.get(key))）{
    del(key)
}
```

这里，判断锁和删除锁是两个独立操作，不是原子操作。

我们可以使用lua脚本来实现：

```null
String luaScript = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
```

这样，判断和删除过程就是原子操作了。

##### 2.9 如果解决两个线程同时获取到锁的问题❓

上面我们解决了释放非自己锁的问题，但是AB两个线程同时执行任务也是不完美的。

我们可以让获得锁的线程开启一个守护线程，用来给快到期的锁续期。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/1251346b-a88a-4ac8-893f-76d9bc68f52a.png?raw=true)

#### 3\. 下一篇Redisson分布式锁

Redis分布式锁在生产中使用自然不需要我们自己去实现每一个细节，Redis分布式锁在java中的解决方案官方推荐就是Redisson

[【Distributed Locks with Redis】](https://redis.io/docs/reference/patterns/distributed-locks/)

> 🏄🏻作者简介：CSDN博客专家，华为云云享专家，阿里云专家博主，疯狂coding的普通码农一枚
> 
> 🚴🏻‍♂️个人主页：[莫逸风](https://moyifeng.blog.csdn.net/)、[【企业级Java开发实战专栏】](http://t.csdn.cn/pNe3z)  
> 
> 🇨🇳喜欢文章欢迎大家👍🏻点赞🙏🏻关注⭐️收藏📄评论↗️转发
> 
> 🏋️‍♂️公众号：莫逸风
> 
> 📱微信：moyifengxue