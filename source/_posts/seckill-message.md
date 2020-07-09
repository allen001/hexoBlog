---
title: 简述秒杀系统
date: 2020-05-22 17:01:48
toc: true
comments: true
tags:
- 秒杀
- docker
- mysql
---

## 系统特征
1: 时间极短
2: 瞬间请求量非常大

## 活动注意点
1: 不能超卖（直接经济损失，平台信誉受损）
2: 防黑产,黄牛 (阿里月饼门) , 机器的请求速度比人的手速快太多了
3: 瞬间爆发的高流量
  > * 典型的读多写少的场景（cache 缓存)
  > * 页面静态化 ,利用cdn服务器缓存前端文件
  > * 按钮置灰3秒. (利用风控规则过滤掉非法用户)
  > * 接口层可以做开关限流( 一旦抢购结束,则直接返回失败)
  > * 堆机器, 搭建集群. 利用nginx做负载均衡
  > * 热点隔离 增加资源  有限放流(熔断)  多次请求合并为一次

4: 尽量把请求拦截在上层
  > * Mysql单机读能力为5k, 写能力为3k
  > * redis单机读能力最高可达10w, 写能力能达到3-5w

5: 预热. 运营人员提前将数据写入redis
6: 异步去扣减库存 (消息队列)
  > * qps=> 每秒查询量
  > * tps=> 每秒处理的事务量
  > * pv=> page view 
  > * uv=> unique view
  > * ip=> 不重复的ip

## 系统业务图
![秒杀系统业务图](https://i.loli.net/2020/05/22/eQkhKmd5fwHztOl.png)

<!-- more -->

## 技术选项cache,nosql,db 
**缓存穿透, 缓存雪崩**
db: 必须具备ACID的特性,才能算得上数据库
1: redis (单线程,单核的内存数据集) 
> * 持久性: 持久化数据 AOF (append only file) ,RDB (redis database)
> *  一致性: 并不支持严格的一致性
> *  隔离性: 天然支持，单线程程序
> * 原子性：2.6版本以后的redis支持lua脚本来间接达到原子性
> ```lua
 watch;
 redis.set(a,1);
 redis.get(a);
 redis.set(b); 
 redis.set(a,3);
```

2: memcached

> * 只支持string
> * 不支持持久化,通过第三方的插件来间接持久化
> * 不支持集群方案, 是通过hash算法来实现集群
> * 多线程的，速度比redis快 (线程安全)

3: 本地缓存 ehcache, guava cache,spring cache
> * 速度最快,但是单机的

## JMeter测试样本
![](https://i.loli.net/2020/05/22/Y6IcksTKeD91Hg5.png)

样本: 请求总数
平均值: 平均响应时间(毫秒)
中位数: 大多数的响应时间
吞吐量 :tps

``` java
 	@GetMapping("/seckill/{id}")
    public String secKill(@PathVariable long id) {
        synchronized (lock) {
            Product product = service.select(id);
            int stock = product.getStock(); //过期的数据
            if (stock > 0) {
                if (service.secKill(id) > 0) {
                    return "秒杀成功";
                }
            }
            return "秒杀失败";
        }
    }
```
![](https://i.loli.net/2020/05/22/69COYGVhu8rRSAt.png)
``` sql
update product set stock=stock-1 where id =#{id} and stock>0
```
![](https://i.loli.net/2020/05/22/NAEI4j63i5l2WBD.png)

## innodb默认的行级锁
vip 秒杀qps: 200w/s

### 这样写有什么问题?
> * 所有的请求都会打到数据库, 数据库压力很大, 和我们的原则: 把请求拦截到上层相悖
> * 虽然说mysql执行很快，但是由于行级锁的存在,每个请求还是会等很久
> * qps:  query per second； 查询是不需要事务的
> * tps:  transaction,  insert, update, delete

## 实战Coding
### 使用docker安装redis
1: `docker pull redis`
2: 配置好redis的配置和data目录
```properties
# Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
daemonize no

# 指定Redis监听端口，默认端口为6379
port 6379

# 绑定的主机地址，不要绑定容器的本地127.0.0.1地址，因为这样就无法在容器外部访问
bind 0.0.0.0

# 持久化
appendonly yes
```
3: 启动redis
```bash
docker run -p 6379:6379 
 --name redis \
 -v Users/aya/docker/redis/conf/redis.conf:/etc/redis/redis.conf \
 -v Users/aya/docker/redis/data:/data \
 -d redis redis-server /etc/redis/redis.conf 
```
4: 测试redis
```bash
docker exec -it redis redis-cli
```
### 使用docker安装mysql
1: `docker pull mysql`
2: 配置mysql
```properties
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000

[client]
default-character-set=utf8

[mysql]
default-character-set=utf8
```
3: 启动mysql
```bash
docker run --restart=always 
 --privileged=true \
 -d \
 -v /Users/aya/docker/mysql/data/:/var/lib/mysql \
 -v /Users/aya/docker/mysql/conf.d:/etc/mysql/conf.d \
 -v /Users/aya/docker/mysql/conf/my.cnf:/etc/mysql/my.cnf \
 -p 3306:3306 \
 --name mysql \
 -e MYSQL_ROOT_PASSWORD=123456 \
 mysql
```
4: 测试
### 初步的减库存逻辑
```java
	@GetMapping("/seckill/{id}")
    public String secKill(@PathVariable long id) {
        Product product = service.select(id);
        int stock = product.getStock();
        if (stock > 0) {
            if (service.secKill(id) > 0) {
                return "秒杀成功";
            }
        }
        return "秒杀失败";
    }
```