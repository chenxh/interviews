https://www.cnblogs.com/jelly12345/p/14889331.html

### 什么是MVCC?
MVCC，全称Multi-Version Concurrency Control，即多版本并发控制。MVCC是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问，在编程语言中实现事务内存。
MVCC在MySQL InnoDB中的实现主要是为了提高数据库并发性能，用更好的方式去处理读-写冲突，做到即使有读写冲突时，也能做到不加锁，非阻塞并发读.


### MVCC带来的好处是？
多版本并发控制（MVCC）是一种用来解决读-写冲突的无锁并发控制，也就是为事务分配单向增长的时间戳，为每个修改保存一个版本，版本与事务时间戳关联，读操作只读该事务开始前的数据库的快照。 所以MVCC可以为数据库解决以下问题

* 在并发读写数据库时，可以做到在读操作时不用阻塞写操作，写操作也不用阻塞读操作，提高了数据库并发读写的性能
* 同时还可以解决脏读，幻读，不可重复读等事务隔离问题，但不能解决更新丢失问题


MVCC + 悲观锁：MVCC解决读写冲突，悲观锁解决写写冲突
MVCC + 乐观锁：MVCC解决读写冲突，乐观锁解决读写冲突



### MVCC的实现原理

MVCC的目的就是多版本并发控制，在数据库中的实现，就是为了解决读写冲突，它的实现原理主要是依赖 ***记录中的 3个隐式字段，undo日志 ，Read View 来实现的*** .
