---
title: TiDB 悲观事务模式
category: reference
---

# TiDB 悲观事务模式

TiDB 默认使用乐观事务模式，存在事务提交时因为冲突而失败的问题。为了保证事务的成功率，需要修改应用程序，加上重试的逻辑。
悲观事务模式可以避免这个问题，应用程序无需添加重试逻辑，就可以正常执行。

> **注意：**
>
> 悲观事务模式目前是一个**试验性**的特性，**不建议在生产环境中使用**。

## 悲观事务模式的行为

悲观事务的行为和 MySQL 基本一致（不一致的地方详见[已知的局限](#已知的局限)）：

- `SELECT FOR UPDATE` 会读取已提交的最新数据，并对读取到的数据加悲观锁。

- `UPDATE`/`DELETE`/`INSERT` 语句都会读取已提交的最新的数据来执行，并对修改的数据加悲观锁。

- 当一行数据被加了悲观锁以后，其他尝试修改这一行的写事务会被阻塞，等待悲观锁的释放。

- 当一行数据被加了悲观锁以后，其他尝试读取这一行的事务不会被阻塞，可以读到已提交的数据。

- 事务提交或回滚的时候，会释放所有的锁。

- 如果并发事务出现死锁，会被死锁检测器检测到，并返回和 MySQL 相同的 DEADLOCK 错误。

- 乐观事务和悲观事务可以共存，事务可以任意指定使用乐观模式或悲观模式来执行。

## 悲观事务的启用方法

因为悲观事务目前还是一个试验性特性，默认是关闭的，启用悲观事务首先需要在 TiDB 的配置文件里添加：

```
[pessimistic-txn]
enable = true
```

`enable` 设置为 `true` 以后，默认的事务模式仍然是乐观事务模式，要进入悲观事务模式有以下三种方式：

- 执行 `BEGIN PESSIMISTIC;` 语句开启的事务，会进入悲观事务模式。
可以通过写成注释的形式 `BEGIN /*!90000 PESSIMISTIC */;` 来兼容 MySQL 语法。

- 执行 `set @@tidb_txn_mode = 'pessimistic';`，使这个 session 执行的所有事务都会进入悲观事务模式。

- 在配置文件设置中默认启用悲观事务模式，使除了自动提交的单语句事务之外的所有事务都会进入悲观事务模式。

```
[pessimistic-txn]
enable = true
default = true
```

在配置了默认启用悲观事务的情况下，可以用以下两种方式使事务进入乐观事务模式：

- 执行 `BEGIN OPTIMISTIC;` 语句开启的事务，会进入乐观事务模式。
可以通过写成注释的形式 `BEGIN /*!90000 OPTIMISTIC */;` 来兼容 MySQL 语法。

- 通过执行 `set @@tidb_txn_mode = 'optimistic';`，使当前的 session 执行的事务进入乐观事务模式。

## 启用方式的优先级

- 优先级最高的是 `BEGIN PESSIMISTIC;` 和 `BEGIN OPTIMISTIC;` 语句。

- 其次是 session 变量 `tidb_txn_mode`。

- 最后是配置文件里的 `default`，当使用普通的 `BEGIN` 语句，且 `tidb_txn_mode` 的值为空字符串 `''` 时，根据 `default` 来决定启用悲观事务还是乐观事务。  

## 相关配置参数

相关配置都在 `[pessimistic-txn]` 类别下，除了前面介绍过的 `enable` 和 `default`，还可配置以下参数：

- `ttl` 

    ```
    ttl = "30s"
    ```

    `ttl` 是悲观事务的锁超时时间，默认值 30 秒，可以在 15 秒到 60 秒之间配置，超出这个范围会报错。如果事务的执行时间超过了 `ttl`，事务会失败。设置过大，会存在 `tidb-server` 宕机时，残留的悲观锁长时间阻塞写的问题，设置的过小，会存在事务来不及执行完，就被其他事务 rollback 的问题。

- `max-retry-count`

    ```
    max-retry-count = 256
    ```

    悲观事务会在事务内自动重试单个语句，这个配置指定单个语句最大重试次数，设置这个参数是为了避免极端情况下，出现无限重试，无法停止。
正常情况下不需要修改这个配置。

## 已知的局限

- 不支持 GAP Lock 和 Next Key Lock 在悲观事务内通过范围条件来更新多行数据的时候，其他的事务可以在这个范围内插入数据而不会被阻塞。

- 不支持 SELECT LOCK IN SHARE MODE。