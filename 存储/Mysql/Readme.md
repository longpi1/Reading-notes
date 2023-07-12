# 图解MySQL介绍

> 主要内容转载自[小林coding](https://xiaolincoding.com/)，以后目录可直接跳转到对应文章；

重点突击 MySQL 索引、事务、锁、日志等面试常问知识。

- **基础篇**👇
  - [执行一条 SQL 查询语句，期间发生了什么？](https://xiaolincoding.com/mysql/base/how_select.html)
  - [MySQL 一行记录是怎么存储的？](https://xiaolincoding.com/mysql/base/row_format.html)
- **索引篇** 👇
  - [索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)
  - [从数据页的角度看 B+ 树](https://xiaolincoding.com/mysql/index/page.html)
  - [为什么 MySQL 采用 B+ 树作为索引？](https://xiaolincoding.com/mysql/index/why_index_chose_bpuls_tree.html)
  - [MySQL 单表不要超过 2000W 行，靠谱吗？](https://xiaolincoding.com/mysql/index/2000w.html)
  - [索引失效有哪些？](https://xiaolincoding.com/mysql/index/index_lose.html)
  - [MySQL 使用 like “%x“，索引一定会失效吗？](https://xiaolincoding.com/mysql/index/index_issue.html)
  - [count(*) 和 count(1) 有什么区别？哪个性能最好？](https://xiaolincoding.com/mysql/index/count.html)
- **事务篇** 👇
  - [事务隔离级别是怎么实现的？](https://xiaolincoding.com/mysql/transaction/mvcc.html)
  - [MySQL 可重复读隔离级别，完全解决幻读了吗？](https://xiaolincoding.com/mysql/transaction/phantom.html)
- **锁篇** 👇
  - [MySQL 有哪些锁？](https://xiaolincoding.com/mysql/lock/mysql_lock.html)
  - [MySQL 是怎么加锁的？](https://xiaolincoding.com/mysql/lock/how_to_lock.html)
  - [update 没加索引会锁全表?](https://xiaolincoding.com/mysql/lock/update_index.html)
  - [MySQL 记录锁+间隙锁可以防止删除操作而导致的幻读吗？](https://xiaolincoding.com/mysql/lock/lock_phantom.html)
  - [MySQL 死锁了，怎么办？](https://xiaolincoding.com/mysql/lock/deadlock.html)
  - [字节面试：加了什么锁，导致死锁的？](https://xiaolincoding.com/mysql/lock/show_lock.html)
- **日志篇** 👇
  - [undo log、redo log、binlog 有什么用？](https://xiaolincoding.com/mysql/log/how_update.html)
- **内存篇** 👇
  - [揭开 Buffer_Pool 的面纱](https://xiaolincoding.com/mysql/buffer_pool/buffer_pool.html)