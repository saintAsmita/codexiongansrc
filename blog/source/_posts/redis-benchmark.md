---
title: redis-benchmark
date: 2019-06-24 20:24:23
tags: redis
---

本文将简单介绍`Redis`性能测试工具`redis-benchmark.exe`，并且对key的长度对读写性能的影响做简单的测试。
<!--more-->

## 工具介绍

最近开发遇到一个加锁问题，提取到的key可能比较长，于是好奇了下key的长度对读写性能的影响，进而发现了Redis自带的性能测试工具`redis-benchmark.exe`。
此工具在你的`Redis安装目录`下。

1、如下命令，将会执行默认的操作，100_000次。具体是啥操作，要看默认的是什么了。
```shell
redis-benchmark -q -n 100000
```
2、如果你只想执行部分命令的测试，那么如下，对`set`命令，随机生成10_000个key，执行1000次。
```shell
redis-benchmark -t set -r 10000 -n 1000
```
结果：
![](https://res.cloudinary.com/saintasmita/image/upload/v1561381813/codexiongan/10_000keys_1000reqs.png)

如果想指定多个命令同时测试，则
```shell
redis-benchmark -t set,lpush -n 100000 -q
```

## 关于key长度的测试

### 网上的一种答案
他说需要修改下redis-benchmark的源码，从而指定key测试，我没有实际操作过，就粘贴下他的测试结果。[Stackoverflow上的好回答](https://stackoverflow.com/questions/6320739/does-name-length-impact-performance-in-redis)

这位作者测试了3个不同长度的key的get。可以发现，这些key的长度，对读性能没什么影响。尤其是前两种，算不得长key。至于第三种key，需要在意的是存储空间。

key = foo，GET，3次
```
59880.24 requests per second
58139.53 requests per second
58479.53 requests per second
```
key = set-allBooksBelongToUser:1234567890，GET，3次
```
60240.96 requests per second
60606.06 requests per second
58479.53 requests per second
```

key = ipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumloreipsumlorem:1234567890，GET，3次
```
58479.53 requests per second
58139.53 requests per second
56179.77 requests per second
```

### 测试Lua脚本

当然，我可以通过执行lua脚本，变相的测试指定key的性能。本机测试，不涉及到网络传输。
测试机器：一个普通的Windows机器。

```shell
redis-benchmark -n 100000 -q script load "redis.call('set','foo','bar')"
```
SET
```
5341.88 requests per second
4795.70 requests per second
5115.61 requests per second
```
GET
```
5057.40 requests per second
4648.78 requests per second
4727.01 requests per second
```

```shell
redis-benchmark -n 100000 -q script load "redis.call('set','foo123456789_123456789_123456789_123456789_123456789_123456789','bar')"
```
SET
```
5078.20 requests per second
4546.69 requests per second
5270.09 requests per second
```
GET
```
4862.87 requests per second
5005.00 requests per second
4804.23 requests per second
```

```shell
redis-benchmark -n 100000 -q script load "redis.call('set','foo123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789_123456789','bar')"
```

SET
```
5104.12 requests per second
5075.63 requests per second
5091.91 requests per second
```
GET
```
4974.38 requests per second
5013.54 requests per second
4715.65 requests per second
```

我们可以看到，key的长度对set,get也没有什么影响。当然，如果key的长度实在太长了，就会有影响，一般也碰不到那么长的key。几百个字符的key还是对性能没有影响的。
当然，不建议用这么长的key，影响阅读。

