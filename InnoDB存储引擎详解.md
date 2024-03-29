## InnoDB存储引擎

- ### InnoDB存储引擎存储结构

  - 最小存储单元：页（16k）

    个人理解：InnoDB存储结构是一课B-tree，叶子节点即是一页（叶子页），一页里面存放着多行数据。

    ![1567843673438](C:\Users\侯泽明\AppData\Roaming\Typora\typora-user-images\1567843673438.png)

    这是叶子节点的结构

    开始往页里面存放数据：是逻辑和物理都有序的，当插入新行或者移动行的时候，物理上就不再有序，但逻辑上有序。

- ### InnoDB存储引擎架构

- ![img](https://github.com/likang315/Java-and-Middleware/raw/master/Mysql%EF%BC%8CInnoDB/InnoDB/InnoDB%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84.png?raw=true)

- 后台线程：

  - 负责刷新缓存池中的数据，保证缓存池中的内存缓存的是最新的数据
  - 将已经修改的数据文件刷新到磁盘文件，同时保证数据库在发生异常的情况下InnoDB能恢复到正常的运行状态

  1. Master Thread：将内存数据异步刷新到磁盘，InnoDB 的主要工作都是该线程完成. 该线程具有最高的优先级
  2. IO Thread：处理 IO 请求的回调
  3. Purge Thread：在事务提交的时候回收 Undo Log
  4. Page Clear Thread ：将脏页刷新操作都放入到单独的线程中来完成
     - 脏页：把数据库读出来的数据修改，还没有刷新到磁盘上

- ### Checkpoint

  事务型数据库一般采用Write Ahead Log策略，当事物提交时先写redo log而后修改内存中的页。当数据库宕机对于还未写入磁盘的修改数据可以通过redo log恢复。Checkpoint作用在于保证该点之前的所有修改的页均已刷新到磁盘，这之前的redo log在恢复数据时可以不需要了。

- Sharp Checkpoint

  发生在数据库关闭时，将所有脏页写入磁盘，数据库运行时一般不使用。

- Fuzzy Checkpoint

  只刷新部分部分脏页。

  1. Master Thread Checkpoint：Master Thread异步已一定频率刷新一定比例脏页。
  2. Flush_LRU_LIST Checkpoint：为了保证LRU中有一定数量的空闲页，Page Clear Thread将对LRU中尾端页进行移除，如果存在脏页则做刷新。
  3. Async/Sync Flush Checkpoint：为了保证redo log循环使用（覆盖），对于需要将redo文件中不可用的脏页进行刷新到磁盘。
  4. Dirty Page too much Checkpoint：脏页数量太多。

- 缓冲池：innodb_buffer_pool

  1. InnoDB 存储引擎是基于磁盘存储的，并将其中所有的记录按照页的方式进行管理. 由于CPU 和磁盘速度的鸿沟, 采用缓冲池技术来提高数据库的性能
  2. 缓存池是一块内存区域, 在 DB 读取页的时候, 首先将从磁盘读到的页存放到缓存池中, 这个过程称为将页"FIX"在缓冲池中,下次再读相同页的时候先判断是否在缓冲池中. 对于写操作，**首先修改缓冲池中的页, 再以一定的频率刷新到磁盘上**，而不是每次发生页修改时触发，通过"CheckPoint 机制"，刷新会磁盘
  3. 缓冲池的大小默认为 **256KB**， 使用LRU(最近最少使用)算法进行管理，InnoDB 对新读取的页会放在 midpoint (默认5/8长度处)
  4. 重做日志缓冲: InnoDB 首先将 Redo Log 日志放入这个缓冲区, 然后以一定的频率刷新到磁盘的 Redo Log 文件中
  5. 缓冲池是通过 **LRU算法 ** 进行管理的，即最频繁使用的页放在LRU列表的前端，而最少使用的页放在LRU列表的尾端，首先应最先释放LRU列表的尾端的页

- ### Innodb关键特性

  - 插入缓冲：提高性能
    - 当插入数据需要更新非聚集索引时，如果每次都更新则需要进行多次随机IO，因此将这些值写入缓冲对相同页的进行合并提高IO性能。
    - 插入非聚集索引时，先判断该索引页是否在缓冲池中，在则直接插入。否则写入到Insert Buffer对象。
    - 条件：二级索引，索引不能是unique(因为如果是unique则必须保证唯一性，此时得检查所有索引页，还是随机IO了)
    - Change Buffer：包括Insert Buffer、Delete Buffer、Purge Buffer，update操作包括将记录标记为已删除和真正将记录删除两个过程，对应后两个Buffer。
    - Insert Buffer内部是一颗B+树
    - Merge Insert Buffer三种情况： 
      1. 对应的索引页被读入缓冲池。
      2. 对应的索引页的可用空间小于1/32，则强制进行合并。
      3. Master Thread中的合并插入缓冲。

  - 两次写：提高数据页的可靠性
    - 在对脏页刷新到磁盘时，如果某一页还没写完就宕机，此时该页数据已经混乱无法通过redo实现恢复。innodb提供了doublewrite机制，其刷新脏页步骤如下：

  ```
  1. 先将脏页数据复制到doublewrite buffer中（2MB内存）
  2. 将doublewrite buffer分两次，每次1MB写入到doublewrite磁盘（2MB）中。
  3. 马上同步脏页数据到磁盘。对于数据混乱的页则可以从doublewrite中读取到，该页写到共享表空间。
  ```

  - 自适应哈希索引
    - InnoDB存储引擎会监控对表上索引的查找，如果观察到建立哈希索引可以带来速度的提升，则建立哈希索引，所以称之为自适应（adaptive） 的。自适应哈希索引通过缓冲池的B+树构造而来，因此建立的速度很快。而且不需要将整个表都建哈希索引，InnoDB存储引擎会自动根据访问的频率和模式 来为某些页建立哈希索引。

  - 异步IO
    - linux和windows中提供异步IO，其可以对连续的页做合并连续页的IO操作使随机IO变顺序IO。
  - 刷新邻接页
    - 刷新页时判断相邻页是否也是脏页。

- InnoDB存储表

  - InnoDB表是基于聚簇索引组成的
  - 在 InnoDB 中, 表都是根据主键顺序组织存放的, 这种存储方式成为索引组织表. 每张表都有一个主键, 如果没有显式指定, InnoDB 会使用第一个非空的唯一索引, 如果没有唯一索引, InnoDB 会自动创建一个 6 字节大小的指针作为主键
  - ![img](https://github.com/likang315/Java-and-Middleware/raw/master/Mysql%EF%BC%8CInnoDB/InnoDB/InnoDB%E5%AD%98%E5%82%A8.png?raw=true)
  - 表空间(TableSpace)：InnoDB 存储引擎逻辑存储的顶层对象,所有数据都存放在表空间. 其中包括数据, 索引, 插入缓冲, 回滚信息, 系统事务信息等
  - 段(Segment)：表空间是由多个段组成的, 常见的段有**数据段, 索引段, 回滚段等**. InnoDB 由于索引组织表的特点, 数据即索引, 索引即数据. 因此数据段是 B+ 树的叶子节点, 索引段是 B+ 树的非叶子节点
  - 区(Extent)：区是由连续页组成的空间, 在任何情况下每个区的大小都为 1MB, 为了保证区中页的连续性, InnoDB 一次从磁盘申请 4-5 个区, **默认情况下 InnoDB 页的大小为 16KB, 一个区中一共有 64 个连续的页**
  - 页(Page)：InnoDB 最小磁盘单位，默认 16KB，常见的页有: 数据页 / undo log页 / 系统页
  - 行(Row)：InnoDB 中数据按行进行存放, 每页最多允许存放 16KB / 2 - 200 = 7992 行记录

  

  - ###### InnoDB 数据页由七部分组成：

    1. File Header(文件头)：固定 38 字节,用来记录头信息, 以及相邻页的指针
    2. Page Header(页头)： 固定 56 字节, 用来记录数据页的状态信息
    3. Infinum 和 Suprenum Records：虚拟的行记录, 用来限定记录的边界
    4. User Recodes(用户记录, 即行记录)
    5. Free Space(空闲记录)
    6. Page Directory(页目录)：存放记录的相对位置, 找到 B+ 树叶⼦节点后, 再通过Page Directory 再进行二分查
    7. File Trailer(文件结尾信息)：固定 8 字节, 用于检测页是否已经完整地写入磁盘

- InnoDB对索引的支持

  - B-tree、B+tree
  - 自适应哈希表

- 数据库对象之一，是为了提高查询效率，索引其实是就一张表，该表保存了主键与索引字段，并指向数据表中的记录，也是B +树，一页就是一个B +树，索引段和数据段

  ###### 1：聚集索引（主键索引）

  - 聚集索引就是按照**每张表的主键构造⼀棵B +树的非叶子结点，**叶子节点中存放整张表的记录数据（数据页），聚集索引的特性决定了索引组织表中数据也是索引的一部分，每个数据都通过一个双向链表来进行连接
  - 一个表只能有一个聚集索引，因为一个表的物理顺序只有一种情况，并且多数情况下查询优化器都倾向于采用聚集索引，因为可以直接在其叶子节点上找到数据

  ###### 2：辅助索引

  - 辅助索引：叶子节点不包括行记录的全部数据，**叶子节点除了包含键值以外，每个叶子节点中还包含了一个指针，指向聚集索引中的主键，**然后再通过主键索引来找到一个完整的记录（存储了此索引到主键的一种映射关系）

  ###### 3：联合索引

  - 联合索引也是一棵B +树，不同的是联合索引的键值数量大于2，此外，逻辑上和单键值的B +树并没有什么区别，键值依旧是有序的，只不过这个有序是一个前提的：遵循最左前缀匹配原则

  [![img](https://github.com/likang315/Java-and-Middleware/raw/master/Mysql%EF%BC%8CInnoDB/InnoDB/%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95.png?raw=true)](https://github.com/likang315/Java-and-Middleware/blob/master/Mysql，InnoDB/InnoDB/联合索引.png?raw=true)

  - 联合索引的规则：
    - 首先会对复合索引的最左边的，也就是第一个字段的数据进行排序，在第一个字段的有序基础上，然后再对后面第二个的字段进行排序，其实就相当于实现了按名称顺序，cid这样一种排序规则
    - select * from stu，其中a = xx且b = xxx;

  ###### 4：覆盖索引

  - InnoDB支持直接**从辅助索引中查询记录并返回**（如果有的话），而不需要查询聚集索引中的记录，使用覆盖索引的好处是辅助索引不包含整行记录的所有信息，故其大小要远小于聚集索引，因此可以减少大量的IO操作
  - 当只需要查询某个字段时

  ##### 6：索引失效

  有时候在执行EXPLAIN查看SQL执行计划时，发现优化器并没有选择索引，而是执行全表扫描，这种情况多发生于联合查询JOIN连接等场景

  1. 联合索引的使用不符合最左前缀匹配原则
  2. 或条件连接的多个条件中有不走索引的字段
  3. LIKE查询前缀模糊匹配
  4. InnoDB出现隐式类型转换（varchar - > bigint），避免条件中出现隐式类型转换
  5. InnoDB评估使用全表扫描比走索引更快
  6. 在联合索引中不能有列的值为NULL，如果有，那么这一列对联合索引就是无效的，所以用特殊值代替

  ##### 7：索引的效率

  - 查找效率高：因为索引表按照索引的字段是有序排列的，是一种有序读，而不是随机读
  - 插入，删除效率低：当对表中的数据进行增加，删除和修改的时候，索引也要动态的维护，这样就降低了数据的维护速度，而且索引也是非常消耗内存的
    1. 进行大量插入时，可以先删除索引，再插入，完成后再加索引
    2. 通过插入缓冲区，插入或者更新，不是一次直接插入到索引页中，而是**先判断插入的非聚集索引页是否在缓冲池中**，如果在，直接插入，否则先放入入读插入缓冲区对象，作为占位符，然后再以⼀定的频率进行Insert Buffer和索引叶子节点的合并操作
    3. 读写分离：牛逼

  ##### 8：索引的使用策略

  1. 什么列要使用索引：
     - 表的主键，外键必须有索引
     - 经常作为查询条件在WHERE或者ORDER BY语句中出现的列要建立索引
     - 索引应该建在小字段上，对于大的文本字段甚至超长字段，不要建索引
  2. 什么列不要使用索引：
     - 经常增删改的列不要建立索引
     - 列中有大量的重复值不建立索引
     - 表记录太少不要建立索引

  ##### 9：索引的类型

  - INDEX普通索引：添加索引的字段，允许出现相同的值
  - UNIQUE唯一索引：不可以出现相同的值，可以有NULL值
  - PRIMARY KEY主键索引：不允许出现相同的值，且不能为NULL值，一个表只能有一个primary_key索引
  - 全文索引：全文索引

  ###### 创建索引

  - CREATE INDEX：只能对表增加普通索引或UNIQUE索引
  - CREATE INDEX index_name ON table_name（column_list）
  - CREATE UNIQUE INDEX index_name ON table_name（column_list）

  ###### 修改索引：

  - ALTER TABLE表名称ADD索引类型（唯一，主键，全文，索引）'索引名'（字段名，字段名）
  - ALTER TABLE'table_name'ADD INDEX'index_name'（'column_list'）;
    - 索引名，可要可不要;如果不要，该字段名就是索引名

  ###### 删除索引：

  - ALTER TABLE'table_name'DROP INDEX'index_name'
  - ALTER TABLE'table_name'DROP PRIMARY KEY;删除主键索引，注意主键索引只能用这种方式删除

  ###### 查看索引：

  - 从tablename \ G中显示索引;

  ###### 强制使用某种索引

  - FORCE INDEX（索引字段名）

  ##### 10：MySQL InnoDB表主键如何选择？

  若没有显示的定义主键，会选择从右往左，第一个不为null的字段作为主键，若没有，则存储引擎会为每一行生成6个字节的RowID，并作为主键，主键不可能无限增长，有界限，分表分库

  ###### 自增主键

  InnoDB使用聚集索引，数据记录本身被存于主索引（一颗B + Tree）的叶子节点上。这就要求同一个叶子节点内（大小为一个内存页或磁盘页）的各条数据记录按主键顺序存放，因此每当有一条新的记录插入时，MySQL会根据其主键将其插入适当的节点和位置，如果页面达到装载因子（Inno DB默认为15/16），则开辟一个新的页面（节点），如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。这样就会形成一个紧凑的索引结构，近似顺序填满，由于每次插入时也不需要移动已有数据，因此效率很高，也不会增加很多开销在维护索引上

  但是如果使用非自增主键，由于每次插入主键的值近似于随机，因此每次新记录都要被插到现有索引页得中间某个位置，此时MySQL不得不为了**将新记录插到合适位置而移动数据**，甚至目标页面可能已经被回写到磁盘上而从缓存中清掉，此时又要从磁盘上读回来，这增加了很多开销，**同时频繁的移动，分页操作造成了大量的碎片，得到了不够紧凑的索引结构**，后续不得不通过OPTIMIZE TABLE来重建表并优化填充页面

