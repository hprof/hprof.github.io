---
layout: post
title: 记一次 MySQL 死锁故障及解决方案
excerpt:  由 SELECT...FOR UPDATE 引起的死锁故障
---

### 功能背景

游戏内有部分红包为限量红包, 玩家达到限量红包的条件时, 会在数据库中查询同配置ID的红包数量来决定是否发放.

#### DDL

```sql
CREATE TABLE red_envelope_bag (
    id BIGINT UNSIGNED NOT NULL AUTO INCREMENT,
    user_id BIGINT NOT NULL COMMENT '用户ID',
    config_id INT DEFAULT NULL COMMENT '红包配置ID',
    -- 其他字段略
    PRIMARY KEY(id),
    KEY user_id_idx(user_id) USING BTREE,
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='红包背包';
```

#### 查询 SQL

```sql
-- Java层的请求线程已自动添加事务, 此处省略事务相关语句
SELECT COUNT(*) FROM red_envelope_bag WHERE config_id = ? FOR UPDATE;
```

当在线上运行时, 出现以下日志:

```log
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

### 问题分析

- **全表扫描并锁住大量行／间隙**
  
  - 当前表上只有 `PRIMARY KEY(id)` 和 `user_id` 索引，没有针对 `config_id` 的索引。
  
  - InnoDB 在执行 `WHERE config_id = ? FOR UPDATE` 时，会退回使用 `PRIMARY KEY(id)` 索引做全表扫描。
  
  - 在 RR 隔离级别下，为了防止幻读，InnoDB 会对扫描到的每一行以及相应的间隙加上临键锁。
  
  - 由于没有合适的索引，全表扫描 + 临键锁实际上会锁住表中几乎所有的行甚至行与行之间的间隙，导致其他事务 插入、更新 依据同样条件（或根本无关联条件）的数据时被阻塞，严重影响并发性能。

- **性能瓶颈**
  
  - 全表扫描的 I/O 成本高，扫描次数随表记录数线性增长。
  
  - 大量锁竞争会引起事务排队，甚至死锁风险上升。

> [MySQL 死锁了，怎么办？ | 小林coding](https://xiaolincoding.com/mysql/lock/deadlock.html)

### 解决思路

#### 方式一: 为`config_id` 添加索引, 并添加`ORDER BY id` 来保证加锁路径一致

该方式实现简单快速, 但并非解决问题的最优方案.

#### 方式二: 建立`red_envelope_stat` 副表来对红包发放情况进行计数, 利用UPDATE的原子性来确保并发安全

**DDL:**

```sql
CREATE TABLE red_envelope_stat (
    config_id INT NOT NULL PRIMARY KEY COMMENT '红包配置ID',
    count INT NOT NULL DEFAULT 0 COMMENT '已发放数量'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='红包计数表';
```

服务器启动时, 需要将所有的`config_id`插入到数据库中.

在插入红包数据前, 先尝试更新相应的红包数量:

```sql
UPDATE red_envelope_stat 
   SET count = count + 1
 WHERE config_id = ?
   AND count < ?;
```

其中参数为最大数量. 若执行返回的条目数为1, 则表明更新成功, 可以继续执行发放逻辑, 反之则中止发放.
