---
title: Redis 使用技巧 - Pipelining
tags:
  - Redis
categories:
  - Redis
date: 2019-08-11 20:25:35
---


## 请求/相应以及 RTT

Redis 是基于 CS 模型的 TCP 服务器，所以我们每次操作都会遵循如下步骤完成：

- 客户端将查询请求发送给服务器，服务器中套接字中读取客户端的请求，客户端会等待服务器的相应，而这个过程通常是阻塞的。
- 服务器处理完成后，将响应返回给客户端。

假设客户端需要向服务器发送四条请求，那么请求和响应顺序应该是这样的：

- Client: INCR X
- Server: 1
- Client: INCR X
- Server: 2
- Client: INCR X
- Server: 3
- Client: INCR X
- Server: 4

从客户端发起请求的那一刻开始计算，直到客户端收到来自服务器的响应，这中间消耗的时间我们称其为 RTT（Round Trip Time）。当客户端需要在一行（例如往同一个列表中添加多个元素，或是操作同一个数据库中的多个 key 值）上进行多次操作时，RTT 会对性能造成不小的影响。假设我们拥有一台性能强大的服务器，这台服务器能够在一秒内处理 100K 个请求，那么在 RTT 为 250 毫秒的情况下，这个性能强大的服务器一秒内也只能处理 4 个请求。

## Redis Pipelining

Redis 提供的 Pipelining 特性允许我们将多个请求一次性发送给服务器，然后再将所有的响应一次性读取回来。

```shell
$ (printf "PING\r\nPING\r\nPING\r\n"; sleep 1) | nc localhost 6379
+PONG
+PONG
+PONG
```

以上小节的例子来看，使用的 Pipelining 之后，请求和相应的顺序应该是这样的(**注意，多条命令是合并在一个请求中发送出去的，处理结果也是在一个响应中返回的**)：

- Client: INCR X
- Client: INCR X
- Client: INCR X
- Client: INCR X
- Server: 1
- Server: 2
- Server: 3
- Server: 4

**如果客户端使用了 Pipelining 这个特性，那么服务器就必须将响应的内容加入到队列中，直到所有操作都处理完成后再一次性返回。当然，这个队列是在内存中的，如果你需要使用 Pipelining 来执行大量的操作，最好是将这些操作进行分批 Pipelining，避免服务器在存储响应内容时消耗过多的内存。**

## 其实不仅仅是 RTT 的问题

Pipelining 这个特性其实不仅仅是解决了 RTT 开销的问题，还防止了过多 socket I/O 带来的消耗，因为多条操作指令能够在一次 `read()` 系统调用中读取出来，多条响应内容也能够在一次 `write()` 系统调用中完成写入。

从官方文档给出的资料可以看出，随着 Pipelining 指令数量的增长，服务器的 qps 也成近似指数关系的增长，最后稳定在大概十倍的位置：

{% asset_img pipeline_iops.png [pipeline_iops] %}

## 实际测试

下面是一段使用 Python 代码进行的测试：

```python
import redis
import time

TIMES = 10000

def bench(func, desc):
    start = time.time()
    func()
    print('{desc} {cost} milliseconds'.format(desc=desc, cost=time.time() - start)) 

def without_pipeling():
    r = redis.Redis(host='localhost', port=6379)
    for _ in range(TIMES):
        r.ping()

def with_pipelining():
    r = redis.Redis(host='localhost', port=6379)
    with r.pipeline(transaction=False) as p:
        for _ in range(TIMES):
            p.ping()

if __name__ == '__main__':
    bench(without_pipeling, 'without_pipeling')
    bench(with_pipelining, 'with_pipelining')
```

测试结果如下：

```text
without_pipeling 1.24247217178 milliseconds
with_pipelining 0.0302491188049 milliseconds
```