## MySQL 索引

在 MySQL 中，索引是在存储引擎层实现的，所以并没有统一的标准。



### 索引的常见模型

- 哈希表，使用链表解决冲突问题，适用于只有等值查询的场景。
- 有序数组，在等值查询和范围查询场景中的性能都非常优秀，更新效率低，适用于静态存储引擎。
- 二叉搜索树，查询时间复杂度 O(log(N))，更新时间复杂度 O(log(N))。数据库存储大多不适用二叉树，因为树高过高，增大了查询时间，一般选择 N 叉树。



### InnoDB 的索引模型

InnoDB 使用了 B+ 树索引模型，索引类型分为主键索引和非主键索引。

- 主键索引的叶子节点存的是整行数据，也被称为聚簇索引（clustered index）。
- 非主键索引的叶子节点内容是主键的值，也被称为二级索引（secondary index）。

使用普通索引时，先搜索索引拿到主键值，再到主键索引搜索获取行数据（回表），因此，建议尽量使用主键索引。

一个数据页满了，按照 B+ 树算法会新增加一个数据页，叫做页分裂。空间利用率降低大概 50%，导致性能下降。当相邻的两个数据页利用率很低的时候会做数据页合并。

- 索引可能因为删除，或者页分裂等原因，导致数据页有空洞，重建索引的过程会创建一个新的索引，把数据按顺序插入，这样页面的利用率最高，也就是索引更紧凑、更省空间。**但是不论是删除主键还是重新创建主键，都会将整个表重建**。可以使用 `alter table T engine = InnoDB` 代替。

自增主键的插入数据模式，每次插入一条新记录都是追加操作，不涉及到挪动其他记录，也不会触发叶子节点的分裂。从性能和存储空间方面考量，自增主键往往是更合理的选择。



#### 覆盖索引

某索引已经覆盖了查询需求，称为覆盖索引。覆盖索引可以减少树的搜索次数，显著提升查询性能。例如将身份证号和姓名建立联合索引，在对高频的根据身份证号查询姓名的请求时，不会回表查询整行记录。当然，索引字段的维护总是有代价的。因此，在建立冗余索引来支持覆盖索引时就需要权衡考虑了。



#### 最左前缀原则

B+ 树这种索引结构，可以利用索引的 ”最左前缀“ 来定位记录，只要满足最左前缀，就可以利用索引来加速检索。最左前缀可以是联合索引的最左 N 个字段，也可以是字符串索引的最左 M 个字符。

由于最左前缀原则，在联合索引中，索引字段顺序的第一原则是：如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。

例如存在一张表 t，表中有三个字段 a，b，c，其中 a b 是联合主键，c ca cb 是已建立的索引，业务里面有这样的两种语句：

```mysql
select * from t where c = N order by a limit 1;
select * from t where c = N order by b limit 1;
```

主键 ab 的聚簇索引组织顺序相当于先按 a 排序，再按 b 排序，c 无序；

索引 ca 的组织是先按 c 排序，再按 a 排序，同时记录主键，这跟索引 c 的顺序是一样的（先按 c 排序，再记录主键）；

索引 cb 的组织是先按 c 排序，再按 b 排序，同时记录主键；

所以 ca 索引的存在是冗余的。



#### 索引下推

在 MySQL 5.6 之前，只能根据最左前缀查询到 ID 后，再开始一个个回表，到主键索引上找出数据行，再对比字段值。MySQL 5.6 引入的索引下推优化（index condition pushdown)，可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。



### 索引的选择

#### 普通索引和唯一索引

##### 查询过程

`select id from T where k = 5`：

- 对于普通索引来说，查找到满足条件的第一个记录后，需要查找下一个记录，直到碰到第一个不满足 k = 5 条件的记录。
- 对于唯一索引来说，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索。

InnoDB 的数据是按数据页为单位来读写的。也就是说，当需要读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是以页为单位，将其整体读入内存，每个数据页的大小默认是 16 KB。对于整型字段，一个数据页可以放近千个 key。因此这两种查询方式在性能上的差距微乎其微。



##### 更新过程

当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InnoDB 会将这些更新操作缓存在 change buffer 中，在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证数据逻辑的正确性。

对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束，而这必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 change buffer 了。

如果要在一张表中插入一个新记录（4, 400），两种索引下的处理流程：

- 第一种情况是，**这个记录要更新的目标页在内存中：**

  - 对于唯一索引来说，找到 3 和 5 之间的位置，判断没有冲突，插入这个值，语句执行结束。
  - 对于普通索引来说，找到 3 和 5 之间的位置，插入这个值，语句执行结束。
  
  二者性能差别很小。
  
- 第二种情况是，**这个记录要更新的目标页不在内存中：**

  - 对于唯一索引来说，需要将数据页读入内存，判断到没有冲突，插入这个值，语句执行结束。
  - 对于普通索引来说，则是将更新记录在 change buffer 中，语句执行就结束了。

  将数据从磁盘读入内存涉及随机 IO 的访问，是数据库里面成本最高的操作之一。change buffer 因为减少了随机磁盘访问，所以对更新性能的提升是会很明显的。

**change buffer：**

- 当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InnoDB 会将这些更新操作缓存在 change buffer 中，在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证数据逻辑的正确性。
- change buffer 是可以持久化的数据，在内存中有拷贝，也会被写入到磁盘上。
- 将 change buffer 中的操作应用到原数据页，得到最新结果的过程称为 merge。除了访问这个数据页会触发 merge 外，系统有后台线程会定期 merge。在数据库正常关闭的过程中，也会执行 merge 操作。
- change buffer 用的是 buffer pool 里的内存，因此不能无限增大。可以通过参数 innodb_change_buffer_max_size 来动态设置大小，这个参数设置为 50 的时候，表示 change buffer 的大小最多只能占用 buffer pool 的 50%。
- 在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。因此，对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时 change buffer 的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。反过来，假设一个业务的更新模式是写入之后马上会做查询，那么即使将更新先记录在 change buffer，但之后由于马上要访问这个数据页，会立即触发 merge 过程。这样随机访问 IO 的次数不会减少，反而增加了 change buffer 的维护代价。

**change buffer 和 redo log：**

如果读语句发生在更新语句后不久，内存中的数据都还在，那么将直接从内存返回结果。倘若内存中没有，则将数据页从磁盘读入内存中，执行 change buffer 中与这个页有关的操作，再返回结果。

WAL 之后如果读数据，不一定需要读盘，也不一定要从 redo log 里面把数据更新以后才可以返回。

**redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写），而 change buffer 主要节省的则是随机读磁盘的 IO 消耗。**
