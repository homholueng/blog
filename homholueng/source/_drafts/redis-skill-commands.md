---
title: Redis 使用技巧 - 命令
tags:
  - Redis
categories:
  - Redis
---

## APPEND

- 最低版本：2.0.0
- 时间复杂图：O(1)

```text
APPEND key value
```

如果 key 已经存在并且是一个字符串，那么该命令会将 value 拼接到字符串的末尾；否则就创建一个空字符串并将 value 拼接到字符串的末尾。

```shell
redis> EXISTS mykey
(integer) 0
redis> APPEND mykey "Hello"
(integer) 5
redis> APPEND mykey " World"
(integer) 11
redis> GET mykey
"Hello World"
```

### 时间序列

使用 `APEED` 命令能够用于创建一个大小固定的列表的表示，我们一般称之为时间序列。

而要访问序列中的每一个元素也很简单，在元素大小已知的情况下：

- `STRLEN`：用于获取列表中元素的个数。
- `GETRANGE`：用于获取特定位置的元素。
- `SETRANGE`：用于设置特定位置的元素。

```shell
(integer) 4
redis> APPEND ts "0035"
(integer) 8
redis> GETRANGE ts 0 3
"0043"
redis> GETRANGE ts 4 7
"0035"
```

## BGSAVE

- 最低版本：1.0.0

```text
BGSAVE
```

在后台将 DB 持久化，该命令会立刻返回，但是 Redis 会 fork 出子进程来完成 DB 持久的操作，而父进程能够继续接收和处理客户端的请求。其他客户端能够使用 `LASTSAVE` 来确认操作是否成功完成了。

## BITCOUNT

- 最低版本：2.6.0
- 时间复杂图：O(n)

```text
BITCOUNT key [start end]
```

计算字符串所包含的比特数，通过 start 和 end 能够计算特定的子串，另外，start 和 end 可以是负数，`-1` 表示倒数第一个字符，以此类推。

计算不存在的 key 则会返回 `0`。

```shell
redis> SET mykey "foobar"
"OK"
redis> BITCOUNT mykey
(integer) 26
redis> BITCOUNT mykey 0 0
(integer) 4
redis> BITCOUNT mykey 1 1
(integer) 6
redis> BITCOUNT mykey -1 -1
(integer) 4
```

### bitmaps

使用 `BITCOUNT` 和 `SETBIT` 能够轻松的在 Redis 中进行 bitmap 的操作，而且由于