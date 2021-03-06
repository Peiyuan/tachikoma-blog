+++
draft = false
title = "[翻译] Fixing MySQL group commit"
date = 2014-07-23T22:20:00Z
tags = [ "mysql", "group commit"]
+++

# 翻译之前

Kristian Nielsen写了Fixing MySQL group commit系列共四篇blog ([第一篇](http://kristiannielsen.livejournal.com/12254.html), [第二篇](http://kristiannielsen.livejournal.com/12408.html), [第三篇](http://kristiannielsen.livejournal.com/12553.html), [第四篇](http://kristiannielsen.livejournal.com/12810.html)). 
读完后对group commit的理解我觉得很有帮助, 因此想翻译前三篇, 借此再整理一下自己的思路; 第四篇偏重具体实现, 故不包括

# 第一篇

这个系列三篇文章描述了MySQL/MariaDB是如何支持group commit这个特性的. Group commit是对数据库性能一次重大的提升. 向持久存储里写数据的开销较大, group commit能减轻这种开销对数据库整体性能的影响.

如下图所示, group commit对性能有很大提升:

![引用原文图](http://knielsen-hq.org/maria/fix-group-commit-1.png "引用原文图")

蓝色和黄色的上升线是group commit开启时的TPS, 其对性能改善的程度随着并发事务数的上升而提升


## 持久化和group commit

在一个传统事务系统中, 当事务提交成功时, 我们认为事务已经被_持久化(Durable)_了. Durable就是ACID中的D, 其含义是某事务提交成功后, 即使系统在其提交成功后任意时刻崩溃(比如电源故障, 内核崩盘, 服务器软件悲剧, 还有很多很多), 系统重启且从崩溃恢复后, 该事务的状态仍是提交成功的.

确保持久化的通常手段是:提交时, 将足够的信息写入事务日志文件(transactional log file),然后用`fsync()`将数据强制刷到磁盘上, 最后提交操作才成功返回.

凭借这些信息数据库在崩溃重启后能进行完整恢复. 当然除了`fsync`, 刷盘也可以通过`fdatasync()`系统调用或者打开日志文件时用`O_DIRECT`选项, 为了简便, 我们用`fsync()`来代指刷盘操作.

`fsync()`是个昂贵的操作. 传统硬盘(HDD)每秒可以进行150次`fsync()`, 而固态硬盘(比如Intel X25-M)每秒可进行1200次. 如果使用带电cache的RAID控制器, 可以减少`fsync()`带来的性能影响, 但不能完全消除 ([译注]我也不太理解这句...).

(除了`fsync()`, 也有其他的手段可以实现持久化. 比如在同步复制的集群(NDB,Galera)里, 假设全部节点不会同时故障, 那事务同步复制到多个节点, 就可以认为事务是持久化的. 不过不论用什么持久化方法, 较之只提交到本地内存, 持久化的代价要昂贵很多)

如果每个提交都进行`fsync()`, 受限于`fsync()`的成本, 数据库TPS被限制在每秒150个事务(HDD). 
Group commit能改善这个状况. 我们可以用一个`fsync()`来合并多个事务同时发生的刷盘请求. 处理多个事务的刷盘请求, 较之处理一个事务, `fsync()`的成本差别不大, 所以如性能图表所示, 合并刷盘请求能大幅提高性能.

## Group commit in Mysql/MariaDB

Mysql在使用InnoDB存储引擎时可以提供完整的ACID. 对于InnoDB, 开启配置`innodb_flush_log_at_trx_commit=1`时可保证持久性. MariaDB使用XtraDB的情况与之类似.

使用持久化的原因, 一方面是已经提交的事务可以不受系统崩溃的影响, 另一方面, 是可以将数据库作为replication(数据复制)的master(复制源)

数据复制使用binlog作为手段时, 保证binlog中的数据内容和存储引擎中的数据内容完全一致就很重要. 如果无法保证两者一致, slave(复制目标)将得到和master不一致的数据, 会产生无法估计的影响, 比如在master上进行的SQL无法在slave上成功执行. 
如果不保证持久化, 在系统崩溃时很多数据将丢失, 如果存储引擎中丢失的数据和binlog中丢失的数据不一样多, 那最终两者数据将不一致.
所以, 当使用binlog时, 保证持久化是MySQL/MariaDB能从崩溃正确恢复并达成最终数据一致性的前提.

MySQL/MariaDB通过XA/(binlog和存储引擎的)二段提交来保证持久化. 提交一个事务有三个步骤:

1. 准备阶段, 事务在存储引擎上进行持久化 ([译注] 指的可能是innoDB的undo log). 这个阶段完成后, 该事务仍可以被回滚. 如之后发生崩溃, 该事务可以被恢复. 
2. 准备阶段成功后, 事务在binlog上进行持久化.
3. 最后, 提交阶段, 存储引擎将事务真正提交. 完成这步后, 事务将不可被回滚.

当系统崩溃并重启后, 恢复过程将扫描binlog. binlog中准备阶段成功但没有提交的事务将进入提交阶段. 其他准备阶段成功的事务([译注] 不在binlog中的事务)将会被回滚. 以此来保证存储引擎和binlog间的数据一致性.

以上三个步骤中, 每一步骤都需要进行`fsync()`, 相比禁用binlog时一个commit只需调用一次`fsync()`, 这种方式比较昂贵, 使得group commit优化更为重要.

不幸的是当启用binlog时, group commit在MySQL/MariaDB上不能工作! Peter Zaissev在2005年就报了这个著名的[Bug#13669](http://bugs.mysql.com/bug.php?id=13669)

如在开篇的图表和性能测试中所示, 我们在一个数据库服务器上跑了很多小事务(在小XtraDB表上使用REPLACE语句), 对比了启用和禁用binlog的情况. 这种性能测试的瓶颈在于持久化时`fsync()`操作的吞吐量.

我们用了两种不同的服务器来进行性能测试,一种有两块Western Digital 10k rpm HDD存储(binlog和XtraDB log写在不同的存储上); 另一种有一块Intel X25-M SSD存储. 两种服务器都运行MariaDB 5.1.44, 都开启了持久化提交, 也都关闭了存储缓存(否则测试结果将出现偏差).

测试图表表明了不同数量的并发线程下的TPS. 对于每种服务器, 有一条线对应禁用binlog的情况, 另一条线对应开启binlog的情况.

我们看到: 在1个运行线程时, 开启binlog会有一定性能消耗, 原因如我们所料, 是因为一次提交需要调用三次`fsync()`.

更糟糕的是, 在开启binlog时group commit并不工作. 随着并发度的增加,  禁用binlog的曲线展现了良好的线性增长的性能, 但开启binlog时的性能曲线则死水一滩. 随着Group commit失效, 高并发时开启binlog的成本高的可怕(HDD存储, 64个并发线程时, 开启binlog将带来两个数量级(大于100倍)的性能损失)

第一篇的结论是: 如果我们能在开启binlog时进行group commit优化, 从而解决`fsync()`带来的性能瓶颈, 那么将获得巨大的性能提升.

第二篇将深入探讨为什么开启binlog时group commit的代码会失效. 第三篇将讨论怎样修复这个bug.

# 第二篇

InnoDB/XtraDB是支持group commit的. group commit在`innobase_commit()`函数中分为两部分完成: 第一部分称为"快"部分, 是在内存中准备要提交的信息

```
trx->flush_log_later = TRUE;
innobase_commit_low(trx);
trx->flush_log_later = FALSE;
```

第二部分称为"慢"部分, 其调用`fsync()`将提交信息刷磁盘.

```
trx_commit_complete_for_mysql(trx)
```

当一个事务提交正在执行"慢"部分时, 之后提交的事务可以同时完成其"快"部分, 然后进入队列等待正在进行的`fsync()`完成. 一旦正在进行的`fsync()`完成, 一个新的`fsync()`可以将队列中等待的所有事务一次刷往磁盘. 当禁用binlog时group commit就是这样工作的.

当开启binlog时, Mysql用XA/二段提交来保证binlog和存储引擎间的数据一致性. 那一次commit就分为以下三个步骤:

```
innobase_xa_prepare()
write() and fsync() binary log
innobase_commit()
```

需要注意的是是InnoDB在`innobase_xa_prepare()`锁住了`prepare_commit_mutex`, 一直到`innobase_commit()`中的"快"部分结束才释放这个锁. 这意味着当一个事务正在执行`innobase_commit()`时, 之后提交的事务都被阻塞在`innobase_xa_prepare()`中等待锁释放. 结果是没有事务会在队列中等待`fsync()`, group commit因此失效.

那现在的问题是开启binlog时为什么InnoDB要锁住`prepare_commit_mutex`? 这是个非常优秀的问题, 在开展一系列调查后, 好像根本没有强有力的理由来支持这个锁的存在.

无论是InnoDB代码注释, 还是bug追踪系统, 或是其它资料, 提及这个锁时都说它是用来确保在存储引擎和binlog中的提交顺序一致. 确实, 如果没有这把锁, 两个事务在存储引擎中的提交顺序是AB, 而在binlog中的顺序可能是BA.

那么下一个问题是: 为什么要保证存储引擎和binlog的提交顺序一致? 目力所及的唯一原因是InnoDB热备和XtraBackup需要保证备份中的数据文件和备份中记录的binlog位置一致.

Sergei Golubchik在2010 MySQL conference期间对此做了一些研究. 结论是XtraDB在获取binlog位置之前会加全局锁`FLUSH TABLE WITH READ LOCK`, 这个锁完全阻塞所有commit, 此时可以保证数据文件和binlog位置是一致的. (InnoDB热备工具是非开源的, 但也应是相同的工作机理). 所以对于备份是不需要用`prepare_commit_mutex`保证(非备份时)数据文件和binlog位置对齐的.

另一种热备方法是LVM快照. LVM快照恢复后启动Mysql服务器, 会进入数据库恢复流程, 来确保数据文件和binlog的数据一致, 也不需要`prepare_commit_mutex`.

所以我们并不能为`prepare_commit_mutex`的存在找到合适的理由. (然后作者在此进行了吐槽, 大意是"你们这些猪头怎么为了这么个荒谬的理由阻碍了group commit的发展?!")

(为了实现完全的group commit, MySQL还需要做另一个修改, 就是将binlog也实现group commit)

第二篇的结论是: 如果抛弃了`prepare_commit_mutex`, MySQL可以迎来有group commit的美好时代. 第三篇将更深入进行讨论.

# 第三篇

这一篇将讨论如何修复group commit的问题. 如第二篇所述, 我们可以去掉`prepare_commit_mutex`, 然后再为binlog加上独立的group commit功能, 这个问题就可以解决.

然而, 我们能做的更好. 第一篇中提到, 开启binlog是我们需要XA来保证系统崩溃并恢复后存储引擎和binlog的数据最终一致性, 这种情况下1个提交需要进行3次`fsync()`刷盘. 尽管用我们修复的group commit可以抵消一部分`fsync()`开销, 但开销仍然很客观. 在[maria-developers邮件列表](https://launchpad.net/~maria-developers)中的一个[讨论](https://lists.launchpad.net/maria-developers/msg01998.html)中, 提到一种方法可以让3次`fsync()`刷盘降为仅1次

这个方法就是仅在为binlog刷盘时进行`fsync()`, 也就是让`innodb_flush_log_at_trx_commit`运行在级别2甚至级别0上.

为了描述这种方法, 假设某个事务A已经写入binlog文件, `fsync()`保证binlog文件已经刷盘. 然后事务A被提交给存储引擎, 在存储引擎将事务A刷盘前系统崩溃.

系统恢复后进入数据库恢复流程, 此时事务A存在于binlog中, 但不存在于存储引擎中, 产生数据不一致. 这个不一致可以通过重放binlog中的事务A来解决. 重放binlog就像在复制的slave端所做的一样. 重放成功后, 就可以恢复成一致状态.

为了能按上述步骤恢复到数据一致状态, 我们需要以下两个条件:

* 对于一个事务, 需要将其在binlog中的位置存到存储引擎中, 这样在崩溃恢复中, 我们才能知道应从binlog哪个位置起开始回放事务.
* 这次我们真的需要保证binlog和存储引擎的commit顺序一致! 否则崩溃恢复时, 有可能的情况是: binlog中两个事务的顺序是AB, 而存储引擎中仅有B被提交, 而A没被持久化到存储引擎中. 这样我们就没法决定如何恢复一致性了.

为了确保binlog和存储引擎的提交顺序一致, 我们不能再回到`prepare_commit_mutex`的方案了, 否则开启binlog时group commit又会失去作用. 我们将用另外的方法来确保顺序. Mark Callaghan在MySQL conference上提到了这样的方法, 可以参看[这里](http://www.facebook.com/note.php?note_id=386328905932).

基本方案就是当事务被写进binlog时, 我们记住其顺序. 我们可以将事务放到一个队列里, 或者为每个事务分配单调递增的全局序号, 或者如Mark所述为每个事务分配某种ticket. 那么在`innobase_commit()`中事务可以通过上述某种方法来保持顺序.

[译注: 理论部分到此结束. 这之后都是作者在简单叙述其解决方案, 之后就不译了. 出门左转见[原文](http://kristiannielsen.livejournal.com/12553.html)]

