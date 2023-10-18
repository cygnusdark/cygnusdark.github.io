---
title: Flink SQL TOP-N
date: 2022-12-05
categories: 大数据
tags:
  - Flink
---

## 1. 创建 Source
``` sql
CREATE TABLE `kafka_json_source_table` (
  user_id        INT,
  item_id        INT,
  category_id    INT,
  user_behavior  VARCHAR,
  time_stamp     TIMESTAMP(3),
  WATERMARK FOR time_stamp AS time_stamp - INTERVAL '3' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'oceanus_advanced4_input',    -- 替换为您要消费的 Topic
  'scan.startup.mode' = 'latest-offset', -- 可以是 latest-offset / earliest-offset / specific-offsets / group-offsets / timestamp 的任何一种
  'properties.bootstrap.servers' = '10.0.0.29:9092',  -- 替换为您的 Kafka 连接地址
  'properties.group.id' = 'testGroup',     -- 必选参数, 一定要指定 Group ID
  'format' = 'json',
  'json.fail-on-missing-field' = 'false',  -- 如果设置为 false, 则遇到缺失字段不会报错。
  'json.ignore-parse-errors' = 'true'      -- 如果设置为 true，则忽略任何解析报错。
);
```

## 2. 创建 Sink
``` sql
CREATE TABLE `jdbc_upsert_sink_table` (
    win_start     TIMESTAMP(3),
    category_id   INT,
    buy_count     INT,
    PRIMARY KEY (win_start,category_id) NOT ENFORCED
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:postgresql://10.0.0.236:5432/postgres?currentSchema=public&reWriteBatchedInserts=true',              -- 请替换为您的实际 MySQL 连接参数
    'table-name' = 'oceanus_advanced4_output', -- 需要写入的数据表
    'username' = 'root',                      -- 数据库访问的用户名（需要提供 INSERT 权限）
    'password' = 'yourpassword',               -- 数据库访问的密码
    'sink.buffer-flush.max-rows' = '200',     -- 批量输出的条数
    'sink.buffer-flush.interval' = '2s'       -- 批量输出的间隔
);
```

## 3. 创建临时视图，用于将原始数据过滤、窗口聚合
``` sql
CREATE VIEW `kafka_json_source_view` AS
SELECT
  TUMBLE_START(time_stamp,INTERVAL '1' MINUTE) AS win_start,
  category_id,
  COUNT(1) AS buy_count
FROM `kafka_json_source_table`
WHERE user_behavior = 'buy'
GROUP BY TUMBLE(time_stamp,INTERVAL '1' MINUTE),category_id;
```

## 4. 统计每分钟 Top3 购买种类
``` sql
INSERT INTO `jdbc_upsert_sink_table`
SELECT
b.win_start,
b.category_id,
CAST(b.buy_count AS INT) AS buy_count
FROM (SELECT *
          ,ROW_NUMBER() OVER (PARTITION BY win_start ORDER BY buy_count DESC) AS rn
      FROM `kafka_json_source_view`
      ) b
WHERE b.rn <= 3;
```

## 5. 总结
本文使用 TUMBLE WINDOW 配合 ROW_NUMBER 函数，统计分析了每分钟内购买量前三的商品种类，用户可根据实际需求选择相应的窗口函数统计对应的 TopN。更多窗口函数的使用参考时间窗口函数。