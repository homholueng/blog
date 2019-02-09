---
title: Redis 各部署场景下 python client 的使用
date: 2018-11-03 12:05:37
categories:
- Python Daily
tags:
- Redis
- Python
---



## 1. Single Node



### 1.1. 介绍

{% asset_img single_instance.png [单实例] %}


这是我们比较常用的一种部署方式，该模式下只包含一个 Redis 实例，该实例存储所有的数据，client 的所有读写操作都在该实例上完成。



### 1.2. 如何使用 client 



在该模式下，我们只需要连接该 Redis 实例并对其进行相应的操作即可：



```python
import redis

host = ''
port = 6379
db = 0

r = redis.StrictRedis(host=host, port=port, db=db)
r.set('foo', 'bar') # return True
r.get('foo') # return 'bar'
```



## 2. Replication



### 2.1. 介绍

{% asset_img master_slave.png [主从备份] %}


即使我们开启了 Redis 的本地持久功能，也无法保证绝对的安全，因为数据仍有可能因为机器的物理损坏而丢失，所以，我们需要在系统中保存 Redis 数据的多个备份。Redis 的**复制（Replication ）**功能能够解决这个问题。



在该部署模式下，Redis 其实是以集群的方式来运行的，集群中会包含多个 Redis 实例，这些实例中只有一个是主（master）节点，其余的都是从（slave）节点，从节点会作为主节点的复制（replication），定期的将主节点中的数据复制过来进行备份。



在该模式下，需要注意下面这些问题：



- Redis 使用的是异步复制策略。
- 不仅主节点能够有从节点，从节点也能拥有自己的从节点。
- 复制功能既可以单纯的作为数据冗余用，也能够通过让多个从服务器处理只读命令来提升 Redis 服务的扩展性（如将繁重的 SORT 命令交给从节点去执行）。
- 该模式下一般会将从节点配置为只读节点，当然你也可以将从节点配置为可写节点，但因为从节点会定期从主节点中同步数据，所以写到从节点中的数据会在数据同步完成后丢失。


### 2.2. 如何使用 client



由于该模式下一般只有主节点能够进行写操作，所以需要在应用中维护主节点和从节点的地址列表，按需操作：



```python
import redis

master_host = ''
master_port = 6379
master_db = 0

slave_1_host = ''
slave_1_port = 6379
slave_1_db = 0


mr = redis.StrictRedis(host=master_host, port=master_port, db=master_db)
mr.set('foo', 'bar') # return True
mr.get('foo') # return 'bar'

sr = redis.StrictRedis(host=slave_1_host, port=slave_1_port, db=slave_1_db)
sr.get('foo') # return 'bar'
sr.set('foo', 'from slave 1') # will raise ReadOnlyError
```





## 3. Replication With Sentinel



### 3.1. 介绍

{% asset_img sentinel.png [Redis Sentinel] %}


在 Replication 的部署方式下，由于从节点对主节点的数据做了备份，那么在主节点宕机时，从节点完全有能力晋升为主节点，以保证 Redis 的可用性，但是，随之而来的问题是：**一旦主节点发生了变化，那么用户端需要修改主节点的地址，同时还要告诉集群中的其他从节点去备份新的主节点的数据。**



而 **Redis Sentinel（哨兵）** 为我们解决了这个问题，Redis Sentinel 支持**自动故障转移（Automatic failover)**：当集群中的主节点宕机时，Sentinel 会在其从节点中从新选举出一个主节点，并重新配置其他的从节点，使得这些从节点能够从新的主节点中复制数据，同时，Sentinel 还会通知后续要使用 Redis 的 client 新的主节点的地址。与此同时，为了保证 Sentinel 不会成为系统中的单点，你还可以在一个架构中运行多个 Sentinel 进程。



除了自动故障转移，Sentinel 还会提供以下功能：



- **监控（Monitoring）**：Sentinel 会定时检测主节点和从节点的运行状态。
- **通知（Notification）**：Sentinel 能够在其检测的 Redis 实例出现错误时通过 API 来通知系统管理员或是其他程序。
- **配置提供（Configuration provider）**：Sentinel 能够为 client 提供服务发现的功能，client 能够从 Sentinel 处获取某个服务当前所对应的主节点的地址，并且 Sentinel 会在自动故障转移发生后返回新的主节点的地址。



需要注意的是，因为 Redis 使用的是异步复制的策略，**所以 Redis 集群无法保证数据的强一致性**。考虑以下场景：



假设一个 Redis 集群中包含三个主节点 A，B，C，且每个主节点都拥有一个从节点 A1，B1，C1。



1. 你向 master B 中写入了一条数据。
2. master B 响应了你并告诉你已经写成功了。
3. master B 将这次写操作广播到其所有的从节点 B1 B2 B3....



若 master B 在第二步完成后和第三步开始前宕机了，那么其从节点 B1 就会被推举为主节点，但是你对 master B 的这次写入操作就会永远丢失。



至于 Redis 为什么不采用同步复制的策略，主要原因还是处于性能考虑。



### 3.2. 如何使用 client



由于 Sentinel 提供了服务发现的功能，我们能够向 Sentinel 获取当前 Redis 集群中某个服务的主节点和从节点的信息，并进行相应的操作：



```python
import redis
from redis.sentinel import Sentinel

sentinel_host = ''
sentinel_port = 16379

rs = Sentinel([(sentinel_host, sentinel_port), ], password='') # 在此传入 Redis 配置的密码
mr = rs.master_for(service_name='myservice') # 获取 myservice 服务的 master 节点，服务名在 Redis 集群配置中设置；该方法默认返回 StrictRedis 实例
mr.set('foo', 'bar') # return True
mr.get('foo') # return 'bar'

sr = rs.slave_for(service_name='myservice')
mr.get('foo') # return 'bar'

```





## 4. Data Sharding 



### 4.1. 介绍

{% asset_img cluster.png [Sharding] %}


在上面介绍的几种模式中，都只有一个 Redis 主节点，而应用的写操作又都是由主节点来完成的，这就会导致**主节点的写能力和存储能力受到单机性能的限制**。



而 Redis 的**数据分片（data sharding）**功能能够为我们解决这个问题，当我们开启的数据分片后，集群中会包含多个主节点，我们写入 Redis 中的数据会被分配并存储到不同的主节点中，一个 Redis 集群会包含 16384 个**哈希槽（hash slot）**，集群使用公式 `CRC16(key) % 16384` 来计算键 `key` 属于哪个槽。



假设一个 Redis 集群中包含三个节点：A，B，C，那么：



- 节点 A 会管理 0 - 5500 号哈希槽
- 节点 B 会管理 5501 - 11000 号哈希槽
- 节点 C 会管理 11001 - 16383 号哈希槽



需要注意的是，**Redis 集群中的节点并不能代理命令请求**，若我们向集群中的某个实例请求读取或写入一个不属于该实例所管理的键，那么其会返回 `-MOVED` 或 `-ASK` 重定向错误，同时，负责管理该键的节点地址和端口的信息会一并返回。



Redis 官方的建议是客户端应该记录每个节点所管理的槽的信息，当集群处于稳定状态时， 所有客户端最终都会保存有一个哈希槽至节点的映射记录， 使得集群非常高效。



### 4.2. 如何使用 client



由于 Redis 集群中的节点不能为我们代理命令请求，而 Python 中的 `redis` 包并不会处理，这就意味着我们需要在应用中自己维护节点所管理的哈希槽信息。但是好在有第三方的包能够处理这个问题： `redis-py-cluster` 会帮我们管理节点信息，我们只需要像使用普通的 client 一样使用它就好。



**注意：在这种情况下我们不需要再通过连接 sentinel 来获取集群中的节点信息, `redis-py-cluster` 会自动帮我们发现集群中的节点。**



```python
import redis-py-cluster

startup_nodes = [{"host": "127.0.0.1", "port": "7000"}] # 集群中的任一非 sentinel 节点

rc = StrictRedisCluster(startup_nodes=startup_nodes, password='pwd')

rc.set("foo", "bar") # True
rc.get("foo") # 'bar'
```





## 5.总结



总的来说，Redis 的部署方式及其特点如下：



| 部署方式                                  | 数据备份 | 数据分片 | 高可用 | py-cli module    | 备注                            |
| ----------------------------------------- | :------- | -------- | ------ | ---------------- | ------------------------------- |
| Single Node                               | ×        | ×        | ×      | redis            | 可开启本地数据持久              |
| Replication                               | √        | ×        | ×      | redis            |                                 |
| Replication with Sentinel                 | √        | ×        | √      | redis            |                                 |
| Sharding (with Replication)               | √        | √        | ×      | redis-py-cluster | 通常都会与 Replication 一起使用 |
| Sharding with Sentinel (with Replication) | √        | √        | √      | redis-py-cluster | 通常都会与 Replication 一起使用 |