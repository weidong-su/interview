> https://xiaolincoding.com/mysql/transaction/mvcc.html

# 事务四大特性（ACID）原子性、一致性、隔离性、持久性？

原子性（Atomicity）：一个事务中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节，而且事务在执行过程中发生错误，会被回滚到事务开始前的状态，就像这个事务从来没有执行过一样，就好比买一件商品，购买成功时，则给商家付了钱，商品到手；购买失败时，则商品在商家手中，消费者的钱也没花出去。
一致性（Consistency）：是指事务操作前和操作后，数据满足完整性约束，数据库保持一致性状态。比如，用户 A 和用户 B 在银行分别有 800 元和 600 元，总共 1400 元，用户 A 给用户 B 转账 200 元，分为两个步骤，从 A 的账户扣除 200 元和对 B 的账户增加 200 元。一致性就是要求上述步骤操作后，最后的结果是用户 A 还有 600 元，用户 B 有 800 元，总共 1400 元，而不会出现用户 A 扣除了 200 元，但用户 B 未增加的情况（该情况，用户 A 和 B 均为 600 元，总共 1200 元）。
隔离性（Isolation）：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致，因为多个事务同时使用相同的数据时，不会相互干扰，每个事务都有一个完整的数据空间，对其他并发事务是隔离的。也就是说，消费者购买商品这个事务，是不影响其他消费者购买的。
持久性（Durability）：事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。
InnoDB 引擎通过什么技术来保证事务的这四个特性的呢？

持久性是通过 redo log （重做日志）来保证的；
原子性是通过 undo log（回滚日志） 来保证的；
隔离性是通过 MVCC（多版本并发控制） 或锁机制来保证的；
一致性则是通过持久性+原子性+隔离性来保证；


# 索引优化办法

MySQL索引对于提高查询性能至关重要，但是如果不正确使用或管理，它们也可能成为性能瓶颈。以下是一些关于MySQL索引优化的方法和建议：

1. 选择合适的索引列：
+ 选择经常出现在WHERE子句、JOIN条件或ORDER BY子句中的列作为索引列。
+ 选择具有高选择性的列（即列中的不同值很多）作为索引列，因为这样的索引更能有效减少数据扫描的范围。

2. 使用复合索引：
+ 如果查询经常同时涉及多个列，考虑创建复合索引，而不是单独为每个列创建索引。
+ 注意复合索引的列顺序，因为MySQL使用最左前缀原则来匹配索引。

3. 避免全表扫描：
通过创建合适的索引，尽量避免全表扫描，特别是在处理大量数据时。

4.删除不必要的索引：
定期审查并删除不再需要或很少使用的索引，以减少写操作的开销和存储空间的占用。

5.使用EXPLAIN分析查询：
使用EXPLAIN关键字分析查询，查看MySQL是如何使用索引的，以及是否有优化空间。

6. 考虑使用覆盖索引：
如果一个索引包含了查询中所需的所有列，那么这个索引被称为覆盖索引。MySQL可以直接从索引中获取数据，而无需回表查询，从而提高查询性能。

7. 保持索引的更新：
当表中的数据发生大量变化时（如插入、删除或更新操作），索引可能会变得碎片化。定期使用OPTIMIZE TABLE命令来重新组织表和索引，提高查询性能。

8. 避免使用NULL值：
尽量避免在索引列中使用NULL值，因为MySQL在处理NULL值时可能不够高效。如果确实需要存储NULL值，可以考虑使用其他方法来表示缺失或默认值。

9.监控索引的使用情况：
使用MySQL的性能监控工具来跟踪索引的使用情况，以便及时发现并解决性能问题。

10. 考虑使用其他类型的索引：
除了常见的B-Tree索引外，MySQL还支持其他类型的索引，如哈希索引、空间索引和全文索引。根据实际需求选择合适的索引类型。
