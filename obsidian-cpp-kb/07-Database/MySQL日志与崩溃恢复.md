---
tags: [database, mysql, redo-log, undo-log, binlog]
created: 2026-07-09
updated: 2026-07-09
status: 面试重点
---

# MySQL 日志与崩溃恢复

## 一句话理解

`redo log` 保证提交后的修改崩溃不丢，`undo log` 保证事务能回滚并支撑 MVCC，`binlog` 用于主从复制和增量恢复；两阶段提交保证 redo log 和 binlog 一致。

## 三种日志分别做什么

| 日志 | 所属层 | 主要作用 |
|------|--------|----------|
| redo log | InnoDB 引擎层 | 崩溃恢复，保证持久性 |
| undo log | InnoDB 引擎层 | 事务回滚，支撑 MVCC |
| binlog | MySQL Server 层 | 主从复制，基于备份做增量恢复 |

一句话记忆：

```text
redo log：提交后不丢
undo log：失败后能撤销
binlog：变更能复制和重放
```

面试表达：

> redo log 是 InnoDB 的物理日志，主要用于崩溃恢复，保证事务提交后的修改不丢；undo log 记录修改前的旧版本，用于事务回滚和 MVCC；binlog 是 MySQL Server 层的逻辑日志，用于主从复制和基于备份的数据恢复。

## redo log

数据库修改数据时，不会每次都立刻把数据页刷回磁盘，因为随机写数据页成本很高。

常见流程：

```text
修改内存中的数据页
数据页变成脏页
先写 redo log
事务提交
脏页之后再慢慢刷盘
```

如果事务提交后，数据页还没刷盘就崩溃：

```text
MySQL 重启
读取 redo log
重放已提交事务的修改
恢复脏页对应的数据
```

所以 redo log 主要保证事务的持久性。

面试表达：

> InnoDB 提交事务时不要求数据页立刻刷盘，只要 redo log 已经持久化，崩溃后就能通过 redo log 重放已提交事务的修改，把脏页恢复出来。因此 redo log 的核心作用是崩溃恢复，保证事务提交后数据不丢。

## undo log

undo log 记录数据被修改前的旧版本。

用途：

- 事务回滚：撤销事务已经做过的修改。
- MVCC：为快照读提供历史版本。

不同操作的回滚思路：

| 操作 | 回滚方式 |
|------|----------|
| insert | 删除插入的记录 |
| delete | 恢复被删除的记录 |
| update | 恢复修改前的旧值 |

例子：

```sql
update user set age = 20 where id = 1;
```

如果原来 `age = 18`，undo log 会记录旧值。事务回滚时，InnoDB 可以把 `age` 恢复成 `18`。

面试表达：

> undo log 记录数据被修改前的旧版本。事务回滚时，InnoDB 根据 undo log 生成反向操作，把已经执行的修改撤销掉，从而保证原子性。除此之外，undo log 还用于 MVCC，为快照读提供历史版本。

## binlog

binlog 是 MySQL Server 层日志，记录数据库的逻辑变更。

主要用途：

- 主从复制：从库读取主库 binlog 并重放。
- 增量恢复：配合全量备份恢复到某个时间点。
- 审计变更：知道执行过哪些数据变更。

binlog 不是 InnoDB 特有的，它在 Server 层。

面试表达：

> binlog 是 MySQL Server 层的逻辑日志，主要用于主从复制和增量恢复。从库通过读取主库 binlog 重放数据变更；恢复数据时，也可以在全量备份基础上重放 binlog，把数据恢复到指定时间点。

## redo log 和 binlog 的区别

| 对比 | redo log | binlog |
|------|----------|--------|
| 所属层 | InnoDB 引擎层 | MySQL Server 层 |
| 记录内容 | 物理页修改 | SQL 或行级逻辑变更 |
| 主要用途 | 崩溃恢复 | 主从复制、增量恢复 |
| 写入方式 | 循环写，空间会复用 | 追加写，写满切新文件 |
| 是否 InnoDB 特有 | 是 | 否 |

为什么不能只要 binlog：

- binlog 更偏逻辑变更，不适合 InnoDB 崩溃后快速恢复脏页。
- InnoDB 需要 redo log 来恢复已提交但尚未刷盘的数据页。

为什么不能只要 redo log：

- redo log 是循环写，不适合长期保存完整变更历史。
- redo log 不适合作为主从复制和基于备份恢复的日志。

面试表达：

> redo log 和 binlog 都记录数据变更，但定位不同。redo log 是 InnoDB 引擎层的物理日志，用于崩溃恢复，保证事务持久性；binlog 是 Server 层的逻辑日志，用于主从复制和基于备份的增量恢复。redo log 不能替代 binlog，因为它不适合复制和长期恢复；binlog 也不能替代 redo log，因为它不适合 InnoDB 崩溃后恢复脏页。

## 两阶段提交

两阶段提交解决的是 redo log 和 binlog 的一致性问题。

如果没有两阶段提交，可能出现：

```text
redo log 写成功
binlog 还没写
MySQL 崩溃
```

主库重启后能根据 redo log 恢复这条数据，但 binlog 没有这条记录，从库无法同步。

也可能出现：

```text
binlog 写成功
redo log 还没提交
MySQL 崩溃
```

从库可能根据 binlog 执行了变更，但主库崩溃恢复时没有恢复这条数据，也会不一致。

两阶段提交流程：

```text
1. redo log prepare
2. 写 binlog
3. redo log commit
```

崩溃恢复时，MySQL 可以根据 redo log 和 binlog 的状态判断事务是否完整提交。

面试表达：

> 两阶段提交是为了解决 redo log 和 binlog 的一致性问题。redo log 用于崩溃恢复，binlog 用于主从复制和增量恢复。如果只写其中一个就崩溃，可能导致主库恢复结果和从库复制结果不一致。两阶段提交先把 redo log 写成 prepare 状态，再写 binlog，最后提交 redo log，从而保证两份日志一致。

## 崩溃恢复怎么理解

事务提交成功后，数据页没刷盘也不会丢，前提是 redo log 已经持久化。

```text
事务提交成功
redo log 已持久化
数据页还是内存脏页
MySQL 崩溃
重启后重放 redo log
恢复已提交事务修改
```

需要区分：

- redo log：用于 InnoDB 自己恢复数据页。
- binlog：用于复制和备份恢复链路。

面试表达：

> 事务提交成功后，即使数据页还没刷盘，MySQL 崩溃后也不会丢已提交的数据。因为 InnoDB 可以通过已经持久化的 redo log 重放修改，把脏页恢复出来。binlog 主要用于复制和增量恢复，两阶段提交保证它和 redo log 的事务状态一致。

## 容易踩坑的地方

1. 把两阶段提交说成“避免数据页没刷盘导致丢失”；这其实是 redo log 的作用。
2. 以为 binlog 也负责 InnoDB 崩溃后恢复脏页；主力是 redo log。
3. 只知道 undo log 支撑 MVCC，忘记它也用于事务回滚。
4. 分不清 redo log 是引擎层，binlog 是 Server 层。
5. 不知道 redo log 循环写，binlog 追加写。
6. 两阶段提交的核心是保证 redo log 和 binlog 一致，避免主从或恢复结果不一致。

## 我的薄弱点

- redo log / undo log / binlog 这轮开始时基本不了解，需要后续按作用反复复测。
- 已能说出事务提交后数据页未刷盘不会丢，但要记住主力是 redo log，不是 binlog。
- 两阶段提交曾误解成“防止刷盘异常导致数据丢失”，需要纠正为“保证 redo log 和 binlog 一致”。
- undo log 能想到回滚，但要补牢它记录旧版本，同时服务于 MVCC。

## 成长记录

- 已能理解数据页不会每次修改都立即刷盘，因此需要 redo log。
- 已能说出提交成功后，redo log 可以帮助恢复未刷盘的数据页。
- 已能说出 undo log 用于事务回滚，按旧值反向恢复。
- 下一轮重点复测：三种日志的作用、redo log 和 binlog 区别、两阶段提交流程、崩溃恢复时 redo log 的作用。

## 面试高频问题

1. redo log、undo log、binlog 分别是干什么的？
2. redo log 为什么能保证事务持久性？
3. 为什么事务提交后数据页没刷盘也不会丢？
4. undo log 除了 MVCC 还能做什么？
5. 事务回滚时 undo log 怎么起作用？
6. binlog 有什么用？
7. redo log 和 binlog 有什么区别？
8. 为什么不能只用 binlog，不用 redo log？
9. 为什么不能只用 redo log，不用 binlog？
10. 什么是两阶段提交？
11. 两阶段提交解决什么问题？
12. redo log prepare、写 binlog、redo log commit 分别处于什么阶段？
13. MySQL 崩溃恢复大概怎么做？

## 关联知识

- [[MySQL事务、MVCC与锁]]
- [[MySQL索引与SQL优化]]
- [[Linux常用排查工具]]
