## PostgreSQL的应用场景

1. **Web应用程序开发**：PostgreSQL作为可靠的关系型数据库系统，广泛用于Web应用程序的数据存储和管理。它适用于各种规模的Web应用，从小型博客到大型电子商务平台都可以使用PostgreSQL来存储用户数据、产品信息和交易记录等[[1](https://www.trustradius.com/products/postgresql/reviews?qs=product-usage)]。
2. **地理信息系统（GIS）**：PostgreSQL具有强大的地理信息扩展（PostGIS），使其成为地理信息系统的理想选择。通过PostGIS，可以在PostgreSQL中存储、查询和分析地理空间数据，例如地理位置、地图和空间对象等[[1](https://www.trustradius.com/products/postgresql/reviews?qs=product-usage)]。
3. **数据分析和报告**：由于其丰富的数据处理和分析功能，PostgreSQL被广泛应用于数据分析和报告领域。它可以处理大量的数据，并支持复杂的查询、聚合和统计分析操作，为用户提供准确的数据洞察[[1](https://www.trustradius.com/products/postgresql/reviews?qs=product-usage)]。
4. **微服务架构**：PostgreSQL在微服务架构中的应用越来越普遍。它可以作为每个微服务的独立数据库，支持数据的隔离和管理。由于其可靠性和性能优势，PostgreSQL成为支持微服务架构中数据存储的首选数据库之一[[1](https://www.trustradius.com/products/postgresql/reviews?qs=product-usage)]。
5. **开源项目和社区**：PostgreSQL是一个开源数据库系统，具有庞大的社区支持和活跃的开发者社区。因此，许多开源项目和社区选择使用PostgreSQL作为其数据存储解决方案。这些项目包括开源软件、社交媒体平台、内容管理系统等[[1](https://www.trustradius.com/products/postgresql/reviews?qs=product-usage)]。
6. **企业应用**：许多企业选择PostgreSQL作为其关键业务应用的后端数据库。PostgreSQL的稳定性、可靠性和可扩展性使其成为处理企业级数据和关键业务流程的理想选择。它支持高并发、高事务处理和数据安全性等企业级要求[[1](https://www.trustradius.com/products/postgresql/reviews?qs=product-usage)]。

需要根据具体的应用需求和业务场景来选择适合的数据库。PostgreSQL的灵活性、性能和可靠性使其适用于各种不同类型的应用场景。无论是开发Web应用、处理地理空间数据，还是进行数据分析和构建企业级应用，PostgreSQL都提供了强大的功能和支持[[1](https://www.trustradius.com/products/postgresql/reviews?qs=product-usage)]。



参考链接：[MySQL与PostgreSQL的对比](https://zhuanlan.zhihu.com/p/435829273)

### PostgreSQL相对于MySQL的优势

**1）支持地理信息处理扩展**

PostGIS 为PostgreSQL提供了存储空间地理数据的支持，使PostgreSQL成为了一个空间数据库，能够进行空间数据管理、数量测量与几何拓扑分析。

O2O业务场景中的LBS业务使用PostgreSQL + PostGIS有无法比拟的优势。

**2）可以快速构建REST API**

PostgREST 可以方便的为任何 PostgreSQL 数据库提供完全的 RESTful API 服务。

**3）支持树状结构**

支持R-trees这样可扩展的索引类型，可以更方便地处理一些特殊数据。MySQL 处理树状的设计会很复杂, 而且需要写很多代码, 而 PostgreSQL 可以高效处理树结构。

**4）有极其强悍的 SQL 编程能力**

支持递归，有非常丰富的统计函数和统计语法支持。

- MySQL：支持 CREATE PROCEDURE 和 CREATE FUNCTION 语句。存储过程可以用 SQL 和 C++ 编写。用户定义函数可以用 SQL、C 和 C++ 编写。
- PostgreSQL：没有单独的存储过程，都是通过函数实现的。用户定义函数可以用 PL/pgSQL（专用的过程语言）、PL/Tcl、PL/Perl、PL/Python 、SQL 和 C 编写。

**5）外部数据源支持**

可以把 70 种外部数据源 (包括 Mysql, Oracle, CSV, hadoop …) 当成自己数据库中的表来查询。Postgres有一个针对这一难题的解决方案：一个名为“外部数据封装器（Foreign Data Wrapper，FDW）”的特性。该特性最初由PostgreSQL社区领袖Dave Page四年前根据SQL标准SQL/MED（SQL Management of External Data）开发。FDW提供了一个SQL接口，用于访问远程数据存储中的远程大数据对象，使DBA可以整合来自不相关数据源的数据，将它们存入Postgres数据库中的一个公共模型。这样，DBA就可以访问和操作其它系统管理的数据，就像在本地Postgres表中一样。例如，使用FDW for MongoDB，数据库管理员可以查询来自文档数据库的数据，并使用SQL将它与来自本地Postgres表的数据相关联。借助这种方法，用户可以将数据作为行、列或JSON文档进行查看、排序和分组。他们甚至可以直接从Postgres向源文档数据库写入（插入、更细或删除）数据，就像一个一体的无缝部署。也可以对Hadoop集群或MySQL部署做同样的事。FDW使Postgres可以充当企业的中央联合数据库或“Hub”。

**6）没有字符串长度限制**

一般关系型数据库的字符串有限定长度8k左右，无限长 TEXT 类型的功能受限，只能作为外部大数据访问。而PostgreSQL的 TEXT 类型可以直接访问，SQL语法内置正则表达式，可以索引，还可以全文检索，或使用xml xpath。MySQL 的各种text字段有不同的限制，要手动区分 small text, middle text, large text… PostgreSQL 没有这个限制，text 能支持各种大小。

**7）支持图结构数据存储**

参考链接：[https://mp.weixin.qq.com/s/cjor82wgDu5gzDvTYpLDWw](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/cjor82wgDu5gzDvTYpLDWw)

**8）对索引的支持更强**

PostgreSQL 的可以使用函数和条件索引，这使得PostgreSQL数据库的调优非常灵活，mysql就没有这个功能，条件索引在web应用中很重要。对于索引类型：

- MySQL：取决于存储引擎。MyISAM：BTREE，InnoDB：BTREE。
- PostgreSQL：支持 B-树、哈希、R-树和 Gist 索引。

InnoDB的表和索引都是按相同的方式存储。也就是说表都是索引组织表。这一般要求主键不能太长而且插入时的主键最好是按顺序递增，否则对性能有很大影响。PostgreSQL不存在这个问题。

索引类型方面，MySQL取决于存储引擎。MyISAM：BTREE，InnoDB：BTREE。PostgreSQL支持 B-树、哈希、R-树和 Gist 索引。

**9）集群支持更好**

Mysql Cluster可能与你的想象有较大差异。开源的cluster软件较少。复制(Replication)功能是异步的并且有很大的局限性。例如，它是单线程的(single-threaded)，因此一个处理能力更强的Slave的恢复速度也很难跟上处理能力相对较慢的Master。

PostgreSQL有丰富的开源cluster软件支持。plproxy 可以支持语句级的镜像或分片，slony 可以进行字段级的同步设置，standby 可以构建WAL文件级或流式的读写分离集群，同步频率和集群策略调整方便，操作非常简单。

另外，PostgreSQL的主备复制属于物理复制，相对于MySQL基于binlog的逻辑复制，数据的一致性更加可靠，复制性能更高，对主机性能的影响也更小。对于WEB应用来说，复制的特性很重要，mysql到现在也是异步复制，pgsql可以做到同步，异步，半同步复制。还有mysql的同步是基于binlog复制，类似oracle golden gate,是基于stream的复制，做到同步很困难，这种方式更加适合异地复制，pgsql的复制基于wal，可以做到同步复制。同时，pgsql还提供stream复制。

**10）事务隔离做的更好**

MySQL 的事务隔离级别 repeatable read 并不能阻止常见的并发更新, 得加锁才可以, 但悲观锁会影响性能, 手动实现乐观锁又复杂. 而 PostgreSQL 的列里有隐藏的乐观锁 version 字段, 默认的 repeatable read 级别就能保证并发更新的正确性, 并且又有乐观锁的性能。

**11）对于字符支持更好一些**

MySQL 里需要 utf8mb4 才能显示 emoji 的坑, PostgreSQL 没这个坑。

**12）对表连接支持较完整**

对表连接支持较完整，MySQL只有一种表连接类型:嵌套循环连接(nested-loop),不支持排序-合并连接(sort-merge join)与散列连接(hash join)。PostgreSQL都支持。

**13）存储方式支持更大的数据量**

PostgreSQL主表采用堆表存放，MySQL采用索引组织表，能够支持比MySQL更大的数据量。

**14）时间精度更高**

MySQL对于时间、日期、间隔等时间类型没有秒以下级别的存储类型，而PostgreSQL可以精确到秒以下。

**15）优化器的功能较完整**

MySQL对复杂查询的处理较弱，查询优化器不够成熟，explain看执行计划的结果简单。性能优化工具与度量信息不足。

PostgreSQL很强大的查询优化器，支持很复杂的查询处理。explain返回丰富的信息。提供了一些性能视图，可以方便的看到发生在一个表和索引上的select、delete、update、insert统计信息，也可以看到cache命中率。网上有一个开源的pgstatspack工具。

**16)序列支持更好**

MySQL 不支持多个表从同一个序列中取 id, 而 PostgreSQL 可以。

**17）对子查询支持更好**

对子查询的支持。虽然在很多情况下在SQL语句中使用子查询效率低下，而且绝大多数情况下可以使用带条件的多表连接来替代子查询，但是子查询的存在在很多时候仍然不可避免。而且使用子查询的SQL语句与使用带条件的多表连接相比具有更高的程序可读性。几乎任何数据库的子查询 (subquery) 性能都比 MySQL 好。

**18）增加列更加简单**

MySQL表增加列，基本上是重建表和索引，会花很长时间。PostgreSQL表增加列，只是在数据字典中增加表定义，不会重建表.



### MySQL相对于PostgreSQL的优势

**1）MySQL比PostgreSQL更流行**

流行对于一个商业软件来说，也是一个很重要的指标，流行意味着更多的用户，意味着经受了更多的考验，意味着更好的商业支持、意味着更多、更完善的文档资料。易用，很容易安装。第三方工具，包括可视化工具，让用户能够很容易入门。

**2）回滚实现更优**

innodb的基于回滚段实现的MVCC机制，相对PG新老数据一起存放的基于XID的MVCC机制，是占优的。新老数据一起存放，需要定时触发VACUUM，会带来多余的IO和数据库对象加锁开销，引起数据库整体的并发能力下降。而且VACUUM清理不及时，还可能会引发数据膨胀。

**3）在Windows上运行更可靠**

与PostgreSQL相比，MySQL更适宜在Windows环境下运行。MySQL作为一个本地的Windows应用程序运行(在 NT/Win2000/WinXP下，是一个服务)，而PostgreSQL是运行在Cygwin模拟环境下。PostgreSQL在Windows下运行没有MySQL稳定，应该是可以想象的。

**4）线程模式相比进程模式的优势**

MySQL使用了线程，而PostgreSQL使用的是进程。在不同线程之间的环境转换和访问公用的存储区域显然要比在不同的进程之间要快得多。

- 进程模式对多CPU利用率比较高。进程模式共享数据需要用到共享内存，而线程模式数据本身就是在进程空间内都是共享的，不同线程访问只需要控制好线程之间的同步。
- 线程模式对资源消耗比较少。所以MySQL能支持远比PostgreSQL多的更多的连接。但PostgreSQL中有优秀的连接池软件软件，如pgbouncer和pgpool，所以通过连接池也可以支持很多的连接。

**5）权限设置上更加完善**

MySQL在权限系统上比PostgreSQL某些方面更为完善。PostgreSQL只支持对于每一个用户在一个数据库上或一个数据表上的 INSERT、SELECT和UPDATE/DELETE的授权，而MySQL允许你定义一整套的不同的数据级、表级和列级的权限。对于列级的权限， PostgreSQL可以通过建立视图，并确定视图的权限来弥补。MySQL还允许你指定基于主机的权限，这对于目前的PostgreSQL是无法实现的，但是在很多时候，这是有用的。

**6）存储引擎插件化机制**

MySQL的存储引擎插件化机制，使得它的应用场景更加广泛，比如除了innodb适合事务处理场景外，myisam适合静态数据的查询场景。

**7）适应24/7运行**

MySQL可以适应24/7运行。在绝大多数情况下，你不需要为MySQL运行任何清除程序。PostgreSQL目前仍不完全适应24/7运行，这是因为你必须每隔一段时间运行一次VACUUM。

**8）更加试用于简单的场景**

PostgreSQL只支持堆表，不支持索引组织表，Innodb只支持索引组织表。

- 索引组织表的优势：表内的数据就是按索引的方式组织，数据是有序的，如果数据都是按主键来访问，那么访问数据比较快。而堆表，按主键访问数据时，是需要先按主键索引找到数据的物理位置。
- 索引组织表的劣势：索引组织表中上再加其它的索引时，其它的索引记录的数据位置不再是物理位置，而是主键值，所以对于索引组织表来说，主键的值不能太大，否则占用的空间比较大。
- 对于索引组织表来说，如果每次在中间插入数据，可能会导致索引分裂，索引分裂会大大降低插入的性能。所以对于使用innodb来说，我们一般最好让主键是一个无意义的序列，这样插入每次都发生在最后，以避免这个问题。

由于索引组织表是按一个索引树，一般它访问数据块必须按数据块之间的关系进行访问，而不是按物理块的访问数据的，所以当做全表扫描时要比堆表慢很多，这可能在OLTP中不明显，但在数据仓库的应用中可能是一个问题。



