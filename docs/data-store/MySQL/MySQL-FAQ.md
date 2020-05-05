B树和B+树 区别，索引结构

count(column) 和 count(*) 是一样的吗

这个误区甚至在很多的资深工程师或者是 DBA 中都普遍存在，很多人都会认为这是理所当然的。实际上，count(column) 和 count(*) 是一个完全不一样的操作，所代表的意义也完全不一样。

count(column) 是表示结果集中有多少个column字段不为空的记录

count(*) 是表示整个结果集有多少条记录

SQL优化，常用的索引？



数据库索引？B+树？为什么要建索引？什么样的字段需要建索引，建索引的时候一般考虑什么？索引会不会使插入、删除***作效率变低，怎么解决（分表***作）？



> 画出 MySQL 架构图，这种变态问题都能问的出来

## MySQL架构

和其它数据库相比，MySQL有点与众不同，它的架构可以在多种不同场景中应用并发挥良好作用。主要体现在存储引擎的架构上，**插件式的存储引擎架构将查询处理和其它的系统任务以及数据的存储提取相分离**。这种架构可以根据业务的需求和实际需要选择合适的存储引擎。

![MySQL逻辑架构图](https://user-gold-cdn.xitu.io/2017/10/22/436c6beeb41e0b3980c57df6d5ed4dca?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



- 最上层是一些客户端和连接服务。**主要完成一些类似于连接处理、授权认证、及相关的安全方案**。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于SSL的安全链接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

- 第二层服务层，主要完成大部分的核心服务功能， 包括查询解析、分析、优化、缓存、以及所有的内置函数，所有跨存储引擎的功能也都在这一层实现，包括触发器、存储过程、视图等 

- 第三层存储引擎层，存储引擎真正的负责了MySQL中数据的存储和提取，服务器通过API与存储引擎进行通信。不同的存储引擎具有的功能不同，这样我们可以根据自己的实际需要进行选取。

- 第四层为数据存储层，主要是将数据存储在运行于该设备的文件系统之上，并完成与存储引擎的交互。



mysql的查询流程大致是：

1. mysql客户端通过协议与mysql服务器建连接，发送查询语句，先检查查询缓存，如果命中，直接返回结果，否则进行语句解析
2. 有一系列预处理，比如检查语句是否写正确了，然后是查询优化（比如是否使用索引扫描，如果是一个不可能的条件，则提前终止），生成查询计划，然后查询引擎启动，开始执行查询，从底层存储引擎调用API获取数据，最后返回给客户端。怎么存数据、怎么取数据，都与存储引擎有关。
3. 然后，mysql默认使用的BTREE索引，并且一个大方向是，无论怎么折腾sql，至少在目前来说，mysql最多只用到表中的一个索引。

客户端请求->连接器（验证用户身份，给予权限） -> 查询缓存（存在缓存则直接返回，不存在则执行后续操作）->分析器（对SQL进行词法分析和语法分析操作） -> 优化器（主要对执行的sql优化选择最优的执行方案方法） -> 执行器（执行时会先看用户是否有执行权限，有才去使用这个引擎提供的接口）->去引擎层获取数据返回（如果开启查询缓存则会缓存查询结果）



## 存储引擎

存储引擎是MySQL的组件，用于处理不同表类型的SQL操作。不同的存储引擎提供不同的存储机制、索引技巧、锁定水平等功能，使用不同的存储引擎，还可以获得特定的功能。

使用哪一种引擎可以灵活选择，**<font color=red>一个数据库中多个表可以使用不同引擎以满足各种性能和实际需求</font>**，使用合适的存储引擎，将会提高整个数据库的性能 。

 MySQL服务器使用可插拔的存储引擎体系结构，可以从运行中的MySQL服务器加载或卸载存储引擎 。



### 存储引擎对比

InnoDB是MySQL5.7 默认的存储引擎

1. InnoDB 支持事务，MyISAM 不支持事务。这是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一；

2. InnoDB 支持外键，而 MyISAM 不支持。对一个包含外键的 InnoDB 表转为 MYISAM 会失败；  

3. InnoDB 是聚集索引，MyISAM 是非聚集索引。聚簇索引的文件存放在主键索引的叶子节点上，因此 InnoDB 必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。而 MyISAM 是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。 

4. InnoDB 不保存表的具体行数，执行 select count(*) from table 时需要全表扫描。而MyISAM 用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快；    

5. InnoDB 最小的锁粒度是行锁，MyISAM 最小的锁粒度是表锁。一个更新语句会锁住整张表，导致其他查询和更新都会被阻塞，因此并发访问受限。这也是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一；

| 对比项   | MyISAM                                                   | InnoDB                                                       |
| -------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| 主外键   | 不支持                                                   | 支持                                                         |
| 事务     | 不支持                                                   | 支持                                                         |
| 行表锁   | 表锁，即使操作一条记录也会锁住整个表，不适合高并发的操作 | 行锁,操作时只锁某一行，不对其它行有影响，<br/>适合高并发的操作 |
| 缓存     | 只缓存索引，不缓存真实数据                               | 不仅缓存索引还要缓存真实数据，对内存要求较高，而且内存大小对性能有决定性的影响 |
| 表空间   | 小                                                       | 大                                                           |
| 关注点   | 性能                                                     | 事务                                                         |
| 默认安装 | 是                                                       | 是                                                           |

 

> 一张表，里面有ID自增主键，当insert了17条记录之后，删除了第15,16,17条记录，再把Mysql重启，再insert一条记录，这条记录的ID是18还是15 ？

如果表的类型是MyISAM，那么是18。因为MyISAM表会把自增主键的最大ID 记录到数据文件中，重启MySQL自增主键的最大ID也不会丢失；

如果表的类型是InnoDB，那么是15。因为InnoDB 表只是把自增主键的最大ID记录到内存中，所以重启数据库或对表进行OPTION操作，都会导致最大ID丢失。



## 数据类型

> CHAR和VARCHAR的区别？

- CHAR和VARCHAR类型在存储和检索方面有所不同
- CHAR列长度固定为创建表时声明的长度，长度值范围是1到255

当CHAR值被存储时，它们被用空格填充到特定长度，检索CHAR值时需删除尾随空格。

> 列的字符串类型可以是什么？

字符串类型是：

- SET
- BLOB
- ENUM
- CHAR
- TEXT
- VARCHAR

> BLOB和TEXT有什么区别？

BLOB是一个二进制对象，可以容纳可变数量的数据。有四种类型的BLOB -

- TINYBLOB
- BLOB
- MEDIUMBLOB和
- LONGBLOB

它们只能在所能容纳价值的最大长度上有所不同。

TEXT是一个不区分大小写的BLOB。四种TEXT类型

- TINYTEXT
- TEXT
- MEDIUMTEXT和
- LONGTEXT

它们对应于四种BLOB类型，并具有相同的最大长度和存储要求。

BLOB和TEXT类型之间的唯一区别在于对BLOB值进行排序和比较时区分大小写，对TEXT值不区分大小写。

> mysql里记录货币用什么字段类型好

NUMERIC和DECIMAL类型被Mysql实现为同样的类型，这在SQL92标准允许。他们被用于保存值，该值的准确精度是极其重要的值，例如与金钱有关的数据。当声明一个类是这些类型之一时，精度和规模的能被(并且通常是)指定。

例如：

salary DECIMAL(9,2)

在这个例子中，9(precision)代表将被用于存储值的总的小数位数，而2(scale)代表将被用于存储小数点后的位数。

因此，在这种情况下，能被存储在salary列中的值的范围是从-9999999.99到9999999.99。

## MySQL索引

- MYSQL官方对索引的定义为：索引（Index）是帮助MySQL高效获取数据的数据结构，所以说**索引的本质是：数据结构**

- 索引的目的在于提高查询效率，可以类比字典、 火车站的车次表、图书的目录等 。

- 可以简单的理解为“排好序的快速查找数据结构”，数据本身之外，<font color=#FF0000>**数据库还维护者一个满足特定查找算法的数据结构**</font>，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据结构上实现高级查找算法。这种数据结构，就是索引。下图是一种可能的索引方式示例。

  

  ![1.png](https://i.loli.net/2019/11/13/czD8xJ2AYrFpaRP.png)

  左边的数据表，一共有两列七条记录，最左边的是数据记录的物理地址

  为了加快Col2的查找，可以维护一个右边所示的二叉查找树，每个节点分别包含索引键值，和一个指向对应数据记录物理地址的指针，这样就可以运用二叉查找在一定的复杂度内获取到对应的数据，从而快速检索出符合条件的记录。

- 索引本身也很大，不可能全部存储在内存中，**一般以索引文件的形式存储在磁盘上**

- 平常说的索引，没有特别指明的话，就是B+树（多路搜索树，不一定是二叉树）结构组织的索引。其中聚集索引，次要索引，覆盖索引，符合索引，前缀索引，唯一索引默认都是使用B+树索引，统称索引。此外还有哈希索引等。

  

### 优势

- **提高数据检索效率，降低数据库IO成本**

- **降低数据排序的成本，降低CPU的消耗**

  

### 劣势

- 索引也是一张表，保存了主键和索引字段，并指向实体表的记录，所以也需要占用内存
- 虽然索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT、UPDATE和DELETE。
  因为更新表时，MySQL不仅要保存数据，还要保存一下索引文件每次更新添加了索引列的字段，
  都会调整因为更新所带来的键值变化后的索引信息



### 索引分类

- **单值索引**：一个索引只包含单个列，一个表可以有多个单列索引
- **唯一索引**：索引列的值必须唯一，但允许有空值
- **复合索引**：一个索引包含多个列



### 哪些情况需要创建索引

1. 主键自动建立唯一索引

2. 频繁作为查询条件的字段

3. 查询中与其他表关联的字段，外键关系建立索引

4. 单键/组合索引的选择问题，who?高并发下倾向创建组合索引

5. 查询中排序的字段，排序字段通过索引访问大幅提高排序速度

6. 查询中统计或分组字段

   

### 哪些情况不要创建索引

1. 表记录太少
2. 经常增删改的表
3. 数据重复且分布均匀的表字段，只应该为最经常查询和最经常排序的数据列建立索引（如果某个数据类包含太多的重复数据，建立索引没有太大意义）
4. 频繁更新的字段不适合创建索引（会加重IO负担）
5. where条件里用不到的字段不创建索引



### MySQL索引结构

**首先要明白索引（index）是在存储引擎（storage engine）层面实现的，而不是server层面**。不是所有的存储引擎都支持所有的索引类型。即使多个存储引擎支持某一索引类型，它们的实现和行为也可能有所差别。 



#### BTree索引

- MyISAM存储引擎，使用B+Tree的数据结构，它相对与BTree结构，所有的数据都存放在叶子节点上，且把叶子节点通过指针连接到一起，形成了一条数据链表，以加快相邻数据的检索效率 

- **<mark>MyISAM引擎索引结构的叶子节点的数据域，存放的并不是实际的数据记录，而是数据记录的地址</mark>**。索引文件与数据文件分离，这样的索引称为***"<mark>非聚簇索引</mark>"***。MyISAM的主索引与辅助索引区别并不大，只是主键索引不能有重复的关键字。 

- **<mark>InnoDB引擎索引结构的叶子节点的数据域，存放的就是实际的数据记录</mark>**（对于主索引，此处会存放表中所有的数据记录；对于辅助索引此处会引用主键，检索的时候通过主键到主键索引中找到对应数据行），或者说，InnoDB的数据文件本身就是主键索引文件，这样的索引被称为***“<mark>聚簇索引</mark>”***，**一个表只能有一个聚簇索引**

- 检索原理

  ![bTree](H:/Technical-Learning/docs/_images/mysql/bTree.png)

  【初始化介绍】 
  上图是一颗b+树，浅蓝色的块我们称之为一个磁盘块，可以看到每个磁盘块包含几个数据项（深蓝色所示）和指针（黄色所示），如磁盘块1包含数据项17和35，包含指针P1、P2、P3，
  P1表示小于17的磁盘块，P2表示在17和35之间的磁盘块，P3表示大于35的磁盘块。
  真实的数据存在于叶子节点即3、5、9、10、13、15、28、29、36、60、75、79、90、99。
  非叶子节点只不存储真实的数据，只存储指引搜索方向的数据项，如17、35并不真实存在于数据表中。

【查找过程】
  如果要查找数据项29，那么首先会把磁盘块1由磁盘加载到内存，此时发生一次IO，在内存中用二分查找确定29在17和35之间，锁定磁盘块1的P2指针，内存时间因为非常短（相比磁盘的IO）可以忽略不计，通过磁盘块1的P2指针的磁盘地址把磁盘块3由磁盘加载到内存，发生第二次IO，29在26和30之间，锁定磁盘块3的P2指针，通过指针加载磁盘块8到内存，发生第三次IO，同时内存中做二分查找找到29，结束查询，总计三次IO。

真实的情况是，3层的b+树可以表示上百万的数据，如果上百万的数据查找只需要三次IO，性能提高将是巨大的，如果没有索引，每个数据项都要发生一次IO，那么总共需要百万次的IO，显然成本非常非常高。

  #### b+树性质

    1. 通过上面的分析，我们知道IO次数取决于b+数的高度h，假设当前数据表的数据为N，每个磁盘块的数据项的数量是m，则有h=㏒(m+1)N，当数据量N一定的情况下，m越大，h越小；而m = 磁盘块的大小 / 数据项的大小，磁盘块的大小也就是一个数据页的大小，是固定的，如果数据项占的空间越小，数据项的数量越多，树的高度越低。这就是为什么每个数据项，即索引字段要尽量的小，比如int占4字节，要比bigint8字节少一半。这也是为什么b+树要求把真实的数据放到叶子节点而不是内层节点，一旦放到内层节点，磁盘块的数据项会大幅度下降，导致树增高。当数据项等于1时将会退化成线性表。
    
    2. 当b+树的数据项是复合的数据结构，比如(name,age,sex)的时候，b+数是按照从左到右的顺序来建立搜索树的，比如当(张三,20,F)这样的数据来检索的时候，b+树会优先比较name来确定下一步的所搜方向，如果name相同再依次比较age和sex，最后得到检索的数据；但当(20,F)这样的没有name的数据来的时候，b+树就不知道下一步该查哪个节点，因为建立搜索树的时候name就是第一个比较因子，必须要先根据name来搜索才能知道下一步去哪里查询。比如当(张三,F)这样的数据来检索时，b+树可以用name来指定搜索方向，但下一个字段age的缺失，所以只能把名字等于张三的数据都找到，然后再匹配性别是F的数据了， 这个是非常重要的性质，即索引的最左匹配特性。

#### <font color=#0000FF>**Hash索引**</font>

- 主要就是通过Hash算法（常见的Hash算法有直接定址法、平方取中法、折叠法、除数取余法、随机数法），将数据库字段数据转换成定长的Hash值，与这条数据的行指针一并存入Hash表的对应位置；如果发生Hash碰撞（两个不同关键字的Hash值相同），则在对应Hash键下以链表形式存储。

  检索算法：在检索查询时，就再次对待查关键字再次执行相同的Hash算法，得到Hash值，到对应Hash表对应位置取出数据即可，如果发生Hash碰撞，则需要在取值时进行筛选。目前使用Hash索引的数据库并不多，主要有Memory等。

  MySQL目前有Memory引擎和NDB引擎支持Hash索引。


#### full-text全文索引

-  全文索引也是MyISAM的一种特殊索引类型，主要用于全文索引，InnoDB从MYSQL5.6版本提供对全文索引的支持。 

-  它用于替代效率较低的LIKE模糊匹配操作，而且可以通过多字段组合的全文索引一次性全模糊匹配多个字段。 
-  同样使用B-Tree存放索引数据，但使用的是特定的算法，将字段数据分割后再进行索引（一般每4个字节一次分割），索引文件存储的是分割前的索引字符串集合，与分割后的索引信息，对应Btree结构的节点存储的是分割后的词信息以及它在分割前的索引字符串集合中的位置。 

#### R-Tree空间索引

 空间索引是MyISAM的一种特殊索引类型，主要用于地理空间数据类型 



### MySQL索引分类

#### 数据结构角度

- B+树索引 
- Hash索引
- Full-Text全文索引
- R-Tree索引

#### 从物理存储角度

- 聚集索引（clustered index） 

- 非聚集索引（non-clustered index），也叫辅助索引（secondary index）

  聚集索引和非聚集索引都是B+树结构

####  从逻辑角度 

-  主键索引：主键索引是一种特殊的唯一索引，不允许有空值 
-  普通索引或者单列索引 
-  多列索引（复合索引）：复合索引指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。使用复合索引时遵循最左前缀集合 
-  唯一索引或者非唯一索引 
-  空间索引：空间索引是对空间数据类型的字段建立的索引，MYSQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING、POLYGON。
   MYSQL使用SPATIAL关键字进行扩展，使得能够用于创建正规索引类型的语法创建空间索引。创建空间索引的列，必须将其声明为NOT NULL，空间索引只能在存储引擎为MYISAM的表中创建 



### MySQL高效索引

**覆盖索引**（Covering Index）,或者叫索引覆盖， 也就是平时所说的不需要回表操作 

- 就是select的数据列只用从索引中就能够取得，不必读取数据行，MySQL可以利用索引返回select列表中的字段，而不必根据索引再次读取数据文件，换句话说**查询列要被所建的索引覆盖**。

- 索引是高效找到行的一个方法，但是一般数据库也能使用索引找到一个列的数据，因此它不必读取整个行。毕竟索引叶子节点存储了它们索引的数据，当能通过读取索引就可以得到想要的数据，那就不需要读取行了。一个索引包含（覆盖）满足查询结果的数据就叫做覆盖索引。

- **判断标准** 

  使用explain，可以通过输出的extra列来判断，对于一个索引覆盖查询，显示为**using index**,MySQL查询优化器在执行查询前会决定是否有索引覆盖查询 



> [美团技术-MySQL索引原理及慢查询优化](https://tech.meituan.com/2014/06/30/mysql-index.html)  





## MySQL查询





# MySQL 事务

MySQL 事务主要用于处理操作量大，复杂度高的数据。比如说，在人员管理系统中，你删除一个人员，你即需要删除人员的基本资料，也要删除和该人员相关的信息，如信箱，文章等等，这样，这些数据库操作语句就构成一个事务！ 



### ACID — 事务基本要素

事务是由一组SQL语句组成的逻辑处理单元，具有4个属性，通常简称为事务的ACID属性。

- **A (Atomicity) 原子性**：整个事务中的所有操作，要么全部完成，要么全部不完成，不可能停滞在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。
- **C (Consistency) 一致性**：在事务开始之前和事务结束以后，数据库的完整性约束没有被破坏。
- **I (Isolation)隔离性**：隔离状态执行事务，使它们好像是系统在给定时间内执行的唯一操作。如果有两个事务，运行在相同的时间内，执行 相同的功能，事务的隔离性将确保每一事务在系统中认为只有该事务在使用系统。这种属性有时称为串行化，为了防止事务操作间的混淆，必须串行化或序列化请 求，使得在同一时间仅有一个请求用于同一数据。
- **D (Durability) 持久性**：在事务完成以后，该事务所对数据库所作的更改便持久的保存在数据库之中，并不会被回滚。



### 事务隔离级别

**并发事务处理带来的问题**

- 更新丢失（Lost Update)： 事务A和事务B选择同一行，然后基于最初选定的值更新该行时，由于两个事务都不知道彼此的存在，就会发生丢失更新问题 
- 脏读(Dirty Reads)：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
- 不可重复读（Non-Repeatable Reads)：事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果 不一致。
- 幻读（Phantom Reads)：系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。



**幻读和不可重复读的区别：**

- 不可重复读的重点是修改：在同一事务中，同样的条件，第一次读的数据和第二次读的数据不一样。（因为中间有其他事务提交了修改）
- 幻读的重点在于新增或者删除：在同一事务中，同样的条件,，第一次和第二次读出来的记录数不一样。（因为中间有其他事务提交了插入/删除）



**并发事务处理带来的问题的解决办法：**

- “更新丢失”通常是应该完全避免的。但防止更新丢失，并不能单靠数据库事务控制器来解决，需要应用程序对要更新的数据加必要的锁来解决，因此，防止更新丢失应该是应用的责任。

- “脏读” 、 “不可重复读”和“幻读” ，其实都是数据库读一致性问题，必须由数据库提供一定的事务隔离机制来解决：

  - 一种是加锁：在读取数据前，对其加锁，阻止其他事务对数据进行修改。
  - 另一种是数据多版本并发控制（MultiVersion Concurrency Control，简称 **MVCC** 或 MCC），也称为多版本数据库：不用加任何锁， 通过一定机制生成一个数据请求时间点的一致性数据快照 （Snapshot)， 并用这个快照来提供一定级别 （语句级或事务级） 的一致性读取。从用户的角度来看，好象是数据库可以提供同一数据的多个版本。



查看当前数据库的事务隔离级别：  

```mysql
show variables like 'tx_isolation'
```



数据库事务的隔离级别有4种，由低到高分别为

- Read uncommitted  读未提交
- Read committed   读已提交
- Repeatable read   可重复读
- Serializable  串行



下面通过事例一一阐述在事务的并发操作中可能会出现脏读，不可重复读，幻读和事务隔离级别的联系。

数据库的事务隔离越严格，并发副作用越小，但付出的代价就越大，因为事务隔离实质上就是使事务在一定程度上“串行化”进行，这显然与“并发”是矛盾的。同时，不同的应用对读一致性和事务隔离程度的要求也是不同的，比如许多应用对“不可重复读”和“幻读”并不敏感，可能更关系数据并发访问的能力。

#### Read uncommitted

读未提交，就是一个事务可以读取另一个未提交事务的数据。

事例：老板要给程序员发工资，程序员的工资是3.6万/月。但是发工资时老板不小心按错了数字，按成3.9万/月，该钱已经打到程序员的户口，但是事务还没有提交，就在这时，程序员去查看自己这个月的工资，发现比往常多了3千元，以为涨工资了非常高兴。但是老板及时发现了不对，马上回滚差点就提交了的事务，将数字改成3.6万再提交。

分析：实际程序员这个月的工资还是3.6万，但是程序员看到的是3.9万。他看到的是老板还没提交事务时的数据。这就是脏读。

那怎么解决脏读呢？Read committed！读提交，能解决脏读问题。

#### Read committed

读提交，顾名思义，就是一个事务要等另一个事务提交后才能读取数据。

事例：程序员拿着信用卡去享受生活（卡里当然是只有3.6万），当他埋单时（程序员事务开启），收费系统事先检测到他的卡里有3.6万，就在这个时候！！程序员的妻子要把钱全部转出充当家用，并提交。当收费系统准备扣款时，再检测卡里的金额，发现已经没钱了（第二次检测金额当然要等待妻子转出金额事务提交完）。程序员就会很郁闷，明明卡里是有钱的…

分析：这就是读提交，若有事务对数据进行更新（UPDATE）操作时，读操作事务要等待这个更新操作事务提交后才能读取数据，可以解决脏读问题。但在这个事例中，出现了一个事务范围内两个相同的查询却返回了不同数据，这就是不可重复读。

那怎么解决可能的不可重复读问题？Repeatable read ！

#### Repeatable read

重复读，就是在开始读取数据（事务开启）时，不再允许修改操作。 **MySQL的默认事务隔离级别** 

事例：程序员拿着信用卡去享受生活（卡里当然是只有3.6万），当他埋单时（事务开启，不允许其他事务的UPDATE修改操作），收费系统事先检测到他的卡里有3.6万。这个时候他的妻子不能转出金额了。接下来收费系统就可以扣款了。

分析：重复读可以解决不可重复读问题。写到这里，应该明白的一点就是，不可重复读对应的是修改，即UPDATE操作。但是可能还会有幻读问题。因为幻读问题对应的是插入INSERT操作，而不是UPDATE操作。

**什么时候会出现幻读？**

事例：程序员某一天去消费，花了2千元，然后他的妻子去查看他今天的消费记录（全表扫描FTS，妻子事务开启），看到确实是花了2千元，就在这个时候，程序员花了1万买了一部电脑，即新增INSERT了一条消费记录，并提交。当妻子打印程序员的消费记录清单时（妻子事务提交），发现花了1.2万元，似乎出现了幻觉，这就是幻读。

那怎么解决幻读问题？Serializable！

#### Serializable 序列化

Serializable 是最高的事务隔离级别，在该级别下，事务串行化顺序执行，可以避免脏读、不可重复读与幻读。简单来说，Serializable会在读取的每一行数据上都加锁，所以可能导致大量的超时和锁争用问题。这种事务隔离级别效率低下，比较耗数据库性能，一般不使用。



| 事务隔离级别                 | 读数据一致性                             | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---------------------------------------- | ---- | ---------- | ---- |
| 读未提交（read-uncommitted） | 最低级被，只能保证不读取物理上损坏的数据 | 是   | 是         | 是   |
| 读已提交（read-committed）   | 语句级                                   | 否   | 是         | 是   |
| 可重复读（repeatable-read）  | 事务级                                   | 否   | 否         | 是   |
| 串行化（serializable）       | 最高级别，事务级                         | 否   | 否         | 否   |



需要说明的是，事务隔离级别和数据访问的并发性是对立的，事务隔离级别越高并发性就越差。所以要根据具体的应用来确定合适的事务隔离级别，这个地方没有万能的原则。



### MVCC 多版本并发控制

MySQL的大多数事务型存储引擎实现都不是简单的行级锁。基于提升并发性考虑，一般都同时实现了多版本并发控制（MVCC），包括Oracle、PostgreSQL。只是实现机制各不相同。

可以认为MVCC是行级锁的一个变种，但它在很多情况下避免了加锁操作，因此开销更低。虽然实现机制有所不同，但大都实现了非阻塞的读操作，写操作也只是锁定必要的行。

MVCC的实现是通过保存数据在某个时间点的快照来实现的。也就是说不管需要执行多长时间，每个事物看到的数据都是一致的。

典型的MVCC实现方式，分为**乐观（optimistic）并发控制和悲观（pressimistic）并发控制**。下边通过InnoDB的简化版行为来说明MVCC是如何工作的。

InnoDB的MVCC，是通过在每行记录后面保存两个隐藏的列来实现。这两个列，一个保存了行的创建时间，一个保存行的过期时间（删除时间）。当然存储的并不是真实的时间，而是系统版本号（system version number）。每开始一个新的事务，系统版本号都会自动递增。事务开始时刻的系统版本号会作为事务的版本号，用来和查询到的每行记录的版本号进行比较。 

**REPEATABLE READ（可重读）隔离级别下MVCC如何工作：**

- SELECT

InnoDB会根据以下两个条件检查每行记录：

1. InnoDB只查找版本早于当前事务版本的数据行，这样可以确保事务读取的行,要么是在开始事务之前已经存在要么是事务自身插入或者修改过的

2. 行的删除版本号要么未定义，要么大于当前事务版本号，这样可以确保事务读取到的行在事务开始之前未被删除

   只有符合上述两个条件的才会被查询出来

- INSERT

  InnoDB为新插入的每一行保存当前系统版本号作为行版本号

- DELETE

  InnoDB为删除的每一行保存当前系统版本号作为行删除标识

- UPDATE

  InnoDB为插入的一行新纪录保存当前系统版本号作为行版本号，同时保存当前系统版本号到原来的行作为删除标识

保存这两个额外系统版本号，使大多数操作都不用加锁。使数据操作简单，性能很好，并且也能保证只会读取到符合要求的行。不足之处是每行记录都需要额外的存储空间，需要做更多的行检查工作和一些额外的维护工作。

MVCC只在COMMITTED READ（读提交）和REPEATABLE READ（可重复读）两种隔离级别下工作。



### 事务日志

事务日志可以帮助提高事务效率：

- 使用事务日志，存储引擎在修改表的数据时只需要修改其内存拷贝，再把该修改行为记录到持久在硬盘上的事务日志中，而不用每次都将修改的数据本身持久到磁盘。
- 事务日志采用的是追加的方式，因此写日志的操作是磁盘上一小块区域内的顺序I/O，而不像随机I/O需要在磁盘的多个地方移动磁头，所以采用事务日志的方式相对来说要快得多。
- 事务日志持久以后，内存中被修改的数据在后台可以慢慢刷回到磁盘。
- 如果数据的修改已经记录到事务日志并持久化，但数据本身没有写回到磁盘，此时系统崩溃，存储引擎在重启时能够自动恢复这一部分修改的数据。

目前来说，大多数存储引擎都是这样实现的，我们通常称之为**预写式日志**（Write-Ahead Logging），修改数据需要写两次磁盘。



### 事务的实现

事务的实现是基于数据库的存储引擎。不同的存储引擎对事务的支持程度不一样。mysql中支持事务的存储引擎有innoDB和NDB。 

事务的实现就是如何实现ACID特性。

innoDB是mysql默认的存储引擎，默认的隔离级别是RR（Repeatable Read），并且在RR的隔离级别下更进一步，通过多版本**并发控制**（MVCC，Multiversion Concurrency Control ）解决不可重复读问题，加上间隙锁（也就是并发控制）解决幻读问题。因此innoDB的RR隔离级别其实实现了串行化级别的效果，而且保留了比较好的并发性能。 



事务的隔离性是通过锁实现，而事务的原子性、一致性和持久性则是通过事务日志实现 。



> 事务是如何通过日志来实现的，说得越深入越好。

**redo log（重做日志**） 实现持久化和原子性。

在innoDB的存储引擎中，事务日志通过重做(redo)日志和innoDB存储引擎的日志缓冲(InnoDB Log Buffer)实现。事务开启时，事务中的操作，都会先写入存储引擎的日志缓冲中，在事务提交之前，这些缓冲的日志都需要提前刷新到磁盘上持久化，这就是DBA们口中常说的“日志先行”(Write-Ahead Logging)。当事务提交之后，在Buffer Pool中映射的数据文件才会慢慢刷新到磁盘。此时如果数据库崩溃或者宕机，那么当系统重启进行恢复时，就可以根据redo log中记录的日志，把数据库恢复到崩溃前的一个状态。未完成的事务，可以继续提交，也可以选择回滚，这基于恢复的策略而定。

在系统启动的时候，就已经为redo log分配了一块连续的存储空间,以顺序追加的方式记录Redo Log,通过顺序IO来改善性能。所有的事务共享redo log的存储空间，它们的Redo Log按语句的执行顺序，依次交替的记录在一起。



 **undo log**  实现一致性

 undo log主要为事务的回滚服务。在事务执行的过程中，除了记录redo log，还会记录一定量的undo log。undo log记录了数据在每个操作前的状态，如果事务执行过程中需要回滚，就可以根据undo log进行回滚操作。单个事务的回滚，只会回滚当前事务做的操作，并不会影响到其他的事务做的操作。 



二种日志均可以视为一种恢复操作，redo_log是恢复提交事务修改的页操作，而undo_log是回滚行记录到特定版本。二者记录的内容也不同，redo_log是物理日志，记录页的物理修改操作，而undo_log是逻辑日志，根据每行记录进行记录。



### 有多少种日志

**错误日志**：记录出错信息，也记录一些警告信息或者正确的信息。

**查询日志**：记录所有对数据库请求的信息，不论这些请求是否得到了正确的执行。

**慢查询日志**：设置一个阈值，将运行时间超过该值的所有SQL语句都记录到慢查询的日志文件中。

**二进制日志**：记录对数据库执行更改的所有操作。

**中继日志**：中继日志也是二进制日志，用来给slave 库恢复

**事务日志**：重做日志redo和回滚日志undo



### MySQL对分布式事务的支持

分布式事务的实现方式有很多，既可以采用innoDB提供的原生的事务支持，也可以采用消息队列来实现分布式事务的最终一致性。这里我们主要聊一下innoDB对分布式事务的支持。

MySQL 从 5.0.3 开始支持分布式事务，**当前分布式事务只支持 InnoDB 存储引擎**。一个分布式事务会涉及多个行动，这些行动本身是事务性的。所有行动都必须一起成功完成，或者一起被回滚。

![img](H:/Technical-Learning/docs/_images/mysql/mysql-xa-transactions.png)

如图，mysql的分布式事务模型。模型中分三块：应用程序（AP）、资源管理器（RM）、事务管理器（TM）:

- 应用程序：定义了事务的边界，指定需要做哪些事务；
- 资源管理器：提供了访问事务的方法，通常一个数据库就是一个资源管理器；
- 事务管理器：协调参与了全局事务中的各个事务。

分布式事务采用两段式提交（two-phase commit）的方式：

- 第一阶段所有的事务节点开始准备，告诉事务管理器ready。
- 第二阶段事务管理器告诉每个节点是commit还是rollback。如果有一个节点失败，就需要全局的节点全部rollback，以此保障事务的原子性。

 

# MySQL锁

锁是计算机协调多个进程或线程并发访问某一资源的机制。

在数据库中，除传统的计算资源（如CPU、RAM、I/O等）的争用以外，数据也是一种供许多用户共享的资源。数据库锁定机制简单来说，就是数据库为了保证数据的一致性，而使各种共享资源在被并发访问变得有序所设计的一种规则

打个比方，我们到淘宝上买一件商品，商品只有一件库存，这个时候如果还有另一个人买，那么如何解决是你买到还是另一个人买到的问题？


这里肯定要用到事物，我们先从库存表中取出物品数量，然后插入订单，付款后插入付款表信息，然后更新商品数量。在这个过程中，使用锁可以对有限的资源进行保护，解决隔离和并发的矛盾。

 

## 锁的分类

#### 从对数据操作的类型分类：

- **读锁**（共享锁）：针对同一份数据，多个读操作可以同时进行，不会互相影响

- **写锁**（排他锁）：当前写操作没有完成前，它会阻断其他写锁和读锁。

#### 从对数据操作的粒度分类：


为了尽可能提高数据库的并发度，每次锁定的数据范围越小越好，理论上每次只锁定当前操作的数据的方案会得到最大的并发度，但是管理锁是很耗资源的事情（涉及获取，检查，释放锁等动作），因此数据库系统需要在高并发响应和系统性能两方面进行平衡，这样就产生了“锁粒度（Lock granularity）”的概念。

- **表级锁**：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低；

- **行级锁**：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高；  

- **页面锁**：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般。

适用：从锁的角度来说，表级锁更适合于以查询为主，只有少量按索引条件更新数据的应用，如Web应用；而行级锁则更适合于有大量按索引条件并发更新少量不同数据，同时又有并发查询的应用，如一些在线事务处理（OLTP）系统。



#### 加锁机制

**乐观锁与悲观锁是两种并发控制的思想，可用于解决丢失更新问题** 

乐观锁会“乐观地”假定大概率不会发生并发更新冲突，访问、处理数据过程中不加锁，只在更新数据时再根据版本号或时间戳判断是否有冲突，有则处理，无则提交事务；

悲观锁会“悲观地”假定大概率会发生并发更新冲突，访问、处理数据前就加排他锁，在整个数据处理过程中锁定数据，事务提交或回滚后才释放锁；



#### 锁模式

- 记录锁： 对索引项加锁，锁定符合条件的行。其他事务不能修改 和删除加锁项； 
- gap锁： 对索引项之间的“间隙”加锁，锁定记录的范围（对第一条记录前的间隙或最后一条将记录后的间隙加锁），不包含索引项本身。其他事务不能在锁范围内插入数据，这样就防止了别的事务新增幻影行。 
- next-key锁： 锁定索引项本身和索引范围。即Record Lock和Gap Lock的结合。可解决幻读问题。 
- 意向锁
- 插入意向锁



 MySQL 不同的存储引擎支持不同的锁机制，所有的存储引擎都以自己的方式实现了锁机制 

|        | 行锁 | 表锁 | 页锁 |
| ------ | ---- | ---- | ---- |
| MyISAM |      | √    |      |
| BDB    |      | √    | √    |
| InnoDB | √    | √    |      |
| Memory |      | √    |      |



## 1. 影响mysql的性能因素

##### 1.1 业务需求对mysql的影响(合适合度)

##### 1.2 存储定位对mysql的影响

- 不适合放进mysql的数据
  - 二进制多媒体数据
  - 流水队列数据
  - 超大文本数据
- 需要放进缓存的数据
  - 系统各种配置及规则数据
  - 活跃用户的基本信息数据
  - 活跃用户的个性化定制信息数据
  - 准实时的统计信息数据
  - 其他一些访问频繁但变更较少的数据

##### 1.3 Schema设计对系统的性能影响

- 尽量减少对数据库访问的请求
- 尽量减少无用数据的查询请求

##### 1.4 硬件环境对系统性能的影响



## 2. 性能分析

### 2.1 MySQL常见瓶颈

- CPU：CPU在饱和的时候一般发生在数据装入内存或从磁盘上读取数据时候

- IO：磁盘I/O瓶颈发生在装入数据远大于内存容量的时候

- 服务器硬件的性能瓶颈：top,free, iostat和vmstat来查看系统的性能状态



**查看Linux系统性能的常用命令**

MySQL数据库是常见的两个瓶颈是CPU和I/O的瓶颈。CPU在饱和的时候一般发生在数据装入内存或从磁盘上读取数据时候，磁盘I/O瓶颈发生在装入数据远大于内存容量的时候，如果应用分布在网络上，那么查询量相当大的时候那么瓶颈就会出现在网络上。Linux中我们常用mpstat、vmstat、iostat、sar和top来查看系统的性能状态。

`mpstat`： mpstat是Multiprocessor Statistics的缩写，是实时系统监控工具。其报告为CPU的一些统计信息，这些信息存放在/proc/stat文件中。在多CPUs系统里，其不但能查看所有CPU的平均状况信息，而且能够查看特定CPU的信息。mpstat最大的特点是可以查看多核心cpu中每个计算核心的统计数据，而类似工具vmstat只能查看系统整体cpu情况。

`vmstat`：vmstat命令是最常见的Linux/Unix监控工具，可以展现给定时间间隔的服务器的状态值，包括服务器的CPU使用率，内存使用，虚拟内存交换情况，IO读写情况。这个命令是我查看Linux/Unix最喜爱的命令，一个是Linux/Unix都支持，二是相比top，我可以看到整个机器的CPU、内存、IO的使用情况，而不是单单看到各个进程的CPU使用率和内存使用率(使用场景不一样)。

`iostat`: 主要用于监控系统设备的IO负载情况，iostat首次运行时显示自系统启动开始的各项统计信息，之后运行iostat将显示自上次运行该命令以后的统计信息。用户可以通过指定统计的次数和时间来获得所需的统计信息。

`sar`： sar（System Activity Reporter系统活动情况报告）是目前 Linux 上最为全面的系统性能分析工具之一，可以从多方面对系统的活动进行报告，包括：文件的读写情况、系统调用的使用情况、磁盘I/O、CPU效率、内存使用状况、进程活动及IPC有关的活动等。

`top`：top命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，类似于Windows的任务管理器。top显示系统当前的进程和其他状况，是一个动态显示过程,即可以通过用户按键来不断刷新当前状态.如果在前台执行该命令，它将独占前台，直到用户终止该程序为止。比较准确的说，top命令提供了实时的对系统处理器的状态监视。它将显示系统中CPU最“敏感”的任务列表。该命令可以按CPU使用。内存使用和执行时间对任务进行排序；而且该命令的很多特性都可以通过交互式命令或者在个人定制文件中进行设定。

除了服务器硬件的性能瓶颈，对于MySQL系统本身，我们可以使用工具来优化数据库的性能，通常有三种：使用索引，使用EXPLAIN分析查询以及调整MySQL的内部配置。



### 2.2 性能下降SQL慢 执行时间长 等待时间长 原因分析

- 查询语句写的烂
- 索引失效（单值 复合）
- 关联查询太多join（设计缺陷或不得已的需求）
- 服务器调优及各个参数设置（缓冲、线程数等）



### 2.3 MySql Query Optimizer

1. Mysql中有专门负责优化SELECT语句的优化器模块，主要功能：通过计算分析系统中收集到的统计信息，为客户端请求的Query提供他认为最优的执行计划（他认为最优的数据检索方式，但不见得是DBA认为是最优的，这部分最耗费时间）

2. 当客户端向MySQL 请求一条Query，命令解析器模块完成请求分类，区别出是 SELECT 并转发给MySQL Query Optimizer时，MySQL Query Optimizer 首先会对整条Query进行优化，处理掉一些常量表达式的预算，直接换算成常量值。并对 Query 中的查询条件进行简化和转换，如去掉一些无用或显而易见的条件、结构调整等。然后分析 Query 中的 Hint 信息（如果有），看显示Hint信息是否可以完全确定该Query 的执行计划。如果没有 Hint 或Hint 信息还不足以完全确定执行计划，则会读取所涉及对象的统计信息，根据 Query 进行写相应的计算分析，然后再得出最后的执行计划。



### 2.4 MySQL常见性能分析手段

在优化MySQL时，通常需要对数据库进行分析，常见的分析手段有**慢查询日志**，**EXPLAIN 分析查询**，**profiling分析**以及**show命令查询系统状态及系统变量**，通过定位分析性能的瓶颈，才能更好的优化数据库系统的性能。

####  2.4.1 性能瓶颈定位 

我们可以通过show命令查看MySQL状态及变量，找到系统的瓶颈：

```shell
Mysql> show status ——显示状态信息（扩展show status like ‘XXX’）

Mysql> show variables ——显示系统变量（扩展show variables like ‘XXX’）

Mysql> show innodb status ——显示InnoDB存储引擎的状态

Mysql> show processlist ——查看当前SQL执行，包括执行状态、是否锁表等

Shell> mysqladmin variables -u username -p password——显示系统变量

Shell> mysqladmin extended-status -u username -p password——显示状态信息
```



#### 2.4.2 Explain(执行计划)

- 是什么：使用Explain关键字可以模拟优化器执行SQL查询语句，从而知道MySQL是如何处理你的SQL语句的。分析你的查询语句或是表结构的性能瓶颈
- 能干吗
  - 表的读取顺序
  - 数据读取操作的操作类型
  - 哪些索引可以使用
  - 哪些索引被实际使用
  - 表之间的引用
  - 每张表有多少行被优化器查询

- 怎么玩

  - Explain + SQL语句
  - 执行计划包含的信息

![expalin](H:/Technical-Learning/docs/_images/mysql/expalin.jpg)

- 各字段解释

  - <mark>**id**</mark>（select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序）

    - id相同，执行顺序从上往下
    - id不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
    - id相同不同，同时存在

  - <mark> **select_type**</mark>（查询的类型，用于区别普通查询、联合查询、子查询等复杂查询）

    - **SIMPLE** ：简单的select查询，查询中不包含子查询或UNION
    - **PRIMARY**：查询中若包含任何复杂的子部分，最外层查询被标记为PRIMARY
    - **SUBQUERY**：在select或where列表中包含了子查询
    - **DERIVED**：在from列表中包含的子查询被标记为DERIVED，mysql会递归执行这些子查询，把结果放在临时表里
    - **UNION**：若第二个select出现在UNION之后，则被标记为UNION，若UNION包含在from子句的子查询中，外层select将被标记为DERIVED
    - **UNION RESULT**：从UNION表获取结果的select

  - <mark> **table**</mark>（显示这一行的数据是关于哪张表的）

  - <mark> **type**</mark>（显示查询使用了那种类型，从最好到最差依次排列	**system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL** ）

    - system：表只有一行记录（等于系统表），是const类型的特例，平时不会出现
    - const：表示通过索引一次就找到了，const用于比较primary key或unique索引，因为只要匹配一行数据，所以很快，如将主键置于where列表中，mysql就能将该查询转换为一个常量
    - eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描
    - ref：非唯一性索引扫描，范围匹配某个单独值得所有行。本质上也是一种索引访问，他返回所有匹配某个单独值的行，然而，它可能也会找到多个符合条件的行，多以他应该属于查找和扫描的混合体
    - range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是在你的where语句中出现了between、<、>、in等的查询，这种范围扫描索引比全表扫描要好，因为它只需开始于索引的某一点，而结束于另一点，不用扫描全部索引
    - index：Full Index Scan，index于ALL区别为index类型只遍历索引树。通常比ALL快，因为索引文件通常比数据文件小。（**也就是说虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘中读的**）
    - all：Full Table Scan，将遍历全表找到匹配的行

     ?> 一般来说，得保证查询至少达到range级别，最好到达ref

  - <mark> **possible_keys**</mark>（显示可能应用在这张表中的索引，一个或多个，查询涉及到的字段若存在索引，则该索引将被列出，但不一定被查询实际使用）

  - <mark> **key**</mark> 

    - （实际使用的索引，如果为NULL，则没有使用索引）

    - **查询中若使用了覆盖索引，则该索引和查询的select字段重叠，仅出现在key列表中**

  ![explain-key](H:/Technical-Learning/docs/_images/mysql/explain-key.png)

  - <mark> **key_len**</mark>

    - 表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精确性的情况下，长度越短越好
    - key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的

  - <mark> **ref**</mark> （显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值）

  - <mark> **rows**</mark> （根据表统计信息及索引选用情况，大致估算找到所需的记录所需要读取的行数）

  - <mark> **Extra**</mark>（包含不适合在其他列中显示但十分重要的额外信息）

    1. <font color=red>using filesort</font>: 说明mysql会对数据使用一个外部的索引排序，不是按照表内的索引顺序进行读取。mysql中无法利用索引完成的排序操作称为“文件排序”![img](H:/Technical-Learning/docs/_images/mysql/explain-extra-1.png)

    2. <font color=red>Using temporary</font>：使用了临时表保存中间结果，mysql在对查询结果排序时使用临时表。常见于排序order by和分组查询group by。![explain-extra-2](H:/Technical-Learning/docs/_images/mysql/explain-extra-2.png)

    3. <font color=red>using index</font>：表示相应的select操作中使用了覆盖索引，避免访问了表的数据行，效率不错，如果同时出现using where，表明索引被用来执行索引键值的查找；否则索引被用来读取数据而非执行查找操作![explain-extra-3](H:/Technical-Learning/docs/_images/mysql/explain-extra-3.png)

    4. using where：使用了where过滤

    5. using join buffer：使用了连接缓存

    6. impossible where：where子句的值总是false，不能用来获取任何元祖

    7. select tables optimized away：在没有group by子句的情况下，基于索引优化操作或对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化

    8. distinct：优化distinct操作，在找到第一匹配的元祖后即停止找同样值的动作

       

- case:

![explain-demo](H:/Technical-Learning/docs/_images/mysql/explain-demo.png)

 

1. 第一行（执行顺序4）：id列为1，表示是union里的第一个select，select_type列的primary表 示该查询为外层查询，table列被标记为<derived3>，表示查询结果来自一个衍生表，其中derived3中3代表该查询衍生自第三个select查询，即id为3的select。【select d1.name......】

2. 第二行（执行顺序2）：id为3，是整个查询中第三个select的一部分。因查询包含在from中，所以为derived。【select id,name from t1 where other_column=''】
3. 第三行（执行顺序3）：select列表中的子查询select_type为subquery，为整个查询中的第二个select。【select id from t3】
4. 第四行（执行顺序1）：select_type为union，说明第四个select是union里的第二个select，最先执行【select name,id from t2】
5. 第五行（执行顺序5）：代表从union的临时表中读取行的阶段，table列的<union1,4>表示用第一个和第四个select的结果进行union操作。【两个结果union操作】



#### 2.4.3 慢查询日志

MySQL的慢查询日志是MySQL提供的一种日志记录，它用来记录在MySQL中响应时间超过阈值的语句，具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志中。

- long_query_time的默认值为10，意思是运行10秒以上的语句。
- 默认情况下，MySQL数据库没有开启慢查询日志，需要手动设置参数开启。
- 如果不是调优需要的话，一般不建议启动该参数。

**查看开启状态**

`SHOW VARIABLES LIKE '%slow_query_log%'`

**开启慢查询日志**

- 临时配置：

```mysql
mysql> set global slow_query_log='ON';
mysql> set global slow_query_log_file='/var/lib/mysql/hostname-slow.log';
mysql> set global long_query_time=2;
```

​	也可set文件位置，系统会默认给一个缺省文件host_name-slow.log

​	使用set操作开启慢查询日志只对当前数据库生效，如果MySQL重启则会失效。

- 永久配置

  修改配置文件my.cnf或my.ini，在[mysqld]一行下面加入两个配置参数

```cnf
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/lib/mysql/hostname-slow.log
long_query_time = 3
```

​	注：log-slow-queries参数为慢查询日志存放的位置，一般这个目录要有mysql的运行帐号的可写权限，一般都	将这个目录设置为mysql的数据存放目录；long_query_time=2中的2表示查询超过两秒才记录；在my.cnf或者	my.ini中添加log-queries-not-using-indexes参数，表示记录下没有使用索引的查询。

可以用 `select sleep(4)` 验证是否成功开启。

在生产环境中，如果手工分析日志，查找、分析SQL，还是比较费劲的，所以MySQL提供了日志分析工具mysqldumpslow。

通过 mysqldumpslow --help查看操作帮助信息

- 得到返回记录集最多的10个SQL

  `mysqldumpslow -s r -t 10 /var/lib/mysql/hostname-slow.log`

- 得到访问次数最多的10个SQL

  `mysqldumpslow -s c -t 10 /var/lib/mysql/hostname-slow.log`

- 得到按照时间排序的前10条里面含有左连接的查询语句

  `mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/hostname-slow.log`

- 也可以和管道配合使用

  `mysqldumpslow -s r -t 10 /var/lib/mysql/hostname-slow.log | more`

**也可使用 pt-query-digest 分析 RDS MySQL 慢查询日志**



#### 2.4.4 Show Profile分析查询

通过慢日志查询可以知道哪些SQL语句执行效率低下，通过explain我们可以得知SQL语句的具体执行情况，索引使用等，还可以结合Show Profile命令查看执行状态。

- Show Profile是mysql提供可以用来分析当前会话中语句执行的资源消耗情况。可以用于SQL的调优的测量

- 默认情况下，参数处于关闭状态，并保存最近15次的运行结果

- 分析步骤

  1. 是否支持，看看当前的mysql版本是否支持

     ```mysql
     mysql>Show  variables like 'profiling';  --默认是关闭，使用前需要开启
     ```

  2. 开启功能，默认是关闭，使用前需要开启

     ```mysql
     mysql>set profiling=1;  
     ```

  3. 运行SQL

  4. 查看结果，show profiles;

  5. 诊断SQL，show profile cpu,block io for query 上一步前面的问题SQL数字号码;

  6. 日常开发需要注意的结论

     - converting HEAP to MyISAM 查询结果太大，内存都不够用了往磁盘上搬了。

     - create tmp table 创建临时表，这个要注意

     - Copying to tmp table on disk   把内存临时表复制到磁盘

     - locked



## 3. 性能优化

### 3.1 索引优化

1. 全值匹配我最爱
2. 最佳左前缀法则
3. 不在索引列上做任何操作（计算、函数、(自动or手动)类型转换），会导致索引失效而转向全表扫描
4. 存储引擎不能使用索引中范围条件右边的列
5. 尽量使用覆盖索引(只访问索引的查询(索引列和查询列一致))，减少select 
6. mysql 在使用不等于(!= 或者<>)的时候无法使用索引会导致全表扫描
7. is null ,is not null 也无法使用索引
8. like以通配符开头('%abc...')mysql索引失效会变成全表扫描的操作
9. 字符串不加单引号索引失效
10. 少用or,用它来连接时会索引失效



**一般性建议**

- 对于单键索引，尽量选择针对当前query过滤性更好的索引

- 在选择组合索引的时候，当前Query中过滤性最好的字段在索引字段顺序中，位置越靠前越好。

- 在选择组合索引的时候，尽量选择可以能够包含当前query中的where字句中更多字段的索引

- 尽可能通过分析统计信息和调整query的写法来达到选择合适索引的目的

- 少用Hint强制索引

  

### 3.2 查询优化

- **永远小标驱动大表（小的数据集驱动大的数据集）**

  `slect * from A where id in (select id from B)`

  等价于

  `select id from B`

  `select * from A where A.id=B.id`

  当B表的数据集必须小于A表的数据集时，用in优于exists

  `select * from A where exists (select 1 from B where B.id=A.id)`

  等价于

  `select *from A`

  `select * from B where B.id = A.id`

  当A表的数据集小于B表的数据集时，用exists优于用in

  注意：A表与B表的ID字段应建立索引。

  

- order by关键字优化

- - order by子句，尽量使用Index方式排序，避免使用FileSort方式排序

- - - mysql支持两种方式的排序，FileSort和Index,Index效率高，它指MySQL扫描索引本身完成排序，FileSort效率较低；
    - ORDER BY 满足两种情况，会使用Index方式排序；①ORDER BY语句使用索引最左前列 ②使用where子句与ORDER BY子句条件列组合满足索引最左前列

![optimization-orderby](H:/Technical-Learning/docs/_images/mysql/optimization-orderby.png)

- - 尽可能在索引列上完成排序操作，遵照索引建的最佳最前缀
  - 如果不在索引列上，filesort有两种算法，mysql就要启动双路排序和单路排序

- - - 双路排序
    - 单路排序
    - 由于单路是后出的，总体而言好过双路

- - 优化策略

- - - 增大sort_buffer_size参数的设置

    - 增大max_lencth_for_sort_data参数的设置

      ![optimization-orderby2](H:/Technical-Learning/docs/_images/mysql/optimization-orderby2.png)

- GROUP BY关键字优化

  - group by实质是先排序后进行分组，遵照索引建的最佳左前缀
  - 当无法使用索引列，增大max_length_for_sort_data参数的设置+增大sort_buffer_size参数的设置
  - where高于having，能写在where限定的条件就不要去having限定了。



### 3.3 数据类型优化

MySQL支持的数据类型非常多，选择正确的数据类型对于获取高性能至关重要。不管存储哪种类型的数据，下面几个简单的原则都有助于做出更好的选择。

- 更小的通常更好：一般情况下，应该尽量使用可以正确存储数据的最小数据类型。

  简单就好：简单的数据类型通常需要更少的CPU周期。例如，整数比字符操作代价更低，因为字符集和校对规则（排序规则）使字符比较比整型比较复杂。

- 尽量避免NULL：通常情况下最好指定列为NOT NULL



>  https://www.jianshu.com/p/3c79039e82aa 









分区： 就是把一张表的数据分成N多个区块，这些区块可以在同一个磁盘上，也可以在不同的磁盘上 

分表： 就是把一张表分成N多个小表。 一张表分成很多表后，每一个小表都是完正的一张表，都对应三个文件，一个.MYD数据文件，.MYI索引文件，.frm表结构文件 

分库：





## MySQL分区

  如果一张表的数据量太大的话，那么myd,myi就会变的很大，查找数据就会变的很慢，这个时候我们可以利用mysql的分区功能，在物理上将这一张表对应的三个文件，分割成许多个小块，这样呢，我们查找一条数据时，就不用全部查找了，只要知道这条数据在哪一块，然后在那一块找就行了。如果表的数据太大，可能一个磁盘放不下，这个时候，我们可以把数据分配到不同的磁盘里面去。

### 能干嘛

- 逻辑数据分割
- 提高单一的写和读应用速度
- 提高分区范围读查询的速度
- 分割数据能够有多个不同的物理文件路径
- 高效的保存历史数据



### 怎么玩

首先查看当前数据库是否支持分区

SHOW VARIABLES LIKE '%partition%';

show plugins;

subtopic



### 分区类型及操作

#### RANGE分区

mysql将会根据指定的拆分策略，,把数据放在不同的表文件上。相当于在文件上,被拆成了小块.但是,对外给客户的感觉还是一张表，透明的。

 按照 range 来分，就是每个库一段连续的数据，这个一般是按比如**时间范围**来的，但是这种一般较少用，因为很容易产生热点问题，大量的流量都打在最新的数据上了。

 range 来分，好处在于说，扩容的时候很简单 

#### List分区

MySQL中的LIST分区在很多方面类似于RANGE分区。和按照RANGE分区一样，每个分区必须明确定义。它们的主要区别在于，LIST分区中每个分区的定义和选择是基于某列的值从属于一个值列表集中的一个值，而RANGE分区是从属于一个连续区间值的集合。

#### 其它

- Hash分区: hash 分发，好处在于说，可以平均分配每个库的数据量和请求压力；坏处在于说扩容起来比较麻烦，会有一个数据迁移的过程，之前的数据需要重新计算 hash 值重新分配到不同的库或表 

- Key分区

- 子分区

#### 对NULL值的处理


MySQL中的分区在禁止空值NULL上没有进行处理，无论它是一个列值还是一个用户定义表达式的值，一般而言，在这种情况下MySQL把NULL当做零。如果你不希望出现类似情况，建议在设计表时声明该列“NOT NULL”



**看上去分区表很帅气，为什么大部分互联网还是更多的选择自己分库分表来水平扩展咧？**

回答：

1）分区表，分区键设计不太灵活，如果不走分区键，很容易出现全表锁

2）一旦数据量并发量上来，如果在分区表实施关联，就是一个灾难

3）自己分库分表，自己掌控业务场景与访问模式，可控。分区表，研发写了一个sql，都不确定mysql是怎么玩的，不太可控

4）运维的坑，嘿嘿



## Mysql分库

为什么要分库
 数据库集群环境后都是多台slave,基本满足了读取操作;
 但是写入或者说大数据、频繁的写入操作对master性能影响就比较大，
 这个时候，单库并不能解决大规模并发写入的问题。

优点

减少增量数据写入时的锁对查询的影响。

由于单表数量下降，常见的查询操作由于减少了需要扫描的记录，使得单表单次查询所需的检索行数变少，减少了磁盘IO，时延变短。

但是它无法解决单表数据量太大的问题。



 是什么？
  一个库里表太多了，导致了海量数据，系统性能下降，把原本存储于一个库的表拆分存储到多个库上，
通常是将表按照功能模块、关系密切程度划分出来，部署到不同库上。





## Mysql分表

### 是什么

####  垂直拆分

垂直分表，
  通常是按照业务功能的使用频次，把主要的、热门的字段放在一起做为主要表；

  然后把不常用的，按照各自的业务属性进行聚集，拆分到不同的次要表中；主要表和次要表的关系一般都是一对一的。

#### 水平拆分(数据分片)

mysql单表的容量不超过500W，否则建议水平拆分



#### 切分策略

#### 导航路由

#### 是否有一些开源方案

##### MySQL Fabric

网址：http://www.mysql.com/products/enterprise/fabric.html

 MySQL Fabric 是一个用于管理 MySQL 服务器群的可扩展框架。该框架实现了两个特性 — 高可用性 (HA) 以及使用数据分片的横向扩展
 官方推荐，但是2014年左右才推出，是真正的分表，不是代理的(不同于mysql-proxy)。
未来很有前景，目前属于测试阶段还没大规模运用于生产,期待它的升级。

##### Atlas


Atlas是由 Qihoo 360, Web平台部基础架构团队开发维护的一个基于MySQL协议的数据中间层项目。
它在MySQL官方推出的MySQL-Proxy 0.8.2版本的基础上，修改了大量bug，添加了很多功能特性。目前该项目在360公司内部得到了广泛应用，很多MySQL业务已经接入了Atlas平台，每天承载的读写请求数达几十亿条。

主要功能：

 * 读写分离
 * 从库负载均衡
 * IP过滤
 * SQL语句黑白名单
 * 自动分表，只支持单库多表，不支持分布式分表，同理，该功能我们可以用分库来代替，多库多表搞不定。

网址： https://github.com/Qihoo360/Atlas

##### TDDL

 江湖外号：头都大了
淘宝根据自己的业务特点开发了TDDL（Taobao Distributed Data Layer ）框架，主要解决了分库分表对应用的透明化以及异构数据库之间的数据复制，它是一个基于集中式配置的 jdbc datasource实现，具有主备，读写分离，动态数据库配置等功能。

TDDL所处的位置（tddl通用数据访问层，部署在客户端的jar包，用于将用户的SQL路由到指定的数据库中）：

![image-20191205103623544](H:/Technical-Learning/docs/_images/mysql/tddl.png)

淘宝很早就对数据进行过分库的处理， 上层系统连接多个数据库，中间有一个叫做DBRoute的路由来对数据进行统一访问。DBRoute对数据进行多库的操作、数据的整合，让上层系统像操作 一个数据库一样操作多个库。但是随着数据量的增长，对于库表的分法有了更高的要求，例如，你的商品数据到了百亿级别的时候，任何一个库都无法存放了，于是 分成2个、4个、8个、16个、32个……直到1024个、2048个。好，分成这么多，数据能够存放了，那怎么查询它？这时候，数据查询的中间件就要能 够承担这个重任了，它对上层来说，必须像查询一个数据库一样来查询数据，还要像查询一个数据库一样快（每条查询在几毫秒内完成），TDDL就承担了这样一 个工作。在外面有些系统也用DAL（数据访问层） 这个概念来命名这个中间件。

![image-20191205103631139](H:/Technical-Learning/docs/_images/mysql/TDDL2.png)

 系出名门，淘宝诞生。功能强大，阿里开源（部分）
主要优点：
1.数据库主备和动态切换
2.带权重的读写分离
3.单线程读重试
4.集中式数据源信息管理和动态变更
5.剥离的稳定jboss数据源
6.支持mysql和oracle数据库
7.基于jdbc规范，很容易扩展支持实现jdbc规范的数据源
8.无server,client-jar形式存在，应用直连数据库
9.读写次数,并发度流程控制，动态变更
10.可分析的日志打印,日志流控，动态变更
TDDL必须要依赖diamond配置中心（diamond是淘宝内部使用的一个管理持久配置的系统，目前淘宝内部绝大多数系统的配置，由diamond来进行统一管理，同时diamond也已开源）。
TDDL动态数据源使用示例说明：http://rdc.taobao.com/team/jm/archives/1645
diamond简介和快速使用：http://jm.taobao.org/tag/diamond%E4%B8%93%E9%A2%98/
TDDL源码：https://github.com/alibaba/tb_tddl
TDDL复杂度相对较高。当前公布的文档较少，只开源动态数据源，分表分库部分还未开源，还需要依赖diamond，不推荐使用。

##### MySQL proxy

 官网提供的，小巧精干型的，但是能力有限，对于大数据量的分库分表无能为力，适合中小型的互联网应用，基本上
  mysql-proxy - master/slave就可以构成一个简单版的读写分离和负载均衡



## 分库分表演变过程

单库多表--->读写分离主从复制--->垂直分库，每个库又可以带着salve--->继续垂直分库，极端情况单库单表
  --->分区(变相的水平拆分表，只不过是单库的)--->水平分表后再放入多个数据库里，进行分布式部署

单库多表
读写分离主从复制
垂直分库（每个库又可以带salve）
继续垂直分库，理论上可以到极端情况，单库单表
分区（partition是变相的水平拆分，只不过是单库内进行）
终于到水平分表，后续放入多个数据库里，进行分布式部署，终极method。

但是理论上OK，实际上中间的各种通信、调度、维护和编码要求，更加高



### 分库分表后的难题

 

分布式事务的问题，数据的完整性和一致性问题。
数据操作维度问题：用户、交易、订单各个不同的维度，用户查询维度、产品数据分析维度的不同对比分析角度。
跨库联合查询的问题，可能需要两次查询
跨节点的count、order by、group by以及聚合函数问题，可能需要分别在各个节点上得到结果后在应用程序端进行合并
额外的数据管理负担，如：访问数据表的导航定位
额外的数据运算压力，如：需要在多个节点执行，然后再合并计算
程序编码开发难度提升，没有太好的框架解决，更多依赖业务看如何分，如何合，是个难题。

  



**不到最后一步，轻易不用进行水平分表**

 







异构索引表 



TODO



 https://tech.meituan.com/2016/11/18/dianping-order-db-sharding.html 





















 