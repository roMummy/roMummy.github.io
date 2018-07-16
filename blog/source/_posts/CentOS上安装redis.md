---
title: CentOS上安装redis
date: 2018-07-13 15:04:21
tags: Redis
categories: 数据库
---

### 目的
为了练习在springboot中使用redis数据库，本人比较懒 不想把redis数据库装在本地，正好有一个免费的服务器可以玩 。 于是在linux上安装redis，下面是安装redis遇到的坑。
### 开始
> 系统: CentOS Linux release 7.4.1708 (Core) </br>
> redis版本: redis-4.0.10 </br>
> redis官网: http://www.redis.cn/download.html

第一步当然是按照官方的步骤一步一步来着。
>$ wget http://download.redis.io/releases/redis-4.0.10.tar.gz </br>
>$ tar xzf redis-4.0.10.tar.gz <br>
>$ cd redis-4.0.10 <br>
>$ make


嗯嗯 一气呵成。没啥坑嘛 哈哈 想多了
> make[3]: gcc：命令未找到

恩恩额 好吧 安装gcc
> yum -y install gcc automake autoconf libtool make

.......................<br>
进过一段时间的等待 终于搞定gcc了！ 继续make
> zmalloc.h:50:31: 致命错误：jemalloc/jemalloc.h：没有那个文件或目录
 #include <jemalloc/jemalloc.h>

 WTF？？
 马丹 这是什么👻 二话不说，百度
 解决方法：
 > make MALLOC=libc

 这是啥？为啥要这么做？网上没找到答案 问了大佬才知道MALLOC=libc是 
 * 编译并指定动态内存分配功能使用libc提供的
 


 原来是内存分配的啊 为啥redis要用这个功能呢 来看看redis的原理：
 > Redis本质上是一个Key-Value类型的内存数据库，很像memcached，整个数据库统统加载在内存当中进行操作，定期通过异步操作把数据库数据flush到硬盘上进行保存。因为是纯内存操作，Redis的性能非常出色，每秒可以处理超过 10万次读写操作，是已知性能最快的Key-Value DB。
Redis的出色之处不仅仅是性能，Redis最大的魅力是支持保存多种数据结构，此外单个value的最大限制是1GB，不像 memcached只能保存1MB的数据，因此Redis可以用来实现很多有用的功能，比方说用他的List来做FIFO双向链表，实现一个轻量级的高性 能消息队列服务，用他的Set可以做高性能的tag系统等等。另外Redis也可以对存入的Key-Value设置expire时间，因此也可以被当作一 个功能加强版的memcached来用。
Redis的主要缺点是数据库容量受到物理内存的限制，不能用作海量数据的高性能读写，因此Redis适合的场景主要局限在较小数据量的高性能操作和运算上


恩恩额 原来redis经常和内存打交道啊 再来看看jemalloc干嘛用的：
> jemalloc是一个通用的malloc（3）实现，它强调了
碎片避免和可扩展的并发支持。jemalloc 
于2005年首次作为FreeBSD libc分配器使用，从那以后它已经
进入许多依赖于其可预测行为的应用程序。2010年，
jemalloc开发工作范围扩大到包括开发人员支持功能
，如堆分析和广泛的监控/调优钩子。现代jemalloc 
版本继续被集成到FreeBSD中，因此多功能性
仍然至关重要。正在进行的开发工作趋势是将jemalloc 
作为广泛的苛刻应用的最佳分配器之一，并且
消除/减轻对现实
世界应用产生实际影响的弱点。

个人理解的是
> redis主要是在内存上操作 而在内存上操作就不可避免的会产生内存碎片  为了优化内存碎片才选择的是jemalloc

好 安装jemalloc

#### 安装jemalloc

>#下载 <br>
 wget https://github.com/jemalloc/jemalloc/releases/download/4.2.1/jemalloc-4.2.1.tar.bz2  <br>
> #解压 <br>
>tar -xjvf jemalloc-4.2.1.tar.bz2 <br>
>#在jemalloc-4.2.1目录下预编译<br>
>./configure –prefix=/usr/local/jemalloc<br>
>#编译<br>
>make -j8 && make install

完成 但是不晓得为啥运行还是有那个错误 我是linux小白。。。
**********
#### 直接安装jemalloc依赖

github上的issues早就告诉了我们答案
* https://github.com/antirez/redis/issues/722

在redis目录下进入 
> cd deps

安装jemalloc依赖
> make hiredis lua jemalloc linenoise

> cd ../

重新编译
> make

成功

### 启动redis

> src/redis-server

加载配置 配置文件在刚刚安装redis文件夹下 
> src/redis-server ./redis.conf 

停止redis

> src/redis-cli shutdown

启动redis客户端

> src/redis-cli

如果被卡住了就使用kill直接杀死
> kill -9 进程号

使用下面这个命令查找进程号
> ps -ef|grep redis

### 配置redis


* 第一步：进入redis目录
> cd redis-4.0.10/

* 第二部：打开配置文件
> vim redis.conf

* 第三部：注释掉环内地址
> #bind 127.0.0.1

* 第四部：配置密码
> requirepass "123456"

* 第五步：保存退出
> wq

* 第六步：重启redis
> ps -ef|grep redis(查找进程号)<br>
> kill -9 xxxx(进程号)<br>
> src/redis-server ./redis.conf(使用配置启动)

OVER

### 测试连接redis（springboot）

配置信息：
```
# REDIS (RedisProperties)
# Redis数据库索引（默认为0）
spring.redis.database=0
# Redis服务器地址
spring.redis.host=localhost
# Redis服务器连接端口
spring.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.redis.password=
# 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-active=8
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-wait=-1
# 连接池中的最大空闲连接
spring.redis.pool.max-idle=8
# 连接池中的最小空闲连接
spring.redis.pool.min-idle=0
# 连接超时时间（毫秒）
spring.redis.timeout=0

```

编写测试类
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class RedisApplicationTests {

	@Autowired
	private StringRedisTemplate stringRedisTemplate;

	@Test
	public void contextLoads() {
		stringRedisTemplate.opsForValue().set("a","\n---------哈哈哈哈哈");
		System.out.print(stringRedisTemplate.opsForValue().get("a"));
	}

}

```