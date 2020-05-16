---
title: Redo log in InnoDB
date: 2016-07-09 17:33:15
tags: [database, mysql]
categories: Technology
---

## What is redo log

For a relational database, ACID is a set of properties that it must support for a transaction. That is to say, a transaction should be atomic, consistent, isolated and durable under the management of such database.
InnoDB, the default storage engine since MySQL 5.5, use a method called **redo log** to implement the durability of a transaction. Redo log consists of redo log buffer and redo log file.
The redo log buffer resides in memory and is volatile while the redo log file resides in disks and is durable. Redo log records the information about a transaction. As the literal meaning of words redo log denotes, you can redo your operations after the system crashes by redo log.

<!-- more -->

## How does it work

As a transaction-based storage engine, InnoDB uses “force log at commit” mechanism to achieve durability.
So before a transaction is committed, all logs of that transaction must be flushed to redo log files. Even though the whole system crashes during the process of transaction commit, this transaction can be recovered by the redo log file after the system boots up again.
There exist a redo log memory buffer where the redo log is written to boost system performance. To ensure that redo log is written to redo log file successfully each time, the InnoDB storage engine need to call the `fsync` because `O_DIRECT` flag is not used when open redo log so that redo log is written to file system buffer firstly. Nevertheless, the time consumed by `fsync` call is up to the performance of disk. Consequently, the performance of disk determines the performance of transaction commit.[Reference1](http://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html)

Taking account of the performance problem mentioned previously, the InnoDB storage engine allows user to set up the frequency of calling `fsync`. Specifically, parameter `innodb_flush_log_at_trx_commit` is used for frequency management. `innodb_flush_log_at_trx_commit` controls the balance between strict ACID compliance for commit operations, and higher performance that is possible when commit-related I/O operations are rearranged and done in batches. You can achieve better performance by changing the default value, but then you can lose up to a second of transactions in a crash.

- The default value of 1 is required for full ACID compliance. With this value, the contents of the InnoDB log buffer are written out to the log file at each transaction commit and the log file is flushed to disk.
- With a value of 0, the contents of the InnoDB log buffer are written to the log file approximately once per second and the log file is flushed to disk. No writes from the log buffer to the log file are performed at transaction commit. Once-per-second flushing is not 100% guaranteed to happen every second, due to process scheduling issues. Because the flush to disk operation only occurs approximately once per second, you can lose up to a second of transactions with any mysqld process crash
- With a value of 2, the contents of the InnoDB log buffer are written to the log file after each transaction commit and the log file is flushed to disk approximately once per second. Once-per-second flushing is not 100% guaranteed to happen every second, due to process scheduling issues. Because the flush to disk operation only occurs approximately once per second, you can lose up to a second of transactions in an operating system crash or a power outage.[Reference2](https://book.douban.com/subject/25872763/)

## The format of redo log file

In InnoDB storage engine, the redo logs are stored in 512-byte format. This means that redo log cache, redo log files are both kept in blocks and each bock has a size of 512 bytes. Besides the log itself in block, log block header and lock block tailer are also stored in each block. In a redo log block, 12 bytes and 8 bytes are occupied by redo log header and redo log tailer respectively(So real information stored in each block is 492 bytes).
{% asset_img redo-log-block.jpg image of redo log file %}
