---
title: MySQL 之怎么保证数据不丢
date: 2021-11-15 14:03:46
tags: MySQL
categories: MySQL
---

![思维导图](MySQL-之怎么保证数据不丢/思维导图.png)



MySQL 的 WAL 机制，全程是 `Write-Ahead Logging`，预写日志系统。指的是 MySQL 的写操作并不是立刻更新到磁盘上，而是先记录在日志上，然后在合适的时间更新到磁盘上。日志主要分为 undo log、redo log 和 binlog，作用分别是「完成 MVCC 从而实现 MySQL 的隔离级别」「降低随机写的性能消耗（转成顺序写），同时防止写操作因为宕机而丢失」「写操作的备份，保证主从一致」



其实，只要 redo log 和 binlog 保证持久化到磁盘，就能确保 MySQL 异常重启后，数据可以恢复。那么，MySQL 写入 binlog 和 redo log 的流程是怎样的呢



### binlog 的写入机制



binlog 的写入逻辑比较简答：**事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中**



一个事务的 binlog 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。这就涉及到了 binlog cache 的保存问题



系统给 binlog cache 分配了一片内存，**每个线程一个**，参数 `binlog_cache_size` 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘



事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 中，并清空 binlog cache。状态如图 1 所示：



![binlog写盘状态](MySQL-之怎么保证数据不丢/binlog写盘状态.png)



可以看到，每个线程都有自己 binlog cache，但是共用同一份 binlog 文件



- 图中的 **write**，指的就是把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快
- 图中的 **fsync**，才是将数据持久化到磁盘的操作。一般情况下，我们认为 `fsync` 才占磁盘的 IOPS



write 和 fsync 的时机，是由参数 `sync_binlog` 控制的：



1. sync_binlog = 0 的时候，表示每次提交事务都只 write，不 fsync
2. sync_binlog = 1 的时候，表示每次提交事务都会执行 fsync
3. sync_binlog = N（N > 1）的时候，表示每次提交事务都 write，但累积 N个事务后才 fsync



因此，在出现 IO 瓶颈的场景里，将 sync_binlog 设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设置成 0，比较常见的是将其设置为 100 ~ 1000 中的某个数值



但是，将 sync_binlog 设置为 N，对应的风险是：如果主机发生异常重启，会丢失最近 N 个事务的 binlog 日志



### redo log 的写入机制



之前说过，事务在执行过程中，生成的 redo log 是要先写到 redo log buffer 的。那么，redo log buffer 里面的内容，是不是每次生成后都要直接持久化到磁盘呢？答案是，不需要



如果事务执行期间 MySQL 发生异常重启，那这部分日志就丢了。由于事务并没有提交，所以这时日志丢了也不会有损失



那么，另一个问题是，事务还没提交的时候，redo log buffer 中的部分日志有没有可能被持久化到磁盘呢？答案是，确实会有



这个问题，要从 redo log 可能存在的三种状态说起。这三种状态，对应的就是图 2 中的三个颜色块



![redo_log存储状态](MySQL-之怎么保证数据不丢/redo_log存储状态.png)



这三种状态分别是：



1. 存在 redo log buffer 中，物理上是在 MySQL 进程内存中
2. 写到磁盘（write），但是并没有持久化（fsync），物理上是在文件系统的 page cache 里面
3. 持久化到磁盘，对应的是 hard disk



日志写到 redo log buffer 是很快的，write 到 page cache 也差不多，但是持久到磁盘的速度的慢多了



为了控制 redo log 的写入策略，InnoDB 提供了 `innodb_flush_log_at_trx_commit` 参数，它有三种可能取值：



1. 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中
2. 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久到磁盘
3. 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache



InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘



注意，**事务执行中间过程的 redo log 也是直接写在 redo log buffer 中的，这些 redo log 也会被后台线程一起持久化到磁盘**。即，一个没有提交的事务的 redo log，也是可能已经持久化到磁盘的



实际上，除了后台线程每秒一次的轮询操作外，还有两种场景会让一个没有提交的事务的 redo log 写入到磁盘中



1. 一种是，**redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘**。注意，由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是只留在了文件系统柜的 page cache
2. 另一种是，**并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘**。假设一个事务 A 执行到一半，已经写了一些 redo log 到 buffer 中，这时候有另外一个线程的事务 B 提交。如果 innodb_flush_log_at_trx_commit 设置的是 1，那么按照这个参数的逻辑，事务 B 要把 redo log buffer 里的日志全部持久化到磁盘。这时候，就会带上事务 A 在 redo log buffer 里的日志一起持久化到磁盘



这里需要说明的是，我们介绍两阶段提交的时候说过，时序上 redo log 先 prepare，再写 binlog，最后再把 redo log commit



如果把 `innodb_flush_log_at_trx_commit` 设置成 1，那么 redo log 在 prepare 阶段就要持久化一次，因为有一个崩溃恢复逻辑是要依赖于 prepare 的 redo log，再加上 binlog 来恢复的



每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB 就认为 redo log 在 commit 的时候就不需要 fsync 了，只会 write 到文件系统的 page cache 中就够了



通常我们说 MySQL 的「双 1」配置，指的就是 `sync_binlog` 和 `innodb_flush_log_at_trx_commit` 都设置成 1.也就是说，一个事务完成提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog



这时候，可能会有一个疑问？这意味着我从 MySQL 看到的 TPS 是每秒两万的话，每秒就会写四万次磁盘。如果磁盘能力也就两万左右，怎么能实现两万的 TPS？



这里，就要用到**组提交（group commit）**机制了



这里，需要先介绍日志逻辑序列号（log sequence number，LSN）的概念。LSN 是单调递增的，**用来对应 redo log 的一个个写入点**。每次写入长度为 length 的 redo log，LSN 的值就会加上 length



LSN 也会写到 InnoDB 的数据页中，来确保数据页不会被多次执行重复的 redo log



如下图，是三个并发事务（trx1, trx2, trx3）在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160



![redo_log组提交](MySQL-之怎么保证数据不丢/redo_log组提交.png)



从图中可以看到：



1. trx1 是第一个到达的，会被选为这组的 leader
2. 等 trx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成了 160
3. trx1 去写盘的时候，带的就是 LSN = 160，因此等 trx1 返回时，所有 LSN 小于等于 160 的 redo log，都已经被持久化到磁盘
4. 这时候 trx2 和 trx3 就可以直接返回了



所以，一次组提交里面，组员越多，节约磁盘 IOPS 的效果就越好。但如果只有单线程压测，那就只能老老实实地一个事务对应一次持久化操作了



在并发更新场景下，第一个事务写完 redo log buffer 以后，接下来这个 `fsync` 越晚调用，组员可能越多，节约 IOPS 的效果就越好



为了让一次 fsync 带的组员更多，MySQL 有一个很有趣的优化：拖时间。redo 两阶段提交的时候，流程图如下



![两阶段提交](MySQL-之怎么保证数据不丢/两阶段提交.png)



图中，把「写 binlog」当成一个动作。但实际上，写 binlog 是分成两步的：



1. 先把 binlog 从 binlog cache 中写到磁盘上的 binlog 文件
2. 调用 fsync 持久化



MySQL 为了让组提交的效果更好，把 redo log 做 fsync 的时间拖到了步骤 1 之后。即，上面的图变成了这样：



![细化两阶段](MySQL-之怎么保证数据不丢/细化两阶段.png)



这么一来，binlog 也可以组提交了。在执行图中第 4 步把 binlog fsync 到磁盘时，如果有多个事务的 binlog 已经写完了，也是一起持久化的，这样也可以减少 IOPS 的消耗



不过通常情况下第 3 步执行得很快，所以 binlog 的 `write` 和 `fsync` 间的间隔时间很短，导致能集合到一起持久化的 binlog 比较少，因此 binlog 的组提交的效果通常不如 redo log 的效果好



如果想提升 binlog 组提交的效果，可以通过设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no delay_count` 来实现



1. **binlog_group_commit_sync_delay** 参数，表示延迟多少微秒后才调用 fsync
2. **binlog_group_commit_sync_no_delay_count** 参数，表示累积多少次以后才调用 fsync



这两个条件是或的关系，也就是说只要有一个满足条件就会调用 fsync。所以，当 `binlog_group_commit_sync_delay` 设置为 0 的时候，`binlog_group_commit_sync_no_delay_count` 也无效了



WAL 机制是减少磁盘写，可是每次提交事务都要写 redo log 和 binlog，这磁盘读写次数也没变少啊



其实，WAL 机制主要得益于两个方面：



1. redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快
2. 组提交机制，可以大幅度降低磁盘的 IOPS 消耗



到这里，**如果你的 MySQL 现在出现了性能瓶颈，而且瓶颈在 IO 上，可以通过哪些方法来提升性能呢？**



针对这个问题，可以考虑一下三种方法：



1. 设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count` 参数，减少 binlog 的写盘次数。这个方式是基于「额外的故意等待」来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险 
2. 将 `sync_binlog` 设置为大于 1 的值（比较常见是 100 ~ 1000）。这样的风险是，主机掉电时会丢 binlog 日志



不建议把 `innodb_flush_log_at_trx_commit` 设置为 0。因为把这个参数设置成 0，表示 redo log 只保存在内存中，这样的话 MySQL 本身异常重启也会丢数据，风险太大。而 redo log 写到文件系统的 page cache 的速度也是很快的，所以将这个参数设置成 2 跟设置成 0 其实性能差不多，但这样 MySQL 异常重启时就不会丢数据了，相比之下风险会更小

