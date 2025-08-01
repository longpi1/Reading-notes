# PostgreSQL与MySQL实现隔离机制的区别

### PostgreSQL 的隔离级别

PostgreSQL 实现了 SQL 标准中定义的四种隔离级别，并采用了一种名为 **MVCC (Multi-Version Concurrency Control, 多版本并发控制)** 的核心机制来实现它们。

#### 默认隔离级别

PostgreSQL 的**默认隔离级别是 `Read Committed` (读已提交)**。

这是在性能和数据一致性之间的一个非常实用的折衷。它保证了一个事务只能读到其他已经提交的事务所做的更改，避免了“脏读”，但可能会遇到“不可重复读”和“幻读”。

------

### MVCC：PostgreSQL 并发控制的基石

要理解隔离级别的实现，必须先理解 MVCC。

- **核心思想**：对数据的任何修改（`UPDATE`, `DELETE`）都不会直接在原地覆盖旧数据，而是会创建一个**新的数据版本**（称为元组 "tuple"）。

- 版本标识

  ：每个数据行（元组）内部都有两个隐藏的系统列：

  - `xmin`：创建该行版本的事务 ID (Transaction ID, XID)。
  - `xmax`：删除或更新该行版本的事务 ID。如果该行版本未被删除/更新，`xmax` 为 0 或无效。

- **事务快照 (Transaction Snapshot)**：当一个事务开始时（或在特定语句开始时），它会获得一个“快照”。这个快照记录了在这一瞬间，哪些事务 ID 是已经提交的、哪些是未提交的、哪些是正在进行的。

- 可见性规则

  ：当一个事务需要读取数据时，它会遍历数据行（的多个版本），并根据其事务快照应用可见性规则来决定哪个版本对它来说是“可见”的。

  - 一个行版本对当前事务可见，必须满足：
    1. 该版本的 `xmin` 对应的事务已经**提交**，且早于当前事务的快照。
    2. 该版本的 `xmax` 为空，或者其对应的事务**未提交**，或者晚于当前事务的快照（意味着在本事务看来，这个删除/更新“尚未发生”）。

**简单来说，MVCC 使得“读”和“写”操作不会相互阻塞**。写操作创建新版本，而读操作根据自己的快照去寻找合适的旧版本。

------

### 各隔离级别的实现原理

#### 1. Read Committed (读已提交) - 默认级别

- **实现原理**：**在事务中的每一条语句开始执行时，都会创建一个新的事务快照**。
- 行为表现：
  - **避免脏读**：因为快照只会包含已提交的事务，所以永远不会读到其他未提交事务的数据。
  - **允许不可重复读 (Non-Repeatable Read)**：在一个事务内，两条相同的 `SELECT` 查询之间，如果另一个事务提交了对相关数据的修改，那么第二条 `SELECT` 会获取一个新的快照，从而读到这个已提交的新数据，导致两次读取结果不一致。
  - **允许幻读 (Phantom Read)**：同理，两次查询之间，另一个事务插入新数据并提交，第二条查询的新快照会包含这条新纪录，导致“幻影”行的出现。

#### 2. Repeatable Read (可重复读)

- **实现原理**：**在事务中的第一条数据操作语句执行时，创建一个事务快照，并在整个事务的生命周期内，始终使用这同一个快照**。
- 行为表现：
  - **避免不可重复读**：因为整个事务都使用同一个旧的快照，所以即使其他事务提交了修改，本事务也“看不见”，保证了多次读取同一行数据的结果是一致的。
  - **避免幻读**：同理，也看不见其他事务提交的新增行。
  - **可能发生序列化异常 (Serialization Anomaly)**：虽然避免了幻读，但可能会遇到更微妙的冲突。例如，事务A读取数据，事务B修改了该数据并提交。当事务A稍后也想修改该数据时，它会发现数据已经被一个在它快照之后的事务修改了。为了保证数据一致性，PostgreSQL 会**中止事务A**，并报错：`ERROR: could not serialize access due to concurrent update`。这是一种**乐观锁**的策略：先假设不会冲突，如果冲突了就让其中一个事务失败回滚。

#### 3. Serializable (可串行化) - 最高级别

- 实现原理

  ：这是 PostgreSQL 的一个亮点，它使用了一种名为 SSI (Serializable Snapshot Isolation, 可串行化快照隔离)的先进技术。

  - 它基于 `Repeatable Read` 的单一快照机制。
  - **增加了“谓词锁”或依赖关系跟踪**：当一个事务读取数据时，PostgreSQL 不仅返回数据，还会记录下一个“读写依赖”。例如，事务A读取了 `WHERE balance > 1000` 的数据，系统会记录下这个条件（谓词）。
  - 如果此时事务B插入或更新了一行数据，使其**满足了**事务A的 `WHERE` 条件，系统就会检测到这种依赖冲突。
  - 当其中一个事务尝试提交时，PostgreSQL 会检查是否存在会导致违反串行化执行结果的依赖关系。如果存在，它会遵循“**先提交者获胜**”的原则，让后提交的事务失败回滚，并报错 `ERROR: could not serialize access due to read/write dependencies`。
  - SSI 同样是一种乐观并发控制，它比传统的悲观锁（在读取时就加锁）提供了更高的并发性。

#### 4. Read Uncommitted (读未提交)

- **实现原理**：在 PostgreSQL 中，**`Read Uncommitted` 的行为和 `Read Committed` 完全一样**。
- **原因**：PostgreSQL 的 MVCC 架构从根本上就不支持“脏读”。让一个事务去看到另一个未提交事务正在创建的行版本，需要重大的架构改动且会带来复杂性。因此，PostgreSQL 选择将此级别等同于 `Read Committed` 处理。

------

### 与 MySQL (InnoDB) 的对比

| 特性 / 方面              | PostgreSQL                                                   | MySQL (InnoDB 引擎)                                          |
| :----------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **默认隔离级别**         | **`Read Committed`** (读已提交)                              | **`Repeatable Read`** (可重复读) [注1]                       |
| **MVCC 实现**            | 行版本直接存储在表的数据文件（堆）中，通过 `xmin` 和 `xmax` 标记。依赖 `VACUUM` 进程清理死版本。 | 通过 `undo log` 链来存储旧版本。当需要读取旧版本时，从 `undo log` 中重构出来。 |
| **Read Committed 实现**  | **每条语句一个新快照 (Snapshot)**。                          | **每条语句一个新读视图 (Read View)**。原理和行为基本一致。   |
| **Repeatable Read 实现** | **整个事务共享一个快照**。不使用间隙锁。面对并发更新时，可能因“序列化异常”而导致事务失败（乐观策略）。 | **整个事务共享一个读视图**。**核心区别**：它使用 **Next-Key Locking (间隙锁 + 行锁)** 来解决幻读问题。当查询一个范围时，它会锁定这个范围内的已有行和行之间的“间隙”，阻止其他事务在此间隙插入新数据（悲观策略）。 |
| **Serializable 实现**    | **SSI (可串行化快照隔离)**。基于快照和依赖关系图，是一种先进的乐观并发控制。 | **将所有 `SELECT` 语句隐式转换为 `SELECT ... LOCK IN SHARE MODE`**。即对所有读取的行加共享锁，强制事务串行执行，是一种悲观锁的实现，并发性较差。 |
| **幻读处理 (在RR级别)**  | **不通过锁来阻止**。允许并发插入，但在当前事务尝试修改数据时，通过检测冲突并回滚事务来保证一致性。 | **通过 Next-Key Locks (间隙锁) 来主动阻止**。其他事务无法在被锁定的间隙中插入数据，因此幻读现象不会发生。 |
| **总结**                 | 采用更现代的乐观并发控制（SSI），在 `Serializable` 级别提供高并发性。RR 级别可能因乐观锁失败而需要应用重试。 | 在 `Repeatable Read` 级别通过悲观锁（间隙锁）提前解决了幻读问题，更易于理解，但可能因锁竞争降低并发度。 |

**[注1]**：虽然 MySQL 的默认值历来是 `Repeatable Read`，但一些云服务商（如 AWS RDS）可能会将其默认配置修改为 `Read Committed` 以获得更高的并发性。

### 核心差异总结

1. **默认级别不同**：PG 默认 `Read Committed`，更注重并发；MySQL 默认 `Repeatable Read`，更注重一致性。

2. RR 级别的幻读解决方案不同

   ：这是两者最核心的区别。

   - **MySQL (InnoDB)**：用**间隙锁（Gap Lock）**，是一种**悲观锁**，提前阻止幻读的发生。
   - **PostgreSQL**：不用间隙锁，是一种**乐观方案**，允许并发写入，但如果这种写入与当前事务产生冲突，会在当前事务提交或修改时使其失败。

3. Serializable 级别实现不同：

   - **PostgreSQL**：用先进的 **SSI**，是乐观的，并发性高。
   - **MySQL (InnoDB)**：用**悲观锁**，将读操作也加上锁，并发性低。