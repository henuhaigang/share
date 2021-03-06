# 一 InnoDB Lock
---
## 1 隐式锁
InnoDB 通常对插入操作无需加锁，而是通过一种“隐式锁”的方式来解决冲突。聚集索引记录中存储了事务id，如果另外有个session查询到了这条记录，会去判断该记录对应的事务id是否属于一个活跃的事务，并协助这个事务创建一个记录锁，然后将自己置于等待队列中
## 2 间隙锁（LOCK_GAP/行级）
表示只锁住一段范围，不锁记录本身，通常表示两个索引记录之间，或者索引上的第一条记录之前，或者最后一条记录之后的锁。可以理解为一种区间锁，一般在RR隔离级别下会使用到GAP锁。你可以通过切换到RC隔离级别，或者开启选项innodb_locks_unsafe_for_binlog来避免GAP锁
## 3 记录锁（LOCK_REC_NOT_GAP/行级）
锁对象只是单纯的锁在记录上，不会锁记录之前的 GAP。在 RC 隔离级别下一般加的都是该类型的记录锁
## 4 插入意向锁（LOCK_INSERT_INTENTION/行级）
INSERT INTENTION锁是GAP锁的一种，如果有多个session插入同一个GAP时，他们无需互相等待，例如当前索引上有记录4和8，两个并发session同时插入记录6，7。他们会分别为(4,8)加上GAP锁，但相互之间并不冲突（因为插入的记录不冲突）。当向某个数据页中插入一条记录时，总是会进行锁检查（构建索引时的数据插入除外），会去检查当前插入位置的下一条记录上是否存在锁对象，这里的下一条记录不是指的物理连续，而是按照逻辑顺序的下一条记录。 如果下一条记录上不存在锁对象：若记录是二级索引上的，先更新二级索引页上的最大事务ID为当前事务的ID；直接返回成功。如果下一条记录上存在锁对象，就需要判断该锁对象是否锁住了GAP。如果GAP被锁住了，并判定和插入意向GAP锁冲突，当前操作就需要等待，加的锁类型为LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION，并进入等待状态。但是插入意向锁之间并不互斥
## 5 排他锁（LOCK_X/行级)
允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁
### 5.1 执行SQL（通过二级索引查询）
RC隔离级别：首先锁住二级索引记录，为NOT GAP X锁；然后锁住对应的聚集索引记录，也是NOT GAP X锁<br>
RR隔离级别下：首先锁住二级索引记录，为LOCK_ORDINARY|LOCK_X锁；然后锁住聚集索引记录，为NOT GAP X锁
### 5.2 执行SQL（通过聚集索引检索，更新二级索引数据）
对聚集索引记录加 LOCK_REC_NOT_GAP | LOCK_X锁;<br>
在标记删除二级索引时，检查二级索引记录上的锁，如果存在和LOCK_X | LOCK_REC_NOT_GAP冲突的锁对象，则创建锁对象并返回等待错误码；否则无需创建锁对象；
## 6 意向 共享锁/排他锁（LOCK_IS/LOCK_IX/表级）
事务打算给数据行加行共享锁/行排他锁，事务在给一个数据行加共享锁前必须先取得该表的IS/IX锁
## 7 共享锁（LOCK_S/行级）
共享锁的作用通常用于在事务中读取一条行记录后，不希望它被别的事务锁修改，但所有的读请求产生的LOCK_S锁是不冲突的。
### 7.1 普通查询在隔离级别为 SERIALIZABLE 
会给记录加 LOCK_S 锁
### 7.2 类似 SQL SELECT … IN SHARE MODE
基于不同的隔离级别，行为有所不同:<br>
RC隔离级别： LOCK_REC_NOT_GAP | LOCK_S；<br>
RR隔离级别：如果查询条件为唯一索引且是唯一等值查询时，加的是 LOCK_REC_NOT_GAP | LOCK_S； <br>
对于非唯一条件查询，或者查询会扫描到多条记录时，加的是LOCK_ORDINARY | LOCK_S锁，
### 7.3 insert duplicate key
通常INSERT操作是不加锁的，但如果在插入或更新记录时，检查到 duplicate key（或者有一个被标记删除的duplicate key），对于普通的INSERT/UPDATE，会加LOCK_S锁
## 8 LOCK_ORDINARY(Next-Key Lock/行级)
MySQL 默认情况下使用RR的隔离级别，而NEXT-KEY LOCK正是为了解决RR隔离级别下的幻读问题。所谓幻读就是一个事务内执行相同的查询，会看到不同的行记录。
在RR隔离级别下这是不允许的。