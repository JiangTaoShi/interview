## 索引是什么
MySQL官方对索引的定义为：索引(Index)是帮助MySQL高校获取数据的数据结构。 

可以得到索引的本质：索引是数据结构，索引的目的是提高查询效率，可以类比英语新华字典，如果我们要查询MySQL这个单词，首先我们需要在目录（索引）定位到M，然后在定位到y，以此类推找到SQL。 

如果没有索引呢，那就需要从A到Z，去遍历的查找一遍，直到找到我们需要的，一个一个找和直接根据目录定位到数据，是不是差的天壤之别呢，这就是索引的妙用。 

## 索引底层数据结构

当数据量大的时候，索引的数据量也很大，所以索引不可能全部放到内存中，因此索引一般以文件的形式存储到硬盘上。

数据本身之外，数据库还维护着一个满足特定查找算法的数据结构，这些结构以某种方式指向数据，这样就可以基于这些数据结构实现高级查找算法。

## 索引算法种类

B-tree索引（重点掌握，之后文章详细讲解）

Hash索引

full-text索引

R-tree索引

## 索引的优势

类似大学图书馆书目索引，提高数据检索效率，降低数据库IO成本

通过索引列对数据进行排序，降低数据排序成本，降低了CPU消耗

## 索引的劣势

- 实际上索引也是一张表，该表保存了主键和索引字段，并指向实体表的记录，所以索引列也是要占用空间的

- 虽然索引大大提高了查询速度，同时却会降低更新表的速度，如果对表INSERT，，UPDATE和DELETE。因为更新表时，MySQL不仅要不存数据，还要保存一下索引文件每次更新添加了索引列的字段，都会调整因为更新所带来的键值变化后的索引信息

- 索引只是提高效率的一个因素，如果你的MySQL有大数据量的表，就需要花时间研究建立优秀的索引，或优化查询语句

## 索引分类

单值索引：即一个索引只包含单个列，一个表可以有多个单列索引

唯一索引：索引列的值必须唯一，但允许有空值

复合索引：即一个索引包含多个列

## 索引语法

创建一：create [unique] index indexName on tableName (columnName (length) )。

如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是

BLOB和TEXT类型，必须指定length。

创建二：alter tableName add [unique] index [indexName] on (columnName (length) )

删除：DROP INDEX [indexName] ON mytable;

查看：SHOW INDEX FROM table_name\G

![](https://img-blog.csdnimg.cn/2021012921490386.png)

## 哪些情况需要建索引

- 主键自动建立唯一索引

- 频繁作为查询的条件的字段应该创建索引
- 查询中与其他表关联的字段，外键关系建立索引
- 频繁更新的字段不适合创建索引：因为每次更新不单单是更新了记录还会更新索引，加重IO负担
- Where条件里用不到的字段不创建索引
- 单间/组合索引的选择问题（在高并发下倾向创建组合索引）
- 查询中排序的字段，若通过索引去访问将大大提高排序的速度
- 查询中统计或者分组字段

## 哪些不适合建索引

- 表记录太少

- 经常增删改的表

- 数据重复且分布平均的表字段，因此应该只为经常查询和经常排序的数据列建立索引。注意，如果某个数据列包含许多重复的内容，为它建立索引就没有太大的实际效果。

## 一、Explain 用法

Explain + SQL 语句;

如：Explain select \* from user;
会生成如下 SQL 分析结果，下面详细对每个字段进行详解

![](https://img-blog.csdnimg.cn/20210129215053872.png)

## 二、id

是一组数字，代表多个表之间的查询顺序，或者包含子句查询语句中的顺序，id 总共分为三种情况，依次详解

- id 相同，执行顺序由上至下
  
  ![](https://img-blog.csdnimg.cn/20210129215127511.png)

- id 不同，如果是子查询，id 号会递增，id 值越大优先级越高，越先被执行
  
  ![](https://img-blog.csdnimg.cn/20210129215146760.png)

- id 相同和不同的情况同时存在
  
  ![](https://img-blog.csdnimg.cn/202101292152064.png)

## 三、select_type 

select_type 包含以下几种值

- simple
- primary
- subquery
- derived
- union
- union result

### simple

简单的 select 查询，查询中不包含子查询或者 union 查询

![](https://img-blog.csdnimg.cn/20210129215230858.png)

### primary

如果 SQL 语句中包含任何子查询，那么子查询的最外层会被标记为 primary

![](https://img-blog.csdnimg.cn/202101292152467.png)

### subquery

在 select 或者 where 里包含了子查询，那么子查询就会被标记为 subQquery，同三.二同时出现

![](https://img-blog.csdnimg.cn/20210129215310227.png)

### derived

在 from 中包含的子查询，会被标记为衍生查询，会把查询结果放到一个临时表中

![](https://img-blog.csdnimg.cn/20210129215332420.png)

### union / union result 

如果有两个 select 查询语句，他们之间用 union 连起来查询，那么第二个 select 会被标记为 union，union 的结果被标记为 union result。它的 id 是为 null 的

![](https://img-blog.csdnimg.cn/20210129215354818.png)

## 四、table

表示这一行的数据是哪张表的数据

## 五、type

type 是代表 MySQL 使用了哪种索引类型，不同的索引类型的查询效率也是不一样的，type 大致有以下种类

- system
- const
- eq_ref
- ref
- range
- index
- all
  ![](https://img-blog.csdnimg.cn/20210129215421411.png)

### system

表中只有一行记录，system 是 const 的特例，几乎不会出现这种情况，可以忽略不计

### const

将主键索引或者唯一索引放到 where 条件中查询，MySQL 可以将查询条件转变成一个常量，只匹配一行数据，索引一次就找到数据了
![](https://img-blog.csdnimg.cn/20210129215440848.png)

### eq_ref

在多表查询中，如 T1 和 T2，T1 中的一行记录，在 T2 中也只能找到唯一的一行，说白了就是 T1 和 T2 关联查询的条件都是主键索引或者唯一索引，这样才能保证 T1 每一行记录只对应 T2 的一行记录

举个不太恰当的例子，EXPLAIN SELECT \* from t1 , t2 where t1.id = t2.id

![](https://img-blog.csdnimg.cn/20210129215455314.png)

### ref

不是主键索引，也不是唯一索引，就是普通的索引，可能会返回多个符合条件的行。
![](https://img-blog.csdnimg.cn/20210129215512216.png)

### range

体现在对某个索引进行区间范围检索，一般出现在 where 条件中的 between、and、<、>、in 等范围查找中。

![](https://img-blog.csdnimg.cn/20210129215535695.png)

### index

将所有的索引树都遍历一遍，查找到符合条件的行。索引文件比数据文件还是要小很多，所以比不用索引全表扫描还是要快很多。

### all

没用到索引，单纯的将表数据全部都遍历一遍，查找到符合条件的数据

## 六、possible_keys

此次查询中涉及字段上若存在索引，则会被列出来，表示可能会用到的索引，但并不是实际上一定会用到的索引

## 七、key
此次查询中实际上用到的索引

## 八、key_len

表示索引中使用的字节数，通过该属性可以知道在查询中使用的索引长度，注意：这个长度是最大可能长度，并非实际使用长度，在不损失精确性的情况下，长度越短查询效率越高

## 九、ref

显示关联的字段。如果使用常数等值查询，则显示 const，如果是连接查询，则会显示关联的字段。

![](https://img-blog.csdnimg.cn/20210129215605789.png)

- tb_emp 表为非唯一性索引扫描，实际使用的索引列为 idx_name，由于 tb_emp.name='rose'为一个常量，所以 ref=const。

- tb_dept 为唯一索引扫描，从 sql 语句可以看出，实际使用了 PRIMARY 主键索引，ref=db01.tb_emp.deptid 表示关联了 db01 数据库中 tb_emp 表的 deptid 字段。

## 十、rows

根据表信息统计以及索引的使用情况，大致估算说要找到所需记录需要读取的行数，rows 越小越好

## 十一、extra

不适合在其他列显示出来，但在优化时十分重要的信息

### using  fileSort（重点优化）

俗称 " 文件排序 " ，在数据量大的时候几乎是“九死一生”，在 order by 或者在 group by 排序的过程中，order by 的字段不是索引字段，或者 select 查询字段存在不是索引字段，或者 select 查询字段都是索引字段，但是 order by 字段和 select 索引字段的顺序不一致，都会导致 fileSort

![](https://img-blog.csdnimg.cn/20210129215634801.png)

### using temporary（重点优化）

使用了临时表保存中间结果，常见于 order by 和 group by 中。
![](https://img-blog.csdnimg.cn/20210129215654609.png)

### USING index（重点）

表示相应的 select 操作中使用了覆盖索引（Coveing Index）,避免访问了表的数据行，效率不错！
如果同时出现 using where，表明索引被用来执行索引键值的查找；如果没有同时出现 using where，表面索引用来读取数据而非执行查找动作。

![](https://img-blog.csdnimg.cn/20210129215712637.png)

### Using wher

表明使用了 where 过滤

### using join buffer

使用了连接缓存

### impossible where

where 子句的值总是 false，不能用来获取任何元组

### select tables optimized away

在没有 GROUPBY 子句的情况下，基于索引优化 MIN/MAX 操作或者
对于 MyISAM 存储引擎优化 COUNT(\*)操作，不必等到执行阶段再进行计算，
查询执行计划生成的阶段即完成优化。

### distinct

优化 distinct，在找到第一匹配的元组后即停止找同样值的工作

```
【推荐】SQL性能优化的目标：至少要达到 range 级别，要求是ref级别，如果可以是consts最好。 
说明： 
1） consts 单表中最多只有一个匹配行（主键或者唯一索引），在优化阶段即可读取到数据。 
2） ref 指的是使用普通的索引（normal index）。 
3） range 对索引进行范围检索。 
反例：explain表的结果，type=index，索引物理文件全扫描，速度非常慢，这个index级别比较range还低，与全表扫描是小巫见大巫。
```

## 建表—开始索引优化

```sql
// 建表
CREATE TABLE IF NOT EXISTS staffs(
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(24) NOT NULL DEFAULT "" COMMENT'姓名',
    age INT NOT NULL DEFAULT 0 COMMENT'年龄',
    pos VARCHAR(20) NOT NULL DEFAULT "" COMMENT'职位',
    add_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT'入职事件'
) CHARSET utf8 COMMENT'员工记录表';

// 插入数据
INSERT INTO `test`.`staffs` (`name`, `age`, `pos`, `add_time`) VALUES ('z3', 22, 'manager', now());
INSERT INTO `test`.`staffs` (`name`, `age`, `pos`, `add_time`) VALUES ('July', 23, 'dev', now());
INSERT INTO `test`.`staffs` (`name`, `age`, `pos`, `add_time`) VALUES ('2000', 23, 'dev', now());

// 建立复合索引（即一个索引包含多个字段）
ALTER TABLE staffs ADD INDEX idx_staffs_nameAgePos(name, age, pos);
```

## 优化一：全部用到索引

### 介绍

建立的复合索引包含了几个字段，查询的时候最好能全部用到，而且严格按照索引顺序，这样查询效率是最高的。（最理想情况，具体情况具体分析）

### SQL 案例

![](https://img-blog.csdnimg.cn/20210206120408566.png)

## 优化二：最左前缀法则

### 介绍

如果建立的是复合索引，索引的顺序要按照建立时的顺序，即从左到右，如：a->b->c（和 B+树的数据结构有关）

### 无效索引举例

- a->c：a 有效，c 无效
- b->c：b、c 都无效
- c：c 无效

### SQL 案例

![](https://img-blog.csdnimg.cn/20210206120447626.png)

## 优化三：不要对索引做以下处理

### 以下用法会导致索引失效

- 计算，如：+、-、\*、/、!=、<>、is null、is not null、or
- 函数，如：sum()、round()等等
- 手动/自动类型转换，如：id = "1"，本来是数字，给写成字符串了

### SQL 案例

![](https://img-blog.csdnimg.cn/20210206120509150.png)

## 优化四：索引不要放在范围查询右边

### 举例

比如复合索引：a->b->c，当 where a="" and b>10 and 3=""，这时候只能用到 a 和 b，c 用不到索引，因为在范围之后索引都失效（和 B+树结构有关）

### SQL 案例

![](https://img-blog.csdnimg.cn/202102061205323.png)

## 优化五：减少 select \* 的使用

### 使用覆盖索引

即：select 查询字段和 where 中使用的索引字段一致。

### SQL 案例

![](https://img-blog.csdnimg.cn/20210206120555171.png)

## 优化六：like 模糊搜索

### 失效情况

- like "%张三%"
- like "%张三"

### 解决方案

- 使用复合索引，即 like 字段是 select 的查询字段，如：select name from table where name like "%张三%"
- 使用 like "张三%"

### SQL 案例

![](https://img-blog.csdnimg.cn/20210206120624131.png)

## 优化七：order by 优化

当查询语句中使用 order by 进行排序时，如果没有使用索引进行排序，会出现 filesort 文件内排序，这种情况在数据量大或者并发高的时候，会有性能问题，需要优化。

### filesort 出现的情况举例

- order by 字段不是索引字段
- order by 字段是索引字段，但是 select 中没有使用覆盖索引，如：`select * from staffs order by age asc;`
- order by 中同时存在 ASC 升序排序和 DESC 降序排序，如：`select a, b from staffs order by a desc, b asc;`
- order by 多个字段排序时，不是按照索引顺序进行 order by，即不是按照最左前缀法则，如：`select a, b from staffs order by b asc, a asc;`

### 索引层面解决方法

- 使用主键索引排序
- 按照最左前缀法则，并且使用覆盖索引排序，多个字段排序时，保持排序方向一致
- 在 SQL 语句中强制指定使用某索引，force index(索引名字)
- 不在数据库中排序，在代码层面排序

### order by 排序算法

- 双路排序
  
  > Mysql4.1 之前是使用双路排序，字面的意思就是两次扫描磁盘，最终得到数据，读取行指针和 ORDER BY 列，对他们进行排序，然后扫描已经排好序的列表，按照列表中的值重新从列表中读取对数据输出。也就是从磁盘读取排序字段，在 buffer 进行排序，再从磁盘读取其他字段。

文件的磁盘 IO 非常耗时的，所以在 Mysql4.1 之后，出现了第二种算法，就是单路排序。

- 单路排序
  > 从磁盘读取查询需要的所有列，按照 orderby 列在 buffer 对它们进行排序，然后扫描排序后的列表进行输出，
  > 它的效率更快一些，避免了第二次读取数据，并且把随机 IO 变成顺序 IO，但是它会使用更多的空间，
  > 因为它把每一行都保存在内存中了。

当我们无可避免要使用排序时，索引层面没法在优化的时候又该怎么办呢？尽可能让 MySQL 选择使用第二种单路算法来进行排序。这样可以减少大量的随机 IO 操作,很大幅度地提高排序工作的效率。下面看看单路排序优化需要注意的点

### 单路排序优化点

- 增大 max_length_for_sort_data

  > 在 MySQL 中,决定使用"双路排序"算法还是"单路排序"算法是通过参数 max*length_for* sort_data 来决定的。当所有返回字段的最大长度小于这个参数值时,MySQL 就会选择"单路排序"算法,反之,则选择"多路排序"算法。所以,如果有充足的内存让 MySQL 存放须要返回的非排序字段,就可以加大这个参数的值来让 MySQL 选择使用"单路排序"算法。

- 去掉不必要的返回字段，避免select * 
  
  > 当内存不是很充裕时,不能简单地通过强行加大上面的参数来强迫 MySQL 去使用"单路排序"算法,否则可能会造成 MySQL 不得不将数据分成很多段,然后进行排序,这样可能会得不偿失。此时就须要去掉不必要的返回字段,让返回结果长度适应 max_length_for_sort_data 参数的限制。
- 增大 sort_buffer_size 参数设置
  
  > 这个值如果过小的话,再加上你一次返回的条数过多,那么很可能就会分很多次进行排序,然后最后将每次的排序结果再串联起来,这样就会更慢,增大 sort_buffer_size 并不是为了让 MySQL 选择"单路排序"算法,而是为了让 MySQL 尽量减少在排序过程中对须要排序的数据进行分段,因为分段会造成 MySQL 不得不使用临时表来进行交换排序。

但是sort_buffer_size 不是越大越好：
- Sort_Buffer_Size 是一个 connection 级参数,在每个 connection 第一次需要使用这个 buffer 的时候,一次性分配设置的内存。
- Sort_Buffer_Size 并不是越大越好,由于是 connection 级的参数,过大的设置和高并发可能会耗尽系统内存资源。
- 据说 Sort_Buffer_Size 超过 2M 的时候,就会使用 mmap() 而不是 malloc() 来进行内存分配,导致效率降低。

## 优化八：group by 
其原理也是先排序后分组，其优化方式可参考order by。where高于having,能写在where限定的条件就不要去having限定了。

## 慢查询日志

- 慢查询日志是MySQL提供的一种日志记录，它用来记录查询响应时间超过阀值的SQL语句

- 这个时间阀值通过参数long_query_time设置，如果SQL语句查询时间大于这个值，则会被记录到慢查询日志中，这个值默认是10秒

- MySQL默认不开启慢查询日志，在需要调优的时候可以手动开启，但是多少会对数据库性能有点影响

### 如何开启慢查询日志

查看是否开启了慢查询日志

```sql
SHOW VARIABLES LIKE '%slow_query_log%'
```

用命令方式开启慢查询日志，但是重启MySQL后此设置会失效

```sql
set global slow_query_log = 1
```

永久生效开启方式可以在my.cnf里进行配置，在[mysqld]下新增以下两个参数，重启MySQL即可生效

```sql
slow_query_log=1
slow_query_log_file=日志文件存储路径
```

### 慢查询时间阀值

查看慢查询时间阀值

```sql
SHOW VARIABLES LIKE 'long_query_time%';
```

修改慢查询时间阀值

```sql
set global long_query_time=3;
```

修改后的时间阀值生效

```
需要重新连接或者新开一个回话才能看到修改值。
```

在MySQL配置文件中修改时间阀值

```sql
[mysqld]下配置
slow_query_log=1
slow_query_log_file=日志文件存储路径
long_query_time=3
log_output=FILE
```

### 慢查询日志分析工具

慢查询日志可能会数据量非常大，那么我们如何快速找到需要优化的SQL语句呢，这个神奇诞生了，它就是mysqldumpshow。

### mysqldumpslow --help语法

![](https://img-blog.csdnimg.cn/20210129235733710.png)

通过mysqldumpslow --help可知这个命令是由三部分组成：mysqldumpslow [日志查找选项] [日志文件存储位置]。

### 日志查找选项

- -s：是表示按何种方式排序
- c：访问次数
- l：锁定时间
- r：返回记录
- t：查询时间
- al：平均锁定时间
- ar：平均返回记录数
- at：平均查询时间
- -t：即为返回前面多少条的数据
- -g：后边搭配一个正则匹配模式，大小写不敏感的

### 常用分析语法

查找返回记录做多的10条SQL

```sql
mysqldumpslow -s r -t 10 日志路径
```

查找使用频率最高的10条SQL

```sql
mysqldumpslow -s c -t 10 日志路径
```

查找按照时间排序的前10条里包含左连接的SQL

```sql
mysqldumpslow -s t -t 10 -g "left join" 日志路径
```

通过more查看日志，防止爆屏

```
mysqldumpslow -s r -t 10 日志路径 | more
```

## Show profiles

### 是什么

是`MySQL`提供可以用来分析当前会话中`SQL`语句执行的资源消耗情况。可以用于`SQL`的调优测量。默认情况下，参数处于关闭状态，并保存最近 15 次的运行结果

### 开启 profiles

- 查看是否开启

```sql
show variables like "%profiling%";
```

- 开启

```sql
set profiling = 1;
```

### 开始分析

- 先执行要分析的`SQL`语句
- 执行`show profiles;`会出现如下结果

  ![](https://img-blog.csdnimg.cn/2021012922065542.png)
  
- 分析以上截图某一条`SQL`语法

```sql
show profile type1,type2.. for query Query_ID
```

- 比如我们分析截图中的第`5`条`SQL`语句

```sql
show profile cpu,block io for query 5
```

![](https://img-blog.csdnimg.cn/20210129220737484.png)

### show profile返回结果字段含义
- Status ： sql 语句执行的状态
- Duration: sql 执行过程中每一个步骤的耗时
- CPU_user: 当前用户占有的 cpu
- CPU_system: 系统占有的 cpu
- Block_ops_in : I/O 输入
- Block_ops_out : I/O 输出
### show profile type 选项

- all：显示所有的性能开销信息
- block io：显示块 IO 相关的开销信息
- context switches: 上下文切换相关开销
- cpu：显示 CPU 相关的信息
- ipc：显示发送和接收相关的开销信息
- memory：显示内存相关的开销信息
- page faults：显示页面错误相关开销信息
- source：显示和 Source_function、Source_file、Source_line 相关的开销信息
- swaps：显示交换次数的相关信息
### status出现以下情况的建议
- System lock
> 确认是由于哪个锁引起的，通常是因为MySQL或InnoDB内核级的锁引起的。`建议`：如果耗时较大再关注即可，一般情况下都还好

- Sending data
> `解释`：从server端发送数据到客户端，也有可能是接收存储引擎层返回的数据，再发送给客户端，数据量很大时尤其经常能看见。
`备注`：Sending Data不是网络发送，是从硬盘读取，发送到网络是Writing to net。`建议`：通过索引或加上LIMIT，减少需要扫描并且发送给客户端的数据量

- Sorting result
> 正在对结果进行排序，类似Creating sort index，不过是正常表，而不是在内存表中进行排序。
`建议`：创建适当的索引
- Table lock
> 表级锁，没什么好说的，要么是因为MyISAM引擎表级锁，要么是其他情况显式锁表
- create sort index
> 当前的SELECT中需要用到临时表在进行ORDER BY排序
`建议`：创建适当的索引
- Creating tmp table
> 创建临时表。先拷贝数据到临时表，用完后再删除临时表。消耗内存，数据来回拷贝删除，消耗时间，`建议`：优化索引
- converting HEAP to MyISAM
> 查询结果太大，内存不够，数据往磁盘上搬了。`建议`：优化索引，可以调整max_heap_table_size
- Copying to tmp table on disk
> 把内存中临时表复制到磁盘上，危险！！！`建议`：优化索引，可以调整tmp_table_size参数，增大内存临时表大小

---

## 前言

## 准备工作——数据库锁

### 创建表 tb_innodb_lock

```sql
drop table if exists test_innodb_lock;
CREATE TABLE test_innodb_lock (
    a INT (11),
    b VARCHAR (20)
) ENGINE INNODB DEFAULT charset = utf8;
insert into test_innodb_lock values (1,'a');
insert into test_innodb_lock values (2,'b');
insert into test_innodb_lock values (3,'c');
insert into test_innodb_lock values (4,'d');
insert into test_innodb_lock values (5,'e');
```

### 创建索引

```sql
create index idx_lock_a on test_innodb_lock(a);
create index idx_lock_b on test_innodb_lock(b);
```

## MySQL 各种锁演示

- 先将自动提交事务改成手动提交：`set autocommit=0;`
- 我们启动两个会话窗口 A 和 B，模拟一个抢到锁，一个没抢到被阻塞住了。

### 行锁（写&读）

- A 窗口执行

```sql
update test_innodb_lock set b='a1' where a=1;
```

```sql
SELECT * from test_innodb_lock;
```

![](https://img-blog.csdnimg.cn/20210129220939899.png)

我们可以看到 A 窗口可以看到更新后的结果

- B 窗口执行

```sql
SELECT * from test_innodb_lock;
```

![](https://img-blog.csdnimg.cn/20210129221012768.png)

我们可以看到 B 窗口不能看到更新后的结果，看到的还是老数据，这是因为 a = 1 的这行记录被 A 窗口执行的 SQL 语句抢到了锁，并且没有执行 commit 提交操作。所以窗口 B 看到的还是老数据。这就是 MySQL 隔离级别中的"读已提交"。

- 窗口 A 执行 commit 操作

```sql
COMMIT;
```

- 窗口 B 查询

```sql
SELECT * from test_innodb_lock;
```

![](https://img-blog.csdnimg.cn/20210129221029941.png)

这个时候我们发现窗口 B 已经读取到最新数据了

### 行锁（写&写）

- 窗口 A 执行更新 a = 1 的记录

```sql
update test_innodb_lock set b='a2' where a=1;
```

这时候并没有 commit 提交，锁是窗口 A 持有。

- 窗口 B 也执行更新 a = 1 的记录

```sql
update test_innodb_lock set b='a3' where a=1;
```

![](https://img-blog.csdnimg.cn/20210129221046638.png)

可以看到，窗口 B 一直处于阻塞状态，因为窗口 A 还没有执行 commit，还持有锁。窗口 B 抢不到 a = 1 这行记录的锁，所以一直阻塞等待。

- 窗口 A 执行 commit 操作

```sql
COMMIT;
```

- 窗口 B 的变化

![](https://img-blog.csdnimg.cn/20210129221107940.png)

可以看到这个时候窗口 B 已经执行成功了

### 表锁

当索引失效的时候，行锁会升级成表锁，索引失效的其中一个方法是对索引自动 or 手动的换型。a 字段本身是 integer，我们加上引号，就变成了 String，这个时候索引就会失效了。

- 窗口 A 更新 a = 1 的记录

```sql
update test_innodb_lock set b='a4' where a=1 or a=2;
```

- 窗口 B 更新 a = 2 的记录

```sql
update test_innodb_lock set b='b1' where a=3;
```

![](https://img-blog.csdnimg.cn/20210129221128628.png)

这个时候发现，虽然窗口 A 和 B 更新的行不一样，但是窗口 B 还是被阻塞住了，就是因为窗口 A 的索引失效，导致行锁升级成了表锁，把整个表锁住了，索引窗口 B 被阻塞了。

- 窗口 A 执行 commit 操作

```sql
COMMIT;
```

- 窗口 B 的变化

![](https://img-blog.csdnimg.cn/20210129221148475.png)

可以看到这个时候窗口 B 已经执行成功了

### 间隙锁

- 什么是间隙锁

当我们采用范围条件查询数据时，InnoDB 会对这个范围内的数据进行加锁。比如有 id 为：1、3、5、7 的 4 条数据，我们查找 1-7 范围的数据。那么 1-7 都会被加上锁。2、4、6 也在 1-7 的范围中，但是不存在这些数据记录，这些 2、4、6 就被称为间隙。

- 间隙锁的危害

范围查找时，会把整个范围的数据全部锁定住，即便这个范围内不存在的一些数据，也会被无辜的锁定住，比如我要在 1、3、5、7 中插入 2，这个时候 1-7 都被锁定住了，根本无法插入 2。在某些场景下会对性能产生很大的影响

- 间隙锁演示

我们先把字段 a 的值修改成 1、3、5、7、9

- 窗口 A 更新 a = 1~7 范围的数据

```sql
update test_innodb_lock set b='b5' where a>1 and a<7;
```

- 窗口 B 在 a = 2 的位置插入数据

```sql
insert into test_innodb_lock values(2, "b6");
```

![](https://img-blog.csdnimg.cn/20210129221206470.png)

这个时候发现窗口 B 更新 a = 2 的操作一直在等待，因为 1~7 范围的数据被间隙锁，锁住了。只有等窗口 A 执行 commit，窗口 B 的 a = 2 才能更新成功

### 行锁分析

- 执行 SQL 分析命令

```sql
show status like 'innodb_row_lock%';
```

![](https://img-blog.csdnimg.cn/20210129221226886.png)

- Variable_name 说明

  - Innodb_row_lock_current_waits：当前正在等待锁定的数量。

  - Innodb_row_lock_time：从系统启动到现在锁定的时长。

  - Innodb_row_lock_time_avg：每次等待锁所花平均时间。

  - Innodb_row_lock_time_max：从系统启动到现在锁等待最长的一次所花的时间。

  - Innodb_row_lock_waits：系统启动后到现在总共等待锁的次数。

## show processlist 简介

### 语法

不同用户之间只能查看自己的数据，如果想查看所有的请用管理员查询

```sql
show processlist;
```

### 返回结果字段说明

- id

  > SQL 的 ID 标识，需要 kill 这个 SQL 进程的时候可以使用

- User

  > 当前连接用户

- Host

  > 所属的 IP 和端口

- db

  > 数据库名

- command

  > 连接状态，一般是`休眠`（sleep），`查询`（query），`连接`（connect），如果一条 SQL 语句是`query`状态，而且`time`时间很长，说明存在`问题`

- time

  > 连接状态持续的时间，单位是秒（s）

- state（`重点分析`）

  > 当前 SQL 语句的状态，是优化的重要参数

- info
  
  > 显示当前所执行的 SQL 语句

## state 详解

state 在优化中是很重要的字段，能提供给我们很多这条 SQL 线程的当前状态，帮助我们能定位分析问题。下面列举出 state 的一些常见的字段。

- state

  > `解释`：代表资源未释放，如果通过连接池连接数据库，那么 state 应该是一个稳定的范围。如果有大量的 SQL 请求忘记关闭数据库连接，会造成大量连接请求阻塞，数据库挂掉。

- checking table

  > `解释`：正在检查数据数据表，这个操作是系统自动的

- closing tables

  > `解释`：表示正在将表中修改的数据刷新到磁盘中去，然后关闭用完的表，这是一个很快的操作。

  > `优化建议`：如果这个过程很慢，那就需要看看磁盘是否满了，或者磁盘在进行大量的 IO 操作等等

- connect out

  > `解释`：主从复制里，从服务器正在连接主服务器

- creating tmp table

  > `解释`：正在创建临时表，临时存放查询结果

- copying to tmp table on disk
  > `解释`：当使用 order by、group by 或者 join 查询时，会出创建临时表的情况，当数据太大，会把内存中的临时表数据存储到硬盘上。

  > `优化建议`：一：优化索引，尽量减少创建临时表。二：优化 SQL 语句逻辑,可以用 Java 代码实现部分耗时的 SQL 逻辑。三：可以调节`tmp_table_size`和`max_heap_table_size`两个参数，增大内存中临时表的大小。

- flushing tables

  > 在执行刷新表，等待其他线程关闭数据库表

- killed

  > `解释`：发送了一个 kill 请求给某线程，那么这个线程将会检查 kill 标志位，同时会放弃下一个 kill 请求。MySQL 会在每次的主循环中检查 kill 标志位，不过有些情况下该线程可能会过一小段才能死掉。如果该线程程被其他线程锁住了，那么 kill 请求会在锁释放时马上生效。

- sending data

  > `解释`：这个字段字面上很容易误导人，大部分人觉得他仅仅是发送数据给客户端，但其实是`收集` + `发送`。当 MySQL 使用索引查询完后，得到一堆行的 id，如果有的查询列不在索引中，那么 MySQL 需要到 id 所在的数据行，将数据取出来返回给客户端。

- sorting for group / order
  > `解释`：SQL 语句中使用了 group 和 order 进行排序

  > `优化建议`：如果出现了创建临时表或者文件内排序的情况，比较耗时的情况下需要优化索引

- Waiting for net / reading from net / writing to net
  > `解释`：主要是网络状态的描述，如大量出现，要检查数据库网络连接状态和流量

  > `优化建议`：比如外挂流量攻击数据库时，会导致网络带宽被占满，大量的连接请求打到数据库，造成数据库崩溃，建议进行防流量攻击。

- locked
  > `解释`：SQL 被锁住了，如表锁，行锁，间隙锁等等。

  > `优化建议`：正确使用索引，避免索引失效升级为表锁。使用 innodb 搜索引擎，不要用 myisam。

- Opening tables

  > `解释`：一个 SQL 线程正在尝试打开数据表，这个过程正常的情况是很快的，但是如果有人在 alter table，或者 lock table 语句之前完之前，其他线程无法打开这个数据表。

- Waiting for tables
  > `解释`：该线程得到通知，数据表结构已经被修改了，需要重新打开数据表以取得新的结构。然后，为了能的重新打开数据表，必须等到所有其他线程关闭这个表。

  > 以下几种情况下会产生这个通知：FLUSH TABLES tbl_name、 ALTER TABLE、 RENAME TABLE、 REPAIR TABLE、 ANALYZE TABLE、或 OPTIMIZE TABLE。

- System lock
  
  > `解释`：正在等待取得一个外部的系统锁。如果当前没有运行多个 mysqld 服务器同时请求同一个表，那么可以通过增加--skip-external-locking 参数来禁止外部系统锁。默认情况下这个参数是关闭的。

## 索引底层数据结构为何是B+树

数据库索引的数据结构有很多种，比如：哈希索引、平衡二叉树索引、B树索引、B+树索引等等。

目前最流行的是B+树索引，那大家有没有想过为什么是B+树索引最流行，为什么其他索引应用不广泛。

### 哈希索引

hash大家应该非常的熟悉，就是我们老生常谈的HashMap里用到的技术。Hash索引其检索效率非常高，索引的检索可以一次定位。

可能很多人又有疑问了，既然Hash索引的效率这么高，为什么都用Hash索引而还要使用B-Tree索引呢?

任何事物都是有两面性的，Hash索引也一样，虽然Hash索引效率高，但是Hash索引本身由于其特殊性也带来了很多限制和弊端，主要有以下这些：

**原因一：**

Hash索引不能使用范围查询

Hash索引仅仅能满足"=","IN"和"<=>"查询(注意<>和＜＝＞是不同的操作），不能使用范围查询，例如WHERE price > 100。

由于Hash索引比较的是进行Hash运算之后的Hash值，所以它只能用于等值的过滤，不能用于基于范围的过滤。

**原因二：**

Hash索引不能利用部分索引键查询。

对于复合索引，Hash索引在计算Hash值的时候，是组合索引键合并后再一起计算Hash值，而不是单独计算Hash值。

所以通过复合索引的前面一个或几个索引键进行查询的时候，Hash索引也无法被利用。

**原因三：**

Hash索引在任何时候都不能避免表扫描。

Hash索引是将索引键通过Hash运算之后，将 Hash运算结果的Hash值和所对应的行指针信息存放于一个Hash表中。

由于不同索引键存在相同Hash值，所以无法从Hash索引中直接完成查询，还是要通过访问表中的实际数据进行相应的比较，并得到相应的结果。

hash索引out出局

### 平衡二叉树索引

![](https://img-blog.csdnimg.cn/20210129222012968.png)

又称 AVL树。 它除了具备二叉查找树的基本特征之外，还具有一个非常重要的特点：它的左子树和右子树都是平衡二叉树。

且左子树和右子树的深度之差的绝对值（平衡因子 ）不超过1。也就是说AVL树每个节点的平衡因子只可能是-1、0和1（左子树高度减去右子树高度）。

**被淘汰的原因**

- 树的高度过高，高度越高，查找速度越慢

- 他支持范围查找，但是他需要在进行回旋查找

比如我要找到大于5的数据

第一步我先定位到5，然后在树上按照二叉树规则去回旋查找大于5其他数据6、7、8、9、10。。。

如果大于5的数据很多，那速度是很慢的。

### B树索引

![](https://img-blog.csdnimg.cn/20210129222110569.png)

大家可以看到B树和二叉树最大的区别在于：它一个节点可以存储两个值，这就意味着它的树高度，比二叉树的高度更低，它的查询速度就更快。这是他的优点

那为什么最终还是不用它呢，还是因为他在范围查找的时候，存在回旋查询的问题。同样order by排序的时候效率也很低，因为要把树上的数据手动排序一遍。

### 终极大佬：B+树

![](https://img-blog.csdnimg.cn/20210129222141402.png)

它是B数的升级版，B+树相比B树，新增叶子节点与非叶子节点关系。

叶子节点中包含了key和value，key存储的是1-10这些数字，value存储的是数据存储地址，非叶子节点中只是包含了key，不包含value。

所有相邻的叶子节点包含非叶子节点，使用链表进行结合，有一定顺序排序，从而范围查询效率非常高。

**比如我们要查找大于5的数据：**

- 首先我们定位到5的位置

- 然后直接将5后面的数据全部拿出来即可，因为这是有序链表，已经排好序了

我们在order by排序的时候为什么要使用索引进行排序，原因就在这。

## 索引为什么会失效

### 单值索引B+树图

单值索引在B+树的结构里，一个节点只存一个键值对

![](https://img-blog.csdnimg.cn/20210129222627346.png)

### 联合索引

开局一张图，由数据库的`a`字段和`b`字段组成一个`联合索引`。

![](https://img-blog.csdnimg.cn/20210129222650447.png)

从本质上来说，联合索引也是一个B+树，和单值索引不同的是，联合索引的键值对不是1，而是大于1个。

### a, b 排序分析

a顺序：1，1，2，2，3，3

b顺序：1，2，1，4，1，2

大家可以发现a字段是有序排列，b字段是无序排列（因为B+树只能选一个字段来构建有序的树）

一不小心又会发现，在a相等的情况下，b字段是有序的。

大家想想平时编程中我们要对两个字段排序，是不是先按照第一个字段排序，如果第一个字段出现相等的情况，就用第二个字段排序。这个排序方式同样被用到了B+树里。

### 分析最佳左前缀原理

#### 先举一个遵循最佳左前缀法则的例子

```sql
select * from testTable where a=1 and b=2
```

**`分析如下：`**

首先a字段在B+树上是有序的，所以我们可以通过二分查找法来定位到a=1的位置。

其次在a确定的情况下，b是相对有序的，因为有序，所以同样可以通过二分查找法找到b=2的位置。

#### 再来看看不遵循最佳左前缀的例子

```sql
select * from testTable where b=2
```

**`分析如下：`**

我们来回想一下b有顺序的前提：在a确定的情况下。

现在你的a都飞了，那b肯定是不能确定顺序的，在一个无序的B+树上是无法用二分查找来定位到b字段的。

所以这个时候，是用不上索引的。大家懂了吗？

### 范围查询右边失效原理

#### 举例

```sql
select * from testTable where a>1 and b=2
```

**`分析如下：`**

首先a字段在B+树上是有序的，所以可以用二分查找法定位到1，然后将所有大于1的数据取出来，a可以用到索引。

b有序的前提是a是确定的值，那么现在a的值是取大于1的，可能有10个大于1的a，也可能有一百个a。

大于1的a那部分的B+树里，b字段是无序的（开局一张图），所以b不能在无序的B+树里用二分查找来查询，b用不到索引。

### like索引失效原理

```sql
where name like "a%"

where name like "%a%"

where name like "%a"
```

**我们先来了解一下%的用途**

- `%放在右边`，代表查询以"a"开头的数据，如：abc

- `两个%%`，代表查询数据中包含"a"的数据，如：cab、cba、abc

- `%放在左边`，代表查询以"a"为结尾的数据，如cba

#### 为什么%放在右边有时候能用到索引

- %放右边叫做：`前缀`

- %%叫做：`中缀`

- %放在左边叫做：`后缀`

没错，这里依然是最佳左前缀法则这个概念

![](https://img-blog.csdnimg.cn/20210129222728891.png)

大家可以看到，上面的B+树是由字符串组成的。

`字符串的排序方式`：先按照第一个字母排序，如果第一个字母相同，就按照第二个字母排序。。。以此类推

**`开始分析`**

**一、%号放右边（前缀）**

由于B+树的索引顺序，是按照首字母的大小进行排序，前缀匹配又是匹配首字母。所以可以在B+树上进行有序的查找，查找首字母符合要求的数据。所以有些时候可以用到索引。

**二、%号放右边**

是匹配字符串尾部的数据，我们上面说了排序规则，尾部的字母是没有顺序的，所以不能按照索引顺序查询，就用不到索引。

**三、两个%%号**

这个是查询任意位置的字母满足条件即可，只有首字母是进行索引排序的，其他位置的字母都是相对无序的，所以查找任意位置的字母是用不上索引的。

### 总结

这里把一些经典的索引失效案例给大家分析了，希望能引发大家的思考，能够通过这些案例，明白其他情况索引失效的原理。

之后我们在讲讲，如何通过索引查询到数据整个流程，`InnoDB`和`MyISAM`两个引擎底层索引的实现区别。

## B+树在满足聚簇索引和覆盖索引的时候不需要回表查询数据，

在B+树的索引中，叶子节点可能存储了当前的key值，也可能存储了当前的key值以及整行的数据，这就是聚簇索引和非聚簇索引。 在InnoDB中，只有主键索引是聚簇索引，如果没有主键，则挑选一个唯一键建立聚簇索引。如果没有唯一键，则隐式的生成一个键来建立聚簇索引。

当查询使用聚簇索引时，在对应的叶子节点，可以获取到整行数据，因此不用再次进行回表查询。

## 什么是聚簇索引？何时使用聚簇索引与非聚簇索引

- 聚簇索引：将数据存储与索引放到了一块，找到索引也就找到了数据
- 非聚簇索引：将数据存储于索引分开结构，索引结构的叶子节点指向了数据的对应行，myisam通过key\_buffer把索引先缓存到内存中，当需要访问数据时（通过索引访问数据），在内存中直接搜索索引，然后通过索引找到磁盘相应数据，这也就是为什么索引不在key buffer命中时，速度慢的原因

澄清一个概念：innodb中，在聚簇索引之上创建的索引称之为辅助索引，辅助索引访问数据总是需要二次查找，非聚簇索引都是辅助索引，像复合索引、前缀索引、唯一索引，辅助索引叶子节点存储的不再是行的物理位置，而是主键值

何时使用聚簇索引与非聚簇索引

![](https://img-blog.csdnimg.cn/20210129232338563.png)

### 非聚簇索引一定会回表查询吗？

不一定，这涉及到查询语句所要求的字段是否全部命中了索引，如果全部命中了索引，那么就不必再进行回表查询。

举个简单的例子，假设我们在员工表的年龄上建立了索引，那么当进行`select age from employee where age < 20`的查询时，在索引的叶子节点上，已经包含了age信息，不会再次进行回表查询。

## 什么是MVCC

全称Multi-Version Concurrency Control，即`多版本并发控制`，主要是为了提高数据库的`并发性能`。以下文章都是围绕InnoDB引擎来讲，因为myIsam不支持事务。

同一行数据平时发生读写请求时，会`上锁阻塞`住。但mvcc用更好的方式去处理读—写请求，做到在发生读—写请求冲突时`不用加锁`。

这个读是指的`快照读`，而不是`当前读`，当前读是一种加锁操作，是`悲观锁`。

那它到底是怎么做到读—写`不用加锁`的，`快照读`和`当前读`又是什么鬼，跟着你们的`贴心老哥`，继续往下看。

### 当前读、快照读都是什么鬼

什么是MySQL InnoDB下的当前读和快照读？

### 当前读

它读取的数据库记录，都是`当前最新`的`版本`，会对当前读取的数据进行`加锁`，防止其他事务修改数据。是`悲观锁`的一种操作。

如下操作都是当前读：

- select lock in share mode (共享锁)

- select for update (排他锁)

- update (排他锁)
- insert (排他锁)
- delete (排他锁)

- 串行化事务隔离级别

### 快照读

快照读的实现是基于`多版本`并发控制，即MVCC，既然是多版本，那么快照读读到的数据不一定是当前最新的数据，有可能是之前`历史版本`的数据。

如下操作是快照读：

- 不加锁的select操作（注：事务级别不是串行化）

### 快照读与mvcc的关系

`MVCCC`是“维持一个数据的多个版本，使读写操作没有冲突”的一个`抽象概念`。

这个概念需要具体功能去实现，这个具体实现就是`快照读`。（具体实现下面讲）

### 数据库并发场景

- `读-读`：不存在任何问题，也不需要并发控制

- `读-写`：有线程安全问题，可能会造成事务隔离性问题，可能遇到脏读，幻读，不可重复读

- `写-写`：有线程安全问题，可能会存在更新丢失问题，比如第一类更新丢失，第二类更新丢失

### MVCC解决并发哪些问题？

mvcc用来解决读—写冲突的无锁并发控制，就是为事务分配`单向增长`的`时间戳`。为每个数据修改保存一个`版本`，版本与事务时间戳`相关联`。

读操作`只读取`该事务`开始前`的`数据库快照`。

**解决问题如下：**

- `并发读-写时`：可以做到读操作不阻塞写操作，同时写操作也不会阻塞读操作。

- 解决`脏读`、`幻读`、`不可重复读`等事务隔离问题，但不能解决上面的`写-写 更新丢失`问题。

**因此有了下面提高并发性能的`组合拳`：**

- `MVCC + 悲观锁`：MVCC解决读写冲突，悲观锁解决写写冲突

- `MVCC + 乐观锁`：MVCC解决读写冲突，乐观锁解决写写冲突

### MVCC的实现原理

它的实现原理主要是`版本链`，`undo日志` ，`Read View `来实现的

### 版本链

我们数据库中的每行数据，除了我们肉眼看见的数据，还有几个`隐藏字段`，得开`天眼`才能看到。分别是`db_trx_id`、`db_roll_pointer`、`db_row_id`。

- db_trx_id

  6byte，最近修改(修改/插入)`事务ID`：记录`创建`这条记录/`最后一次修改`该记录的`事务ID`。
  
- db_roll_pointer（版本链关键）

  7byte，`回滚指针`，指向`这条记录`的`上一个版本`（存储于rollback segment里）

- db_row_id

  6byte，隐含的`自增ID`（隐藏主键），如果数据表`没有主键`，InnoDB会自动以db_row_id产生一个`聚簇索引`。
  
- 实际还有一个`删除flag`隐藏字段, 记录被`更新`或`删除`并不代表真的删除，而是`删除flag`变了

![](https://img-blog.csdnimg.cn/20210129223032180.png)

如上图，`db_row_id`是数据库默认为该行记录生成的`唯一隐式主键`，`db_trx_id`是当前操作该记录的`事务ID`，而`db_roll_pointer`是一个`回滚指针`，用于配合`undo日志`，指向上一个`旧版本`。

每次对数据库记录进行改动，都会记录一条`undo日志`，每条undo日志也都有一个`roll_pointer`属性（INSERT操作对应的undo日志没有该属性，因为该记录并没有更早的版本），可以将这些`undo日志都连起来`，`串成一个链表`，所以现在的情况就像下图一样：

![](https://img-blog.csdnimg.cn/20210129223052605.png)

对该记录每次更新后，都会将旧值放到一条undo日志中，就算是该记录的一个旧版本，随着更新次数的增多，所有的版本都会被`roll_pointer`属性连接成一个`链表`，我们把这个链表称之为`版本链`，版本链的头节点就是当前记录最新的值。另外，每个版本中还包含生成该版本时对应的事务id，这个信息很重要，在根据ReadView判断版本可见性的时候会用到。

### undo日志

Undo log 主要用于`记录`数据被`修改之前`的日志，在表信息修改之前先会把数据拷贝到`undo log`里。

当`事务`进行`回滚时`可以通过undo log 里的日志进行`数据还原`。

**Undo log 的用途**

- 保证`事务`进行`rollback`时的`原子性和一致性`，当事务进行`回滚`的时候可以用undo log的数据进行`恢复`。

- 用于MVCC`快照读`的数据，在MVCC多版本控制中，通过读取`undo log`的`历史版本数据`可以实现`不同事务版本号`都拥有自己`独立的快照数据版本`。

**undo log主要分为两种：**

- insert undo log

  代表事务在insert新记录时产生的undo log , 只在事务回滚时需要，并且在事务提交后可以被立即丢弃

- update undo log（主要）

  事务在进行update或delete时产生的undo log ; 不仅在事务回滚时需要，在快照读时也需要；
  
  所以不能随便删除，只有在快速读或事务回滚不涉及该日志时，对应的日志才会被purge线程统一清除
  
### Read View(读视图)

事务进行`快照读`操作的时候生产的`读视图`(Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个`快照`。

记录并维护系统当前`活跃事务的ID`(没有commit，当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以越新的事务，ID值越大)，是系统中当前不应该被`本事务`看到的`其他事务id列表`。

Read View主要是用来做`可见性`判断的, 即当我们`某个事务`执行`快照读`的时候，对该记录创建一个Read View读视图，把它比作条件用来判断`当前事务`能够看到`哪个版本`的数据，既可能是当前`最新`的数据，也有可能是该行记录的undo log里面的`某个版本`的数据。

**Read View几个属性**

- `trx_ids`: 当前系统活跃(`未提交`)事务版本号集合。

- `low_limit_id`: 创建当前read view 时“当前系统`最大事务版本号`+1”。

- `up_limit_id`: 创建当前read view 时“系统正处于活跃事务`最小版本号`”

- `creator_trx_id`: 创建当前read view的事务版本号；

### Read View可见性判断条件

![](https://img-blog.csdnimg.cn/20210129223124533.png)

- `db_trx_id` < `up_limit_id` || `db_trx_id` == `creator_trx_id`（显示）

  如果数据事务ID小于read view中的`最小活跃事务ID`，则可以肯定该数据是在`当前事务启之前`就已经`存在`了的,所以可以`显示`。
  
  或者数据的`事务ID`等于`creator_trx_id` ，那么说明这个数据就是当前事务`自己生成的`，自己生成的数据自己当然能看见，所以这种情况下此数据也是可以`显示`的。
  
- `db_trx_id` >= `low_limit_id`（不显示）

  如果数据事务ID大于read view 中的当前系统的`最大事务ID`，则说明该数据是在当前read view 创建`之后才产生`的，所以数据`不显示`。如果小于则进入下一个判断
  
- `db_trx_id`是否在`活跃事务`（trx_ids）中

  - `不存在`：则说明read view产生的时候事务`已经commit`了，这种情况数据则可以`显示`。

  - `已存在`：则代表我Read View生成时刻，你这个事务还在活跃，还没有Commit，你修改的数据，我当前事务也是看不见的。

### MVCC和事务隔离级别

上面所讲的`Read View`用于支持`RC`（Read Committed，读提交）和`RR`（Repeatable Read，可重复读）`隔离级别`的`实现`。

### RR、RC生成时机

- `RC`隔离级别下，是每个`快照读`都会`生成并获取最新`的`Read View`；

- 而在`RR`隔离级别下，则是`同一个事务中`的`第一个快照读`才会创建`Read View`, `之后的`快照读获取的都是`同一个Read View`，之后的查询就`不会重复生成`了，所以一个事务的查询结果每次`都是一样的`。

### 解决幻读问题

- `快照读`：通过MVCC来进行控制的，不用加锁。按照MVCC中规定的“语法”进行增删改查等操作，以避免幻读。

- `当前读`：通过next-key锁（行锁+gap锁）来解决问题的。

### RC、RR级别下的InnoDB快照读区别

- 在RR级别下的某个事务的对某条记录的第一次快照读会创建一个快照及Read View， 将当前系统活跃的其他事务记录起来，此后在调用快照读的时候，还是使用的是同一个Read View，所以只要当前事务在其他事务提交更新之前使用过快照读，那么之后的快照读使用的都是同一个Read View，所以对之后的修改不可见；

- 即RR级别下，快照读生成Read View时，Read View会记录此时所有其他活动事务的快照，这些事务的修改对于当前事务都是不可见的。而早于Read View创建的事务所做的修改均是可见

- 而在RC级别下的，事务中，每次快照读都会新生成一个快照和Read View, 这就是我们在RC级别下的事务中可以看到别的事务提交的更新的原因

### 总结

从以上的描述中我们可以看出来，所谓的MVCC指的就是在使用`READ COMMITTD`、`REPEATABLE READ`这两种隔离级别的事务在执行普通的`SEELCT`操作时访问记录的`版本链`的过程，这样子可以使不同事务的`读-写`、`写-读`操作`并发执行`，从而`提升系统性能`。

## 读写分离有哪些解决方案？

读写分离是依赖于主从复制，而主从复制又是为读写分离服务的。因为主从复制要求`slave`不能写只能读（如果对`slave`执行写操作，那么`show slave status`将会呈现`Slave_SQL_Running=NO`，此时你需要按照前面提到的手动同步一下`slave`）。

**方案一**

使用mysql-proxy代理

优点：直接实现读写分离和负载均衡，不用修改代码，master和slave用一样的帐号，mysql官方不建议实际生产中使用

缺点：降低性能， 不支持事务

**方案二**

使用AbstractRoutingDataSource+aop+annotation在dao层决定数据源。  
如果采用了mybatis， 可以将读写分离放在ORM层，比如mybatis可以通过mybatis plugin拦截sql语句，所有的insert/update/delete都访问master库，所有的select 都访问salve库，这样对于dao层都是透明。 plugin实现时可以通过注解或者分析语句是读写方法来选定主从库。不过这样依然有一个问题， 也就是不支持事务， 所以我们还需要重写一下DataSourceTransactionManager， 将read-only的事务扔进读库， 其余的有读有写的扔进写库。

**方案三**

使用AbstractRoutingDataSource+aop+annotation在service层决定数据源，可以支持事务.

缺点：类内部方法通过this.xx\(\)方式相互调用时，aop不会进行拦截，需进行特殊处理。

## 分库分表后面临的问

- **事务支持** 分库分表后，就成了分布式事务了。如果依赖数据库本身的分布式事务管理功能去执行事务，将付出高昂的性能代价； 如果由应用程序去协助控制，形成程序逻辑上的事务，又会造成编程方面的负担。

- **跨库join**

  只要是进行切分，跨节点Join的问题是不可避免的。但是良好的设计和切分却可以减少此类情况的发生。解决这一问题的普遍做法是分两次查询实现。在第一次查询的结果集中找出关联数据的id,根据这些id发起第二次请求得到关联数据。 分库分表方案产品

- **跨节点的count,order by,group by以及聚合函数问题** 这些是一类问题，因为它们都需要基于全部数据集合进行计算。多数的代理都不会自动处理合并工作。解决方案：与解决跨节点join问题的类似，分别在各个节点上得到结果后在应用程序端进行合并。和join不同的是每个结点的查询可以并行执行，因此很多时候它的速度要比单一大表快很多。但如果结果集很大，对应用程序内存的消耗是一个问题。

- **数据迁移，容量规划，扩容等问题** 来自淘宝综合业务平台团队，它利用对2的倍数取余具有向前兼容的特性（如对4取余得1的数对2取余也是1）来分配数据，避免了行级别的数据迁移，但是依然需要进行表级别的迁移，同时对扩容规模和分表数量都有限制。总得来说，这些方案都不是十分的理想，多多少少都存在一些缺点，这也从一个侧面反映出了Sharding扩容的难度。

- **ID问题**

- 一旦数据库被切分到多个物理结点上，我们将不能再依赖数据库自身的主键生成机制。一方面，某个分区数据库自生成的ID无法保证在全局上是唯一的；另一方面，应用程序在插入数据之前需要先获得ID,以便进行SQL路由. 一些常见的主键生成策略

**UUID** 使用UUID作主键是最简单的方案，但是缺点也是非常明显的。由于UUID非常的长，除占用大量存储空间外，最主要的问题是在索引上，在建立索引和基于索引进行查询时都存在性能问题。 **Twitter的分布式自增ID算法Snowflake** 在分布式系统中，需要生成全局UID的场合还是比较多的，twitter的snowflake解决了这种需求，实现也还是很简单的，除去配置信息，核心代码就是毫秒级时间41位 机器ID 10位 毫秒内序列12位。

- 跨分片的排序分页

  般来讲，分页时需要按照指定字段进行排序。当排序字段就是分片字段的时候，我们通过分片规则可以比较容易定位到指定的分片，而当排序字段非分片字段的时候，情况就会变得比较复杂了。为了最终结果的准确性，我们需要在不同的分片节点中将数据进行排序并返回，并将不同分片返回的结果集进行汇总和再次排序，最后再返回给用户。如下图所示：

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200310170753848.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RoaW5rV29u,size_16,color_FFFFFF,t_70)
  
### 大表怎么优化？某个表有近千万数据，CRUD比较慢，如何优化？分库分表了是怎么做的？分表分库了有什么问题？有用到中间件么？他们的原理知道么？

当MySQL单表记录数过大时，数据库的CRUD性能会明显下降，一些常见的优化措施如下：

1.  **限定数据的范围：** 务必禁止不带任何限制数据范围条件的查询语句。比如：我们当用户在查询订单历史的时候，我们可以控制在一个月的范围内。；
2.  **读/写分离：** 经典的数据库拆分方案，主库负责写，从库负责读；
3.  **缓存：** 使用MySQL的缓存，另外对重量级、更新少的数据可以考虑使用应用级别的缓存；

还有就是通过分库分表的方式进行优化，主要有垂直分表和水平分表

1.  **垂直分区：**

    **根据数据库里面数据表的相关性进行拆分。** 例如，用户表中既有用户的登录信息又有用户的基本信息，可以将用户表拆分成两个单独的表，甚至放到单独的库做分库。

    **简单来说垂直拆分是指数据表列的拆分，把一张列比较多的表拆分为多张表。** 如下图所示，这样来说大家应该就更容易理解了。

    ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOC82LzE2LzE2NDA4NDM1NGJhMmUwZmQ?x-oss-process=image/format,png)

    **垂直拆分的优点：** 可以使得行数据变小，在查询时减少读取的Block数，减少I/O次数。此外，垂直分区可以简化表的结构，易于维护。

    **垂直拆分的缺点：** 主键会出现冗余，需要管理冗余列，并会引起Join操作，可以通过在应用层进行Join来解决。此外，垂直分区会让事务变得更加复杂；

    #### 垂直分表

    把主键和一些列放在一个表，然后把主键和另外的列放在另一个表中

    ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2pwZy90dVNhS2M2U2ZQcjh0NFBaVVFJVUszVHl0aWF3T0VRa2dFZnBCTm4xZkNtdEVhMkRaNTlISFNSaWN2SEIzeU43Yk5LY1hkc3NWZGFNb25TOEFKanY5cFdBLzY0MA?x-oss-process=image/format,png)

    ##### 适用场景

    - 1、如果一个表中某些列常用，另外一些列不常用
    - 2、可以使数据行变小，一个数据页能存储更多数据，查询时减少I/O次数

    ##### 缺点

    - 有些分表的策略基于应用层的逻辑算法，一旦逻辑算法改变，整个分表逻辑都会改变，扩展性较差
    - 对于应用层来说，逻辑算法增加开发成本
    - 管理冗余列，查询所有数据需要join操作

2.  **水平分区：**

    **保持数据表结构不变，通过某种策略存储数据分片。这样每一片数据分散到不同的表或者库中，达到了分布式的目的。 水平拆分可以支撑非常大的数据量。**

    水平拆分是指数据表行的拆分，表的行数超过200万行时，就会变慢，这时可以把一张的表的数据拆成多张表来存放。举个例子：我们可以将用户信息表拆分成多个用户信息表，这样就可以避免单一表数据量过大对性能造成影响。

    ![数据库水平拆分](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOC82LzE2LzE2NDA4NGI3ZTllNDIzZTM?x-oss-process=image/format,png)

    水品拆分可以支持非常大的数据量。需要注意的一点是:分表仅仅是解决了单一表数据过大的问题，但由于表的数据还是在同一台机器上，其实对于提升MySQL并发能力没有什么意义，所以 **水平拆分最好分库** 。

    水平拆分能够 **支持非常大的数据量存储，应用端改造也少**，但 **分片事务难以解决** ，跨界点Join性能较差，逻辑复杂。

    《Java工程师修炼之道》的作者推荐 **尽量不要对数据进行分片，因为拆分会带来逻辑、部署、运维的各种复杂度** ，一般的数据表在优化得当的情况下支撑千万以下的数据量是没有太大问题的。如果实在要分片，尽量选择客户端分片架构，这样可以减少一次和中间件的网络I/O。

    #### 水平分表：

    表很大，分割后可以降低在查询时需要读的数据和索引的页数，同时也降低了索引的层数，提高查询次数

    ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2pwZy90dVNhS2M2U2ZQcjh0NFBaVVFJVUszVHl0aWF3T0VRa2dkQVpyU1Y3M2liMWZkRENYS2M3QUd6Wmhid3FjS0ZVWkpGWThwMFZkVXRPM3JNYzZ2eDFBdzVBLzY0MA?x-oss-process=image/format,png)

    ##### 适用场景

    - 1、表中的数据本身就有独立性，例如表中分表记录各个地区的数据或者不同时期的数据，特别是有些数据常用，有些不常用。
    - 2、需要把数据存放在多个介质上。

    ##### 水平切分的缺点

    - 1、给应用增加复杂度，通常查询时需要多个表名，查询所有数据都需UNION操作
    - 2、在许多数据库应用中，这种复杂度会超过它带来的优点，查询时会增加读一个索引层的磁盘次数

    **下面补充一下数据库分片的两种常见方案：**

    - **客户端代理：** **分片逻辑在应用端，封装在jar包中，通过修改或者封装JDBC层来实现。** 当当网的 **Sharding-JDBC** 、阿里的TDDL是两种比较常用的实现。
    - **中间件代理：** **在应用和数据中间加了一个代理层。分片逻辑统一维护在中间件服务中。** 我们现在谈的 **Mycat** 、360的Atlas、网易的DDB等等都是这种架构的实现。
    
### MySQL数据库cpu飙升到500\%的话他怎么处理？

当 cpu 飙升到 500\%时，先用操作系统命令 top 命令观察是不是 mysqld 占用导致的，如果不是，找出占用高的进程，并进行相关处理。

如果是 mysqld 造成的， show processlist，看看里面跑的 session 情况，是不是有消耗资源的 sql 在运行。找出消耗高的 sql，看看执行计划是否准确， index 是否缺失，或者实在是数据量太大造成。

一般来说，肯定要 kill 掉这些线程\(同时观察 cpu 使用率是否下降\)，等进行相应的调整\(比如说加索引、改 sql、改内存参数\)之后，再重新跑这些 SQL。

也有可能是每个 sql 消耗资源并不多，但是突然之间，有大量的 session 连进来导致 cpu 飙升，这种情况就需要跟应用一起来分析为何连接数会激增，再做出相应的调整，比如说限制连接数等

### 大表数据查询，怎么优化

1.  优化shema、sql语句+索引；

2.  第二加缓存，memcached, redis；

3.  主从复制，读写分离；

4.  垂直拆分，根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统；

5.  水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key, 为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改，sql中尽量带sharding key，将数据定位到限定的表上去查，而不是扫描全部的表；

## MySQL主从复制原理以及流程

主从复制：将主数据库中的DDL和DML操作通过二进制日志（BINLOG）传输到从数据库上，然后将这些日志重新执行（重做）；从而使得从数据库的数据与主数据库保持一致。

**主从复制的作用**

1.  主数据库出现问题，可以切换到从数据库。

2.  可以进行数据库层面的读写分离。

3.  可以在从数据库上进行日常备份。

**MySQL主从复制解决的问题**

- 数据分布：随意开始或停止复制，并在不同地理位置分布数据备份
- 负载均衡：降低单个服务器的压力
- 高可用和故障切换：帮助应用程序避免单点失败
- 升级测试：可以用更高版本的MySQL作为从库

## MySQL 主从复制架构

### 单向主从

一个主人，一个仆人

![](https://img-blog.csdnimg.cn/20210130000235920.png)

### 一主多从

一个主人，多个仆人

![](https://img-blog.csdnimg.cn/20210130000324297.png)

### 互为主从

两台机器都可以写入数据，两台机器既是主人，也是仆人

![](https://img-blog.csdnimg.cn/20210130000356446.png)

### 多主多从

![](https://img-blog.csdnimg.cn/20210130000426605.png)

### 级联双主复制逻辑架构

级联复制模式下，部分slave的数据同步不连接主节点，而是连接从节点。因为如果主节点有太多的从节点，就会损耗一部分性能用于replication（复制），那么我们可以让3~5个从节点连接主节点，其它从节点作为二级或者三级与从节点连接，这样不仅可以缓解主节点的压力，并且对数据一致性没有负面影响。

![](https://img-blog.csdnimg.cn/20210130000501273.png)

## MySQL主从复制集群搭建—binlog二进制文件方式

今天我们就来讲讲如何实现`MySQL集群`的搭建。主从复制有两种方式可以实现，`binlog`和`GTID`，这期我们先通过`binlog`方式来实现，下篇我们来讲`binlog`的原理，和注意事项。

## 一主一从集群搭建

### binlog 简介

`Mysql`中有一个`binlog`二进制日志，这个日志会记录下`主服务器`所有修改了的`SQL`语句，`从服务器`把主服务器上的`binlog`二进制日志，在指定的位置开始复制`主服务器`所有修改的语句，在`从服务器`上执行一遍。

简而言之就是，`主服务器`会把`create、update、delete`语句都记录到一个二进制文件中（binlog），`从服务器`读取这个文件，执行一遍文件中记录的`create、update、delete`语句。从而实现主从数据同步。

### 准备工作

- 三台服务器：192.168.216.111、192.168.216.222、192.168.216.333
- 主从和主主我们用 111 和 222 两台机器，111 位主，222 位从。主主时两台机器都为主。双主多从时，333为从
- 服务器环境：采用 Windows 的，因为大多数小伙伴都是用 Windows 系统，方便大家学习，真实企业中用 Linux。
- 三台机器分别装好 MySQL 数据库，并能互相 ping 通。

### 配置主从库 my.ini 或者 my.cnf 文件

`my.ini是Windows系统的，my.cnf是Linux系统的`，我们这期主要以 Windows 系统为例

- 在 111 和 222 的 my.ini 中的[mysqld]节点下配置
  > - server-id = `唯一ID`：主服务器唯一 ID，一般设置为机器 IP 地址后三位
  > - log-bin = `二进制日志文件存放路径`：这个是启动并记录 binlog 日志
  > - log-err = `错误日志路径`（可选）：启动错误日志
  > - read-only = `0`：0是读写都行（主库），1是只读（从库）
  > - binlog-lgnore-db= `数据库名`（可选）：设置不要主从复制的数据库
  > - binlog-do-db = `数据库名`（可选）：需要复制的数据库名

### 111 主库建授权用户给 222 从库

当主库和从库都配置完 my.ini 文件之后，还需要主库建立一个授权用户，让从库能通过这个用户登录到主库上。

- 语法

```sql
111主库执行：
GRANT REPLICATION SLAVE ON *.* TO '用户名'@'从机IP' IDENTIFIED BY '密码';（建立授权用户）
FLUSH PRIVILEGES;(刷新MySQL的系统权限相关表)
```

- 在从机上试试可否连接上主机

```sql
222从库执行：
mysql -h 主机IP -usally -pilovesally
```

- 如果连接失败，看看是不是防火墙的原因，配置防火墙的 IP 规则

### 开始主从复制

- 查看 master 111 主机状态

```sql
show master status;
```

![](https://img-blog.csdnimg.cn/20210130000716576.png)

这里主要看`File`和`Position`两个参数，`File`代表从哪个日志文件里同步数据，`Position`代表从这个文件的什么位置开始同步数据，binlog-do-db 和 binlog-lgnore-db 意思为同步哪几个数据库和不同步哪几个数据库。

- 从库登录主库设置同步数据文件

如果之前做过同步数据，那么请先停止（stop slave;），否则会报错。

```sql
222执行：
MASTER_HOST='主机IP',
MASTER_USER='主机用户名'，
MASTER_PASSWORD='主机密码',
MASTER_LOG_FILE='File名字'，
MASTER_LOG_POS=Position数字；
```

- 启动从库复制功能

```sql
start slave;
```

- 查看从库同步状态

```sql
show slave status\G;
```

![](https://img-blog.csdnimg.cn/20210130000734681.png)

这两个参数都是 YES，说明同步成功！这时候可以插入一些新数据，试试从库能不能同步这些数据！

## 主主复制集群搭建

上面介绍了主从复制的实现方法，我们在主从复制的基础上介绍主主复制（只需要把 111 也变成 222 的从机），把上面讲的`222`从库改成主库，实现`111`和`222`两个库互为主从，不懂的同学可以看看上篇文章的主主复制架构图。

### 从库转主库

- 将上面`222`从库变成主库，在222上执行如下语句，注意这次的从机 IP 是`111`的 IP，因为要互为主备

```sql
GRANT REPLICATION SLAVE ON *.* TO '用户名'@'从机IP' IDENTIFIED BY '密码';（建立授权用户）
FLUSH PRIVILEGES;(刷新MySQL的系统权限相关表)
```

### 配置两个主主数据库 my.ini

- 在[mysqld]下配置如下参数

```sql
auto_increment_increment=2   #步长值auto_imcrement。一般有n台主MySQL就填n
auto_increment_offset=1  #起始值。一般填第n台主MySQL
```

### 111 同步 222 数据

- 查看 master 222 主机状态

```sql
show master status;
```

![](https://img-blog.csdnimg.cn/20210130000750739.png)

- `111`设置同步`222`数据文件

如果之前做过同步数据，那么请先停止（stop slave;），否则会报错。

```sql
MASTER_HOST='主机IP',
MASTER_USER='主机用户名'，
MASTER_PASSWORD='主机密码',
MASTER_LOG_FILE='File名字'，
MASTER_LOG_POS=Position数字；
```

- 查看`111`库同步状态

```sql
show slave status\G;
```

![](https://img-blog.csdnimg.cn/20210130000809454.png)

这两个参数都是 YES，说明同步成功！这时候可以插入一些新数据，看看`111`和`222`两个库能不能互相同步数据。

### 双主多从集群搭建

我们在上面双主集群的基础上，创建双主多从集群，这时候`333`机器就该上场了。因为`111`和`222`机器都是主，那么`333`机器作为从机，随便挂靠在其中一个主机上便可。我们这里选`111`吧。

步骤和第一个主从复制集群搭建的一样，按照上面的操作即可。

当我们做好所有操作之后，在`111`主机上新增数据进行测试，发现`222`和`333`均已同步数据。但是在`222`新增数据测试时，会发现`111`同步了，但是`333`并没有同步。因为`333`是挂在`111`下的从库，所有`222`主机新增数据的时候，`333`并没有同步`222`的数据，这显然是不行的。解决方案很简单，在两台主机 111 和 222 的配置文件里都加上如下配置，重启即可。

```sql
log-slave-updates=on
```

从库作为其他从库的主库时，必须添加这个参数才能生效。`111`和`222`互为对方的从库，`333`是`111`的从库，所以`111`和`222`要加上这个参数，大家好好理解一下这个逻辑。

## MySQL主从复制集群—gtid实现详解

### GTID 简介

从 MySQL 5.6.5 版本新增了一种主从复制方式：`GTID`，其全称是`Global Transaction Identifier`，即全局事务标识。通过`GTID`保证每个主库提交的事务在集群中都有唯一的一个`事务ID`。强化了数据库主从的一致性和故障恢复数据的容错能力。在主库宕机发生主从切换的情况下。`GTID`方式可以让其他从库自动找到新主库复制的位置，而且`GTID`可以忽略已经执行过的事务，减少了数据发生错误的概率。

### GTID 组成

`GTID`是对一个已经提交事务的编号，并且是全局唯一的。`GTID`是由`UUID`和`TID`组成的。`UUID`是`MySQL`实例的唯一标识，`TID`代表该实例上已经提交的事务数量，随着事务提交数量递增。

举个例子：`3E11FA47-71CA-11E1-9E33-C80AA9429562:23`，冒号前面是`UUID`，后面是`TID`。

### GTID 工作原理

- 主库 master 提交一个事务时会产生 GTID，并且记录在 binlog 日志中

- 从库 salve I/O 线程读取 master 的 binlog 日志文件，并存储在 slave 的 relay log 中。slave 将 master 的 GTID 这个值，设置到 gtid_next 中，即下一个要读取的 GTID 值。

- slave 读取这个 gtid_next，然后对比 slave 自己的 binlog 日志中是否有这个 GTID

- 如果有这个记录，说明这个 GTID 的事务已经执行过了，可以忽略掉

- 如果没有这个记录，slave 就会执行该 GTID 事务，并记录到 slave 自己的 binlog 日志中。在读取执行事务前会先检查其他 session 持有该 GTID，确保不被重复执行。

- 在解析过程中会判断是否有主键，如果没有就用二级索引，如果没有就用全部扫描。

### GTID实现
### 环境

这里我们准备两台机器，一主一从。
- 主（master）：192.168.216.111
- 从（salve）：192.168.216.222

### master主库配置

在`[mysqld]`下配置，配置完需要重启

```sql
#GTID:
server_id=111  #服务器id，一般设置为机器 IP 地址后三位
gtid_mode=on  #开启gtid模式

#强制gtid一致性，开启后对于特定create table不被支持
enforce_gtid_consistency=on

#binlog
log_bin = 二进制日志文件存放路径
log-slave-updates=true

#强烈建议，其他格式可能造成数据不一致
binlog_format=row

#relay log
skip_slave_start=1
```

### slave从库配置
在`[mysqld]`下配置，配置完需要重启

```sql
#GTID:
gtid_mode=on #开启gtid模式
enforce_gtid_consistency=on

#服务器id，一般设置为机器 IP 地址后三位
server_id=222

#binlog
log-bin=slave-binlog
log-slave-updates=true

#强烈建议，其他格式可能造成数据不一致
binlog_format=row

#relay log
skip_slave_start=1
```

### 检查GTID是否开启

```sql
show variables like '%gtid%';
```

![](https://img-blog.csdnimg.cn/20210130001033290.png)

### 主库建立授权用户

```sql
# 建立授权用户
GRANT REPLICATION SLAVE ON *.* TO '用户名'@'从机IP' IDENTIFIED BY '密码';
# 刷新MySQL的系统权限相关表
FLUSH PRIVILEGES;
```
### salve连接到master

```sql
CHANGE MASTER TO  
MASTER_HOST='master的IP',    
MASTER_USER='用户名',    
MASTER_PASSWORD='密码',    
MASTER_PORT=端口号,    
# 1 代表采用GTID协议复制
# 0 代表采用老的binlog复制
MASTER_AUTO_POSITION = 1;
```

### 开启主从复制

```sql
start slave;
```

### 查看slave状态
```sql
show slave status\G
```

### 在master上查看salve信息

```sql
show slave hosts;
```

![](https://img-blog.csdnimg.cn/2021013000105729.png)

至此`GTID`主从复制方式搭建完毕，可以操作主库验证一下从库是否同步了数据。

## ACID特性的实现原理

### 一、基础概念

事务（Transaction）是访问和更新数据库的程序执行单元；事务中可能包含一个或多个sql语句，这些语句要么都执行，要么都不执行。作为一个关系型数据库，MySQL支持事务，本文介绍基于MySQL5.6。

首先回顾一下MySQL事务的基础知识。

**逻辑架构和存储引擎**

![](https://img-blog.csdnimg.cn/20210205002435988.png)

如上图所示，MySQL服务器逻辑架构从上往下可以分为三层：

（1）第一层：处理客户端连接、授权认证等。

（2）第二层：服务器层，负责查询语句的解析、优化、缓存以及内置函数的实现、存储过程等。

（3）第三层：存储引擎，负责MySQL中数据的存储和提取。MySQL中服务器层不管理事务，事务是由存储引擎实现的。MySQL支持事务的存储引擎有InnoDB、NDB Cluster等，其中InnoDB的使用最为广泛；其他存储引擎不支持事务，如MyIsam、Memory等。

如无特殊说明，后文中描述的内容都是基于InnoDB。

**2. 提交和回滚**

典型的MySQL事务是如下操作的：

```sql
start transaction;
……  #一条或多条sql语句
commit;
```

其中start transaction标识事务开始，commit提交事务，将执行结果写入到数据库。如果sql语句执行出现问题，会调用rollback，回滚所有已经执行成功的sql语句。当然，也可以在事务中直接使用rollback语句进行回滚。

**自动提交**

MySQL中默认采用的是自动提交（autocommit）模式，如下所示：

![](https://img-blog.csdnimg.cn/20210205002644197.png)

在自动提交模式下，如果没有start transaction显式地开始一个事务，那么每个sql语句都会被当做一个事务执行提交操作。

通过如下方式，可以关闭autocommit；需要注意的是，autocommit参数是针对连接的，在一个连接中修改了参数，不会对其他连接产生影响。

![](https://img-blog.csdnimg.cn/20210205002704750.png)

如果关闭了autocommit，则所有的sql语句都在一个事务中，直到执行了commit或rollback，该事务结束，同时开始了另外一个事务。

**特殊操作**

在MySQL中，存在一些特殊的命令，如果在事务中执行了这些命令，会马上强制执行commit提交事务；如DDL语句(create table/drop table/alter/table)、lock tables语句等等。

不过，常用的select、insert、update和delete命令，都不会强制提交事务。

**3. ACID特性**

ACID是衡量事务的四个特性：

- 原子性（Atomicity，或称不可分割性）
- 一致性（Consistency）
- 隔离性（Isolation）
- 持久性（Durability）

按照严格的标准，只有同时满足ACID特性才是事务；但是在各大数据库厂商的实现中，真正满足ACID的事务少之又少。例如MySQL的NDB Cluster事务不满足持久性和隔离性；InnoDB默认事务隔离级别是可重复读，不满足隔离性；Oracle默认的事务隔离级别为READ COMMITTED，不满足隔离性……因此与其说ACID是事务必须满足的条件，不如说它们是衡量事务的四个维度。

### 二、原子性

**1. 定义**

原子性是指一个事务是一个不可分割的工作单位，其中的操作要么都做，要么都不做；如果事务中一个sql语句执行失败，则已执行的语句也必须回滚，数据库退回到事务前的状态。

**2. 实现原理：undo log**

在说明原子性原理之前，首先介绍一下MySQL的事务日志。MySQL的日志有很多种，如二进制日志、错误日志、查询日志、慢查询日志等，此外InnoDB存储引擎还提供了两种事务日志：redo log(重做日志)和undo log(回滚日志)。其中redo log用于保证事务持久性；undo log则是事务原子性和隔离性实现的基础。

下面说回undo log。实现原子性的关键，是当事务回滚时能够撤销所有已经成功执行的sql语句。InnoDB实现回滚，靠的是undo log：当事务对数据库进行修改时，InnoDB会生成对应的undo log；如果事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的样子。

undo log属于逻辑日志，它记录的是sql执行相关的信息。当发生回滚时，InnoDB会根据undo log的内容做与之前相反的工作：对于每个insert，回滚时会执行delete；对于每个delete，回滚时会执行insert；对于每个update，回滚时会执行一个相反的update，把数据改回去。

以update操作为例：当事务执行update时，其生成的undo log中会包含被修改行的主键(以便知道修改了哪些行)、修改了哪些列、这些列在修改前后的值等信息，回滚时便可以使用这些信息将数据还原到update之前的状态。

### 三、持久性

**1. 定义**

持久性是指事务一旦提交，它对数据库的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何影响。

**2. 实现原理：redo log**

redo log和undo log都属于InnoDB的事务日志。下面先聊一下redo log存在的背景。

InnoDB作为MySQL的存储引擎，数据是存放在磁盘中的，但如果每次读写数据都需要磁盘IO，效率会很低。为此，InnoDB提供了缓存(Buffer Pool)，Buffer Pool中包含了磁盘中部分数据页的映射，作为访问数据库的缓冲：当从数据库读取数据时，会首先从Buffer Pool中读取，如果Buffer Pool中没有，则从磁盘读取后放入Buffer Pool；当向数据库写入数据时，会首先写入Buffer Pool，Buffer Pool中修改的数据会定期刷新到磁盘中（这一过程称为刷脏）。

Buffer Pool的使用大大提高了读写数据的效率，但是也带了新的问题：如果MySQL宕机，而此时Buffer Pool中修改的数据还没有刷新到磁盘，就会导致数据的丢失，事务的持久性无法保证。

于是，redo log被引入来解决这个问题：当数据修改时，除了修改Buffer Pool中的数据，还会在redo log记录这次操作；当事务提交时，会调用fsync接口对redo log进行刷盘。如果MySQL宕机，重启时可以读取redo log中的数据，对数据库进行恢复。redo log采用的是WAL（Write-ahead logging，预写式日志），所有修改先写入日志，再更新到Buffer Pool，保证了数据不会因MySQL宕机而丢失，从而满足了持久性要求。

既然redo log也需要在事务提交时将日志写入磁盘，为什么它比直接将Buffer Pool中修改的数据写入磁盘(即刷脏)要快呢？主要有以下两方面的原因：

（1）刷脏是随机IO，因为每次修改的数据位置随机，但写redo log是追加操作，属于顺序IO。

（2）刷脏是以数据页（Page）为单位的，MySQL默认页大小是16KB，一个Page上一个小修改都要整页写入；而redo log中只包含真正需要写入的部分，无效IO大大减少。

**3. redo log与binlog**

我们知道，在MySQL中还存在binlog(二进制日志)也可以记录写操作并用于数据的恢复，但二者是有着根本的不同的：

（1）作用不同：redo log是用于crash recovery的，保证MySQL宕机也不会影响持久性；binlog是用于point-in-time recovery的，保证服务器可以基于时间点恢复数据，此外binlog还用于主从复制。

（2）层次不同：redo log是InnoDB存储引擎实现的，而binlog是MySQL的服务器层(可以参考文章前面对MySQL逻辑架构的介绍)实现的，同时支持InnoDB和其他存储引擎。

（3）内容不同：redo log是物理日志，内容基于磁盘的Page；binlog的内容是二进制的，根据binlog_format参数的不同，可能基于sql语句、基于数据本身或者二者的混合。

（4）写入时机不同：binlog在事务提交时写入；redo log的写入时机相对多元：

前面曾提到：当事务提交时会调用fsync对redo log进行刷盘；这是默认情况下的策略，修改innodb_flush_log_at_trx_commit参数可以改变该策略，但事务的持久性将无法保证。

除了事务提交时，还有其他刷盘时机：如master thread每秒刷盘一次redo log等，这样的好处是不一定要等到commit时刷盘，commit速度大大加快。

### 四、隔离性

**1. 定义**

与原子性、持久性侧重于研究事务本身不同，隔离性研究的是不同事务之间的相互影响。隔离性是指，事务内部的操作与其他事务是隔离的，并发执行的各个事务之间不能互相干扰。严格的隔离性，对应了事务隔离级别中的Serializable (可串行化)，但实际应用中出于性能方面的考虑很少会使用可串行化。

隔离性追求的是并发情形下事务之间互不干扰。简单起见，我们主要考虑最简单的读操作和写操作(加锁读等特殊读操作会特殊说明)，那么隔离性的探讨，主要可以分为两个方面：

- (一个事务)写操作对(另一个事务)写操作的影响：锁机制保证隔离性
- (一个事务)写操作对(另一个事务)读操作的影响：MVCC保证隔离性

**2. 锁机制**

首先来看两个事务的写操作之间的相互影响。隔离性要求同一时刻只能有一个事务对数据进行写操作，InnoDB通过锁机制来保证这一点。

锁机制的基本原理可以概括为：事务在修改数据之前，需要先获得相应的锁；获得锁之后，事务便可以修改数据；该事务操作期间，这部分数据是锁定的，其他事务如果需要修改数据，需要等待当前事务提交或回滚后释放锁。

**行锁与表锁**

按照粒度，锁可以分为表锁、行锁以及其他位于二者之间的锁。表锁在操作数据时会锁定整张表，并发性能较差；行锁则只锁定需要操作的数据，并发性能好。但是由于加锁本身需要消耗资源(获得锁、检查锁、释放锁等都需要消耗资源)，因此在锁定数据较多情况下使用表锁可以节省大量资源。MySQL中不同的存储引擎支持的锁是不一样的，例如MyIsam只支持表锁，而InnoDB同时支持表锁和行锁，且出于性能考虑，绝大多数情况下使用的都是行锁。

**如何查看锁信息**

有多种方法可以查看InnoDB中锁的情况，例如：

```sql
select * from information_schema.innodb_locks; #锁的概况
show engine innodb status; #InnoDB整体状态，其中包括锁的情况
```

下面来看一个例子：

```sql
#在事务A中执行：
start transaction;
update account SET balance = 1000 where id = 1;
#在事务B中执行：
start transaction;
update account SET balance = 2000 where id = 1;
```

此时查看锁的情况：

![](https://img-blog.csdnimg.cn/20210205003136877.png)

show engine innodb status查看锁相关的部分：

![](https://img-blog.csdnimg.cn/20210205003154682.png)

通过上述命令可以查看事务24052和24053占用锁的情况；其中lock_type为RECORD，代表锁为行锁(记录锁)；lock_mode为X，代表排它锁(写锁)。

除了排它锁(写锁)之外，MySQL中还有共享锁(读锁)的概念。由于本文重点是MySQL事务的实现原理，因此对锁的介绍到此为止，后续会专门写文章分析MySQL中不同锁的区别、使用场景等，欢迎关注。

介绍完写操作之间的相互影响，下面讨论写操作对读操作的影响。

**3. 脏读、不可重复读和幻读**

首先来看并发情况下，读操作可能存在的三类问题：

（1）脏读：当前事务(A)中可以读到其他事务(B)未提交的数据（脏数据），这种现象是脏读。举例如下（以账户余额表为例）：

![](https://img-blog.csdnimg.cn/2021020500323396.png)

（2）不可重复读：在事务A中先后两次读取同一个数据，两次读取的结果不一样，这种现象称为不可重复读。脏读与不可重复读的区别在于：前者读到的是其他事务未提交的数据，后者读到的是其他事务已提交的数据。举例如下：

![](https://img-blog.csdnimg.cn/20210205003250782.png)

（3）幻读：在事务A中按照某个条件先后两次查询数据库，两次查询结果的条数不同，这种现象称为幻读。不可重复读与幻读的区别可以通俗的理解为：前者是数据变了，后者是数据的行数变了。举例如下：

![](https://img-blog.csdnimg.cn/20210205003308111.png)

**4. 事务隔离级别**

SQL标准中定义了四种隔离级别，并规定了每种隔离级别下上述几个问题是否存在。一般来说，隔离级别越低，系统开销越低，可支持的并发越高，但隔离性也越差。隔离级别与读问题的关系如下：

![](https://img-blog.csdnimg.cn/20210205003336678.png)

在实际应用中，读未提交在并发时会导致很多问题，而性能相对于其他隔离级别提高却很有限，因此使用较少。可串行化强制事务串行，并发效率很低，只有当对数据一致性要求极高且可以接受没有并发时使用，因此使用也较少。因此在大多数数据库系统中，默认的隔离级别是读已提交(如Oracle)或可重复读（后文简称RR）。

可以通过如下两个命令分别查看全局隔离级别和本次会话的隔离级别：

![](https://img-blog.csdnimg.cn/20210205003358861.png)

![](https://img-blog.csdnimg.cn/20210205003414388.png)

InnoDB默认的隔离级别是RR，后文会重点介绍RR。需要注意的是，在SQL标准中，RR是无法避免幻读问题的，但是InnoDB实现的RR避免了幻读问题。

**5. MVCC**

RR解决脏读、不可重复读、幻读等问题，使用的是MVCC：MVCC全称Multi-Version Concurrency Control，即多版本的并发控制协议。下面的例子很好的体现了MVCC的特点：在同一时刻，不同的事务读取到的数据可能是不同的(即多版本)——在T5时刻，事务A和事务C可以读取到不同版本的数据。

![](https://img-blog.csdnimg.cn/20210205003514715.png)

MVCC最大的优点是读不加锁，因此读写不冲突，并发性能好。InnoDB实现MVCC，多个版本的数据可以共存，主要基于以下技术及数据结构：

1）隐藏列：InnoDB中每行数据都有隐藏列，隐藏列中包含了本行数据的事务id、指向undo log的指针等。

2）基于undo log的版本链：前面说到每行数据的隐藏列中包含了指向undo log的指针，而每条undo log也会指向更早版本的undo log，从而形成一条版本链。

3）ReadView：通过隐藏列和版本链，MySQL可以将数据恢复到指定版本；但是具体要恢复到哪个版本，则需要根据ReadView来确定。所谓ReadView，是指事务（记做事务A）在某一时刻给整个事务系统（trx_sys）打快照，之后再进行读操作时，会将读取到的数据中的事务id与trx_sys快照比较，从而判断数据对该ReadView是否可见，即对事务A是否可见。

trx_sys中的主要内容，以及判断可见性的方法如下：

`low_limit_id`：表示生成ReadView时系统中应该分配给下一个事务的id。如果数据的事务id大于等于low_limit_id，则对该ReadView不可见。

`up_limit_id`：表示生成ReadView时当前系统中活跃的读写事务中最小的事务id。如果数据的事务id小于up_limit_id，则对该ReadView可见。

`rw_trx_ids`：表示生成ReadView时当前系统中活跃的读写事务的事务id列表。如果数据的事务id在low_limit_id和up_limit_id之间，则需要判断事务id是否在rw_trx_ids中：如果在，说明生成ReadView时事务仍在活跃中，因此数据对ReadView不可见；如果不在，说明生成ReadView时事务已经提交了，因此数据对ReadView可见。

下面以RR隔离级别为例，结合前文提到的几个问题分别说明。

（1）脏读

![](https://img-blog.csdnimg.cn/2021020500362015.png)

当事务A在T3时刻读取zhangsan的余额前，会生成ReadView，由于此时事务B没有提交仍然活跃，因此其事务id一定在ReadView的rw_trx_ids中，因此根据前面介绍的规则，事务B的修改对ReadView不可见。接下来，事务A根据指针指向的undo log查询上一版本的数据，得到zhangsan的余额为100。这样事务A就避免了脏读。

（2）不可重复读

![](https://img-blog.csdnimg.cn/20210205003640489.png)

当事务A在T2时刻读取zhangsan的余额前，会生成ReadView。此时事务B分两种情况讨论，一种是如图中所示，事务已经开始但没有提交，此时其事务id在ReadView的rw_trx_ids中；一种是事务B还没有开始，此时其事务id大于等于ReadView的low_limit_id。无论是哪种情况，根据前面介绍的规则，事务B的修改对ReadView都不可见。

当事务A在T5时刻再次读取zhangsan的余额时，会根据T2时刻生成的ReadView对数据的可见性进行判断，从而判断出事务B的修改不可见；因此事务A根据指针指向的undo log查询上一版本的数据，得到zhangsan的余额为100，从而避免了不可重复读。

（3）幻读

![](https://img-blog.csdnimg.cn/20210205003708754.png)

MVCC避免幻读的机制与避免不可重复读非常类似。

当事务A在T2时刻读取0<id<5的用户余额前，会生成ReadView。此时事务B分两种情况讨论，一种是如图中所示，事务已经开始但没有提交，此时其事务id在ReadView的rw_trx_ids中；一种是事务B还没有开始，此时其事务id大于等于ReadView的low_limit_id。无论是哪种情况，根据前面介绍的规则，事务B的修改对ReadView都不可见。

当事务A在T5时刻再次读取0<id<5的用户余额时，会根据T2时刻生成的ReadView对数据的可见性进行判断，从而判断出事务B的修改不可见。因此对于新插入的数据lisi(id=2)，事务A根据其指针指向的undo log查询上一版本的数据，发现该数据并不存在，从而避免了幻读。

**扩展**

前面介绍的MVCC，是RR隔离级别下“非加锁读”实现隔离性的方式。下面是一些简单的扩展。

（1）读已提交（RC）隔离级别下的非加锁读

RC与RR一样，都使用了MVCC，其主要区别在于：

RR是在事务开始后第一次执行select前创建ReadView，直到事务提交都不会再创建。根据前面的介绍，RR可以避免脏读、不可重复读和幻读。

RC每次执行select前都会重新建立一个新的ReadView，因此如果事务A第一次select之后，事务B对数据进行了修改并提交，那么事务A第二次select时会重新建立新的ReadView，因此事务B的修改对事务A是可见的。因此RC隔离级别可以避免脏读，但是无法避免不可重复读和幻读。

（2）加锁读与next-key lock

按照是否加锁，MySQL的读可以分为两种：

一种是非加锁读，也称作快照读、一致性读，使用普通的select语句，这种情况下使用MVCC避免了脏读、不可重复读、幻读，保证了隔离性。

另一种是加锁读，查询语句有所不同，如下所示：

```sql
#共享锁读取
select...lock in share mode
#排它锁读取
select...for update
```

加锁读在查询时会对查询的数据加锁（共享锁或排它锁）。由于锁的特性，当某事务对数据进行加锁读后，其他事务无法对数据进行写操作，因此可以避免脏读和不可重复读。而避免幻读，则需要通过next-key lock。next-key lock是行锁的一种，实现相当于record lock(记录锁) + gap lock(间隙锁)；其特点是不仅会锁住记录本身(record lock的功能)，还会锁定一个范围(gap lock的功能)。因此，加锁读同样可以避免脏读、不可重复读和幻读，保证隔离性。

**6. 总结**

概括来说，InnoDB实现的RR，通过锁机制（包含next-key lock）、MVCC（包括数据的隐藏列、基于undo log的版本链、ReadView）等，实现了一定程度的隔离性，可以满足大多数场景的需要。

不过需要说明的是，RR虽然避免了幻读问题，但是毕竟不是Serializable，不能保证完全的隔离，下面是两个例子：

第一个例子，如果在事务中第一次读取采用非加锁读，第二次读取采用加锁读，则如果在两次读取之间数据发生了变化，两次读取到的结果不一样，因为加锁读时不会采用MVCC。

第二个例子，如下所示，大家可以自己验证一下。

![](https://img-blog.csdnimg.cn/20210205003849841.png)

### 五、一致性

**1. 基本概念**

一致性是指事务执行结束后，数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态。数据库的完整性约束包括但不限于：实体完整性（如行的主键存在且唯一）、列完整性（如字段的类型、大小、长度要符合要求）、外键约束、用户自定义完整性（如转账前后，两个账户余额的和应该不变）。

**2. 实现**

可以说，一致性是事务追求的最终目标：前面提到的原子性、持久性和隔离性，都是为了保证数据库状态的一致性。此外，除了数据库层面的保障，一致性的实现也需要应用层面进行保障。

实现一致性的措施包括：

- 保证原子性、持久性和隔离性，如果这些特性无法保证，事务的一致性也无法保证

- 数据库本身提供保障，例如不允许向整形列插入字符串值、字符串长度不能超过列的限制等

- 应用层面进行保障，例如如果转账操作只扣除转账者的余额，而没有增加接收者的余额，无论数据库实现的多么完美，也无法保证状态的一致

### 六、总结

下面总结一下ACID特性及其实现原理：

- 原子性：语句要么全执行，要么全不执行，是事务最核心的特性，事务本身就是以原子性来定义的；实现主要基于undo log

- 持久性：保证事务提交后不会因为宕机等原因导致数据丢失；实现主要基于redo log

- 隔离性：保证事务执行尽可能不受其他事务影响；InnoDB默认的隔离级别是RR，RR的实现主要基于锁机制（包含next-key lock）、MVCC（包括数据的隐藏列、基于undo log的版本链、ReadView）

- 一致性：事务追求的最终目标，一致性的实现既需要数据库层面的保障，也需要应用层面的保障