---
layout:     post
title:      项目总结：在redis集群使用pipeline功能
subtitle:   redis
date:       2020-07-20
author:     BY
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - lc
---

### 目录

- redis中slot概念
- redis一条普通命令的执行路由
- redis 集群 smart client : jedis
- jedis一条pipeline命令的源码
- redis集群不支持pipeline的原因
- 解决方案
- 

**_本文是基于redis-5来讲解的, 提到的client均为jedis client。_**

本人在做项目中需要用到pipeline，发现redis集群并不能支持pipeline功能，

因此本文抛砖引玉，将讲述不支持的原因以及应对方案。

pipeline的使用场景：
1. 命令量大；
2. 命令结果前后无依赖；

## redis中slot概念

redis有固定16384个槽(slot)，数据就存储在槽中。
（课后习题:为什么固定16384，不能更多？）

redis的key通过CRC计算并与16383取模，可以计算出key所在的槽。
```java
int slot = getCRC16(key) & (16384 - 1)
```

而槽又根据均衡原则分配在不同的数据节点上，每个数据节点上维护一定数量的槽。

[![oPfnh.md.png](https://wx1.sbimg.cn/2020/08/06/oPfnh.md.png)](https://sbimg.cn/image/oPfnh)

## redis集群命令路由

首先我们要明确几个知识点：
1. redis集群有多个数据节点，每个数据节点除了维护自己所属的一批槽点外，还知道集群内其它节点负责的槽点；

2. redis服务端只负责返回自己槽点内的数据，不会跨节点代理请求；

3. redis客户端发送的命令，只落在某一个node，不能给多个node发送；

以上3点说明，当一个命令落在了不适合的数据节点上，是不会获得数据。

例如,一条命令get redis-test, 键redis-test的槽点是10000，所属节点是node-3。

如果这个命令发送给node-1的节点，由于node-1不对槽点是10000负责，所以是不会返回期望数据的。

所以一条错误节点命令的路由如下图

[![ook9K.md.png](https://wx2.sbimg.cn/2020/08/06/ook9K.md.png)](https://sbimg.cn/image/ook9K)

如果命令路由到的所在node包含所属的key所属的slot，那么就尝试执行这个命令；
否则就返回重定向ip port给客户端；
客户端需要重新发送请求；

## redis 集群 smart client

在上小节，redis集群主要是应对大量请求，如果如上所述，大量的命令都会发生重定向，发送两次请求，显然会存在网络I/O时间问题。

而使用redis的业务多是时间敏感的。

针对以上问题，诞生了smart_client, smart含义体现在，该client将每一条命令发送到正确的node上。

思考：为什么官方不支持server端代理分发请求请求到其它节点，反而要client来处理这个问题？
为了保证IO效率最大化。

接下来看看smart_client是如何工作的：
1. client创建了与每个节点的连接池；
2. client维护了集群中所有 (slot, node) 的映射关系；
3. client计算每次命令key的slot，根据slot获取目标node的链接；
4. client使用正确的连接发送命令；

图
[![o3M8Y.md.png](https://wx1.sbimg.cn/2020/08/06/o3M8Y.md.png)](https://sbimg.cn/image/o3M8Y)

由于集群存在扩容、缩容，slot会在不同node间发生迁移，所以还有第四点，

4. 当slot发生了迁移，client要重定向到新的node，已经更新本地的（slot, node) 映射；


jedisCluster一条普通命令的代码
```java
public Long sismember(final String key) {
    // client如果在发送多命令，不执行如下代码
    checkIsInMultiOrPipeline();
    // 执行命令
    client.sismember(key);
    // 获取结果
    return client.getIntegerReply() == 1;
  }

public Boolean sismember(final String key, final String member) {
return new JedisClusterCommand<Boolean>(connectionHandler, maxAttempts) {
  @Override
  public Boolean execute(Jedis connection) {
    return connection.sismember(key, member);
  }
}.run(key);
}

public T run(String key) {
    return runWithRetries(JedisClusterCRC16.getSlot(key), this.maxAttempts, false, false);
  }

// 根据key计算出所在的slot，然后从池子里获取这个slot对应的连接
private T runWithRetries(final int slot, int attempts, boolean tryRandomNode, boolean asking) {
    ...
    connection = connectionHandler.getConnectionFromSlot(slot);
    return execute(connection);
    ...
}

protected Connection sendCommand(final Command cmd, final byte[]... args) {
    try {
      connect();
      Protocol.sendCommand(outputStream, cmd, args);
      pipelinedCommands++;
      return this;
    } catch (JedisConnectionException ex) {
    }
  }

// 获取结果
public Long getIntegerReply() {
    // 在获取结果前，将buf里没有发送的命令发出去
    flush();
    pipelinedCommands--;
    return (Long) readProtocolWithCheckingBroken();
  }

//阻塞等待
protected Object readProtocolWithCheckingBroken() {
    try {
      return Protocol.read(inputStream);
    } catch (JedisConnectionException exc) {
    }
  }
```

## jedis一条pipeline命令的源码
在jedisCluster中，pipeline发送过程和上面的差不多，主要区别在于获取返回结果的过程不同。

pipeline具体的发送方式由客户端确定，redis server不参与。

有的客户端是一次发送多条命令，然后获取结果。有的客户端是连续发送多次命令，再获取结果。

jedis的发送流程如下：


源码：
```java
//创建pipeline对象时，已经指定了client
Pipeline pipeline = jedis.pipelined();
pipeline.sismember(k,v);
pipeline.incr(k);
pipeline.incr(k,v);
List<Object> result = pipeline.syncAndReturnAll();

//pipeline的命令发送同常规命令发送过程一样，可以看上一节的介绍
public Response<Boolean> sismember(final String key, final String member) {
    getClient(key).sismember(key, member);
    return getResponse(BuilderFactory.BOOLEAN);
  }

public List<Object> syncAndReturnAll() {
    if (getPipelinedResponseLength() > 0) {
      List<Object> unformatted = client.getAll();
      ...
      }
      return formatted;
    }
  }

public List<Object> getAll(int except) {
    List<Object> all = new ArrayList<Object>();
    flush();
    while (pipelinedCommands > except) {
      try {
        all.add(readProtocolWithCheckingBroken());
      } catch (JedisDataException e) {
        all.add(e);
      }
      pipelinedCommands--;
    }
    return all;
  }

protected Object readProtocolWithCheckingBroken() {
    try {
      return Protocol.read(inputStream);
    } catch (JedisConnectionException exc) {
      ...
    }
  }
```

[![o3CQO.md.png](https://wx2.sbimg.cn/2020/08/06/o3CQO.md.png)](https://sbimg.cn/image/o3CQO)

## redis集群不支持pipeline的原因
综上，我们可以总结如下：
1. pipeline将多个命令只发给一个node；但是这些命令的槽点可能对应其它node；
2. node间不负责代理转发请求；
3. 如果key不在接受到命令的node上，会没有正确结果； 

所以对于redis集群来说，简单的使用pipeline并不能达到我们的目的。

## 解决方案 

我们先回顾下，管道的目的是要解决什么问题？ 要解决网络I/O时间问题，就是减少网络通信次数。

结合smart_client的特性，保存有slot -> node的映射关系，方案如下：

1. 对一批命令里的key分别计算槽点，获得对应的node链接；

2. 对同属同一个node的key进行归档，得到每个node需要执行的命令列表；

3. 之后对每个node链接分别执行pipeline；

图[![o3ka7.md.png](https://wx2.sbimg.cn/2020/08/06/o3ka7.md.png)](https://sbimg.cn/image/o3ka7)

上面的方案还存在一定问题,在讨论smart_client说到，当slot发生迁移时，smart_client会根据返回的数据，自动重定向；

但是由于pipeline是批量返回结果的，不能针对每个命令发起重定向，

slot迁移后，需要更新slot -> node的映射，以及重新请求获取结果。


项目如何快速的识别集群变。
公司集群的ip port都是注册在zk上，我们只要监听着zk，就可以随时知道集群node的变化；

```java
keyValues.forEach( pair -> {
            JedisPool pool = jedisPipelineCluster.getConnectionHandler().getJedisPoolFromSlot(JedisClusterCRC16.getSlot(pair.getLeft()));

            if (!poolKeyValues.containsKey(pool)) {
                poolKeyValues.put(pool, new ArrayList());
            }
            poolKeyValues.get(pool).add(pair);
        });
```

## 解决方案2

使用hash_tag,这个方案是在写入key的时候，将多个key写入同一个node。

## 思考

1. 方案1，如何读写分离？
事实上，集群不建议读写分离操作

2. 为什么槽的数量固定是16384？

3. mget mset 是否也是这样的？
