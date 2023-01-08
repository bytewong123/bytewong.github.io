---
title: mysql知识点汇总——如何保证ACID
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["mysql"]
tags: ["mysql"]
---

# mysql知识点汇总——如何保证ACID
#技术/数据库/mysql

## A原子性
利用InnoDB的undo log undo log（回滚日志）记录需要回滚的日志信息，是实现原子性的关键，当事务回滚时能够撤销所有已经成功执行的sql语句

undo log记录了这些回滚需要的信息，当事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的样子。

1. undo log 包含两个隐藏字段 trx_id 和 roll_pointer。
	1. trx_id 表示当前这个事务的 id，MySQL 会为每个事务分配一个 id，这个 id 是递增的。
	2. roll_pointer 是一个指针，指向这个事务之前的 undo log
> 现在有一个 id 为 10 的事务 A 正在执行，undo log 日志的roll_pointer指向一个空的undo log
> 紧接着 id 为 18 的事务 B 开始执行，就会再生成一条 undo log 日志，同时新生成的日志的 roll_pointer 指向上一条 undo log 日志
> 日志与日志之间通过 roll_pointer 指针连接，就形成了 undo log 版本链
2. 在对数据库进行修改时，innoDB引擎除了会产生redo log，还会产生undo log。在操作任何数据之前，首先将数据备份到Undo Log中，然后进行数据修改。InnoDB实现回滚，靠的是undo log：当事务对数据库进行修改时，InnoDB会生成对应的undo log；如果事务执行失败导致事务需要回滚，就利用undo log中的信息将数据回滚到修改之前的样子
3.  undolog通过redolog持久化

## D持久性
依赖于innodb的redolog

### update更新流程
1. 更新buffer pool里页的数据
2. 生成一个redolog对象
3. commit后，持久化redolog对象

> 不直接把buffer pool脏页写回磁盘，而是使用redolog，主要是因为页写回磁盘是随机io

### redolog细节
1. InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这块“粉板”总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写
2. write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件
3. write pos 和 checkpoint 之间的是“粉板”上还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，表示“粉板”满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下
4. redo log文件数可以通过innodb_log_files_in_group控制；每个文件大小可以通过innodb_log_file_size控制
5. innodb_flush_log_at_trx_commit
	1. 0：事务提交时，不立即对redolog持久化，这个任务交给后台线程去做
	2. 1：事务提交时，立即对redolog持久化
	3. 2：事务提交时，立即将redo log写到操作系统的缓冲区，并不会直接将redo log持久化，这种情况下，如果数据库挂了，操作系统没挂，那么事务的持久性还是可以保证

## I隔离性
利用锁和MVCC机制

## C一致性
一致性是目的，由AID三大特性一起保证