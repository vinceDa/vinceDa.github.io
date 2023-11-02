---
categories: [mysql]
tags: [database]
---

## 什么是redo log？
假设MySQL的每次更新操作都会写入磁盘，那么这个过程就是先在磁盘中找到这条记录，然后更新写入磁盘。这个过程中涉及了多次磁盘操作，IO成本和查找成本都很高，这势必会导致MySQL的效率问题。所以，为了解决这个问题，InnoDB的设计者就设计redo log。这个设计技术就是Write-Ahead Logging（WAL），先写日志，在写磁盘。redo log就是这个日志。



有了redo log后，一条数据的更新操作就变成了先把记录写到redo log然后更新内存。同时，InnoDB引擎会在适当的时候，将这个操作记录更新到磁盘，这个更新往往是在系统空闲时做的。



## crash-safe
redo log除了减少和磁盘的交互提高效率之外，crash-safe是它最主要的能力。crash-safe的意思是即使数据库发生异常重启，之前的提交记录都不会丢失。那么，redo log是怎么实现crash-safe的呢？



redo log是可配置的固定大小的的文件.它是一个首尾连接的环形数组，如下图所示：

<img src="https://raw.githubusercontent.com/vinceDa/image-host/main/img/MySQL/redo_log/redo_log.png" alt="redo_log.png" style="zoom:50%;" />



write pos表示当前记录的位置，一边写一边往后移；checkpoint是当前要擦除的位置，当write pos和checkpoint重合时则表示日志文件写满了，需要进行擦除，这时候InnoDB会先把日志中的记录更新到磁盘，然后擦除这部分日志腾出空间，将checkpoint往后移。通过这样一种机制，当数据库重启时能够通过redo log找回之前的提交记录。

## innodb_flush_log_at_trx_commit
innodb_flush_log_at_trx_commit这个参数用来控制redo log刷新到磁盘的策略：

- 0：表示事务提交时不进行写入重做日志操作，这个操作仅在master thread中完成，而在master thread中每秒会进行一次redo log文件的fsunc操作
- 1（默认）：表示事务提交时必须调用一次fsync操作
- 2：表示事务提交时将redo log写入redo log文件，但仅写入文件系统的缓存中，不进行fsync操作。在这个设置下，当MySQL数据库发生宕机而操作系统不发生宕机时，并不会导致事务的丢失。而当操作系统宕机时，未从文件系统缓存刷新到redo log文件中的事务将会丢失



我们可以测试一下，在不同的设置下的插入效率。
创建一个表f

```sql
CREATE TABLE `f` (
  `k` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3;
```


创建存储过程

```sql
create procedure batchInsert(a int)
begin
        declare i int default 0;
        loop_name:loop -- 循环开始
            if i>a then 
                leave loop_name;  -- 判断条件成立则结束循环  好比java中的 boeak
			insert into f values(i);
            end if;
            set i=i+1;
        end loop;  -- 循环结束
end
```


调整innodb_flush_log_at_trx_commit并分别执行50w次插入操作

```sql
call batchInsert(500000);
```


得到以下结果

| innodb_flush_log_at_trx_commit | 执行时间 |
| --- | --- |
| 0 | 1 min 14.01 sec |
| 1 | 2 min 38.10 sec |
| 2 | 1 min 29.77 sec |



从图中可以看出，将innodb_flush_log_at_trx_commit设置成0或2都可以大幅提升效率，但是我们还是建议设置为1。因为这样能够最大程度保证数据的完整性，但是如果你的业务场景需要执行快速的操作并且允许一部分数据的丢失，设置成其他的参数也是可以的。



这时候你可能又有疑问了，如果每次事务提交都刷磁盘，那这是怎么提升效率的呢？



由于write pos和checkpoint的存在，当刷磁盘的时候，我们是知道应该刷入那一部分日志的，也就是说刷盘的时候大部分情况下是顺序写的，这比随机写的效率要大得多。



## 和binlog的区别
binlog是二进制日志记录对数据发生或潜在发生更改的SQL语句，并以二进制的形式保存在磁盘中。主要用于数据库的主从复制以及增量恢复，既然binlog也能用于数据恢复，为什么还要有redo log？两者的区别是什么呢？



先回答第一个问题。因为最开始MySQL没有InnoDB引擎，默认的MyISAM引擎不支持crash-safe能力，而且binlog只能用于归档，它是以追加写的形式记录日志，当crash的时候不能确认哪些记录没有刷入磁盘。所以InnoDB实现了redo log来支持crash-safe，它通过循环写的方式记录日志，当crash的时候write pos和checkpoint中间的数据就是没有刷入磁盘的数据。



这两种日志有以下几个区别

1. redo log是InnoDB特有的；binlog是MySQL的Server层实现的，所有引擎都可以使用
2. redo log是物理日志，记录的是“在某个数据也做了什么修改”；binlog是逻辑日志，记录的是这个语句的原始逻辑，给 ID=2 这一行的 c 字段加 1 ”。
3. redo log是循环写的，当文件写满后需要清理空间到磁盘上，之前的日志会被覆盖；binlog是追加写，文件写满后会切换到下一个文件，不会覆盖之前的日志。
4. 

## 总结
今天，我介绍了redo log。redo log是为了提供crash-safe能力。innodb_flush_log_at_trx_commit表示事务提交时刷盘的策略，建议设置成1，牺牲操作的部分性能换取保证数据的完整性。同时也介绍了redo log和binlog的区别，两者的用途不同，大家不要混淆。



## 参考资料

1. [极客时间-MySQL45讲](https://time.geekbang.org/column/intro/100020801?tab=catalog)
2. MySQL技术内幕：InnoDB存储引擎（第2版）
