---
title: Flink 数据倾斜优化
date: 2022-12-01
categories: 大数据
tags: 
 - Flink
---

# 定义
当进行聚合运算时（GroupBy/KeyBy + Agg），如果聚合所使用的key存在热点，则会导致数据倾斜。如统计某日各个省份的车流量，则负责运算北京、上海等一线城市的count subtask节点则会成为热点，处理数据的压力会比较大。

# 危害

## 任务卡死
keyBy 或 rebalance 下游的算子，如果单个 subtask 存在热点并完全卡死，会把整个 Flink 任务卡死。看如下示例：
如下图所示，上游每个 Subtask 中会有 3 个 resultSubPartition，连接下游算子的 3 个 subtask。下游每个 subtask 会有 2 个 InputChannel，连接上游算子的 2 个 subtask。Local BufferPool为subtask中的ResultSubpartition/InputChannel所共用，在正常运行过程中如果没有反压，所有的 buffer pool 是用不完的。

一旦subtask B0变成热点，则会引起反压，依次产生如下问题：
1. Subtask B0 内的 A0 和 A1 两个 InputChannel 会被占满；Subtask B0 公共的 BufferPool 中可申请到的空间也被占满
2. Subtask A0 和 A1 的 B0 ResultSubPartition 被占满；Subtask A0 和 A1 公共的 BufferPool 中可申请到的空间也被占满
3. 如图2所示，Subtask A0 的主线程会从上游读取数据消费，按照数据的 KeyBy 规则，将数据发送到 B0、B1、B2 三个 ResultSubpartition 中；可以看到，如果 B0 这个ResultSubpartition占满了，且 B0 在公共的 Local BufferPool 中可申请到的空间也被占满。现在有一条数据被keyby后发往B0，但是现在 B0 这个ResultSubpartition 没有空间了，所以主线程就会卡在申请 buffer 上，直到可以再申请到 buffer

Subtask A0 的主线程被卡住，则不会往下游的任何subtask发送数据了，如图1所示，下游的Subtask B1和Subtask B2不再接收新数据。整个任务处于瘫痪状态

## Checkpoint时间变长
checkpoint barrier也是一种特殊的数据，如果整个任务中各个可用buffer变少，则checkpoint barrier的传输也会因为找不到可用buffer而降低速度；由于checkpoint barrier的对齐机制，会造成当前checkpoint的barrier迟迟无法对齐，进而超时。

## State变大
对于有两个以上输入管道的 Operator，存在checkpoint barrier对齐机制，接受到较快的输入管道的 barrier 后，它后面数据会被缓存起来但不处理，直到较慢的输入管道的 barrier 也到达，这些被缓存的数据会被放到state 里面，导致 checkpoint 变大。

# 解决办法
## 修改分区策略

### 目标
让不需要shuffle的两个算子间进行shuffle，打乱数据，从而避免数据倾斜

### 手段
在Flink任务提交后，经常可以看到web ui中的一些算子之间采用的分区策略是forward，在该分区策略下很可能会存在数据倾斜现象。如以下情况：
某kafka topic统计每个省份的车次，针对每个省份都有一个partition，共计36个partition，同时设有36个source算子，36个flatmap算子。由于source和flatmap满足one-to-one关系，且并行度相同，则Flink默认会采用forward这个分区策略来关联source和flatmap这两个算子。
Flink默认设置forward分区策略有两个条件：
1. 两个算子满足one-to-one关系
2. 两个算子并行度相同

此时，北京和上海对应的flatmap算子必然会出现热点数据，由于source到flatmap算子之间并不需要有特定的对应关系，因此可以采用不同的分区策略来将数据打乱，让不同省份的车流数据落到所有的flatmap算子，消除数据倾斜。

因此，我们只需要破坏forward分区策略的条件即可
1. 修改两个算子的并行度
2. 强行设定分区策略：``dataStream.rebalance();``

## 两阶段聚合
所谓两阶段聚合，即在需要shuffle的两个算子之间，再加一层算子

### 目标
先进行一次聚合，减小算子2和算子3之间的数据量，减轻算子2和算子3之间的热点问题
新增新的shuffle，打散算子1和算子2之间的数据，减轻算子1和算子2之间的热点问题

### 手段
我们以sql的优化作为范例进行讲解，这样更加直观和简洁。DataStream API无非就是仿照sql的group by + agg模式，增加一层keyby + agg。

#### 修改sql
有如下需求，按天统计每个类目的成交额

``` sql
SELECT 
    date_format(ctime, '%Y%m%d') as cdate, -- 将数据从时间戳格式（2018-12-04 15:44:54），转换为date格式(20181204)
       category_id,
    sum(price) as category_gmv
FROM src
GROUP BY date_format(ctime, '%Y%m%d'), category_id; --按照天做聚合
``` 
以这个SQL为例，其数据流程图如下，一个小方块表示一条成交记录，不同颜色代表不同的category_id
Group By + Agg 模式中，SQL作业性能与数据分布非常相关，如果数据中存在数据倾斜，也就是某个key的数据异常的多，那么某个聚合节点就会成为瓶颈，作业就会有明显的反压及延时现象。 
用两阶段聚合方法优化后的SQL如下：

``` sql
SELECT cdate,category_id,sum(category_gmv_p) as category_gmv
FROM(
    SELECT 
        date_format(ctime, '%Y%m%d') as cdate, -- 将数据从时间戳格式（2018-12-04 15:44:54），转换为date格式(20181204)
           category_id,
        sum(price) as category_gmv_p
    FROM src
    GROUP BY category_id, mod(hash_code(FLOOR(RAND(1)*1000), 256),date_format(ctime, '%Y%m%d'); --按照天做聚合
)
GROUP BY cdate,category_id
```

SQL中做了将一个Group By+Agg拆称了两个，子查询里按照category_id和mod(hash_code(FLOOR(RAND(1)*1000), 256)分组，将同一个category_id上的数据打散成了256份，先做一层聚合。外层Group By+Agg，将子查询聚合后的结果再次做聚合。这样通过两层聚合的方式，即可大大缓解某聚合节点拥堵的现象。其数据流程图如下：
这种方法达到了两个优化目标，在日期的基础上再将数据分成256份，打散数据，减轻算子1和算子2之间的热点问题；在算子2进行了初步的sum聚合，减小了到达算子3的数据量，减轻了算子2和算子3之间的热点问题。 该方法通过取余的方式将数据进一步打散，另有给key添加随机数的方式进行打散

#### Local-Global

LocalGlobal和PartialFinal其实都属于两阶段聚合，只不过封装了拆解逻辑，我们只需要对Flink SQL任务做简单的配置即可。

LocalGlobal优化可以用来解决聚合时的数据倾斜问题。其核心思想是，将聚合分为两个阶段执行，先在上游进行局部(本地/Local)聚合，再在下游进行全局(Global)聚合，类似MapReduce的Combine + Reduce，即先进行一个本地Reduce，再进行全局Reduce。该方法，只完成了先进行一次聚合，减少数据量这个目标
以如下场景为例

``` sql
SELECT color, sum(id)
FROM T
GROUP BY color
```

开启LocalGlobal：

``` java
TableEnvironment tEnv = ...
Configuration configuration = tEnv.getConfig().getConfiguration();

// 要使用LocalGlobal优化，需要先开启MiniBatch 
configuration.setString("table.exec.mini-batch.enabled", "true"); 
configuration.setString("table.exec.mini-batch.allow-latency", "5 s");
configuration.setString("table.exec.mini-batch.size", "5000");

// 开启LocalGlobal
configuration.setString("table.optimizer.agg-phase-strategy", "TWO_PHASE");
```

#### Partial-Final

LocalGlobal优化针对普通聚合（例如SUM、COUNT、MAX、MIN和AVG）有较好的效果，对于COUNT DISTINCT收效不明显，因为COUNT DISTINCT在Local聚合时，对于DISTINCT KEY的去重率不高，导致在Global节点仍然存在热点
如下场景，统计一天的UV

``` sql
SELECT day, COUNT(DISTINCT user_id)
FROM T
GROUP BY day
``` 

如果user_id比较稀疏，即便开启了LocalGlobal优化，收效也并不明显，因为COUNT DISTINCT在Local阶段时，去重率并不高，这就导致在Global阶段仍然存在热点问题。不满足第一条目标和第二条目标。
为了解决这一问题，需要将原始聚合拆分成两层聚合:

``` sql
SELECT day, SUM(cnt)
FROM (
    SELECT day, COUNT(DISTINCT user_id) as cnt
    FROM T
    GROUP BY day, MOD(HASH_CODE(user_id), 1024)
)
GROUP BY day
```

现在Blink Planner提供了PartialFinal功能，无需自己拆解sql，只要简单的配置即可，配置如下：

``` java
TableEnvironment tEnv = ...
Configuration configuration = tEnv.getConfig().getConfiguration();

// 开启MiniBatch 
configuration.setString("table.exec.mini-batch.enabled", "true"); 
configuration.setString("table.exec.mini-batch.allow-latency", "5 s");
configuration.setString("table.exec.mini-batch.size", "5000");

// 开启LocalGlobal
configuration.setString("table.optimizer.agg-phase-strategy", "TWO_PHASE");

// 开启Split Distinct
configuration.setString("table.optimizer.distinct-agg.split.enabled", "true");
``` 