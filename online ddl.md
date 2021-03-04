#### oneline DDL 
##### 1. mysql各版本对ddl的处理方式是不同的，主要有三种：

1. Copy Table方式： 这是InnoDB最早支持的方式。顾名思义，通过临时表拷贝的方式实现的。新建一个带有新结构的临时表，将原表数据全部拷贝到临时表，然后Rename，完成创建操作。这个方式过程中，原表是可读的，不可写。但是会消耗一倍的存储空间。

2. Inplace方式：这是原生MySQL 5.5，以及innodb_plugin中提供的方式。所谓Inplace，也就是在原表上直接进行，不会拷贝临时表。相对于Copy                        Table方式，这比较高效率。原表同样可读的，但是不可写。

3. Online方式：这是MySQL 5.6以上版本中提供的方式，也是今天我们重点说明的方式。无论是Copy Table方式，还是Inplace方式，原表只能允许读取，           不可写。对应用有较大的限制，因此MySQL最新版本中，InnoDB支持了所谓的Online方式DDL。与以上两种方式相比，online方式支持DDL时不仅可           以读，还可以写，对于dba来说，这是一个非常棒的改进


##### 2. ONLine DDL特性支持了IN-PLACE的表修改和并发DML，有以下好处：

- 提高mysql可用性，在DDL时候，DML没有被阻塞；
- 提供可选项，通过在DDL上使用LOCK语句允许在性能和并发上取得平衡；
- 相对于表COPY方式，更少的磁盘使用率和IO过载；

在缺省情况下，InnoDB会优先使用INPLACE，尽量少的锁进行DDL部署。但用户仍然可以通过在Alter Table使用ALGORITHM和LOCK语句来控制DDL执行，具体语法如：

```
ALTER TABLE tbl_name ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE;   
```
###### 3. ALGORITHM语句和LOCK语句
> 3.1 ALGORITHM语句

对于ALGORITHM语句，InnoDB提供了以下两个方式：

1）ALGORITHM=INPLACE

在执行DDL时不进行复制，而是在已有数据进行，可以避免重建表带来的IO和CPU消耗，保证在执行DDL的时候，DB仍有良好的性能和并发。

2）ALGORITHM=COPY

复制原始表，在执行DDL的时候，并允许DML的写，但是允许DQL查询。在执行复制时候，需要UNDO/REDO log，使得COPY的性能比INPLACE要低；并且对于COPY操作，需要将数据读入Buffer Pool，这增加了从内存中清除频繁访问的数据。而INPLACE方式相对需要读取较少的数据到Buffer Pool中。

> 3.2 LOCK语句

其中，对于LOCK语句，InnoDB提供了以下四种锁力度；

1）LOCK=NONE

不适用锁，表示在执行DDL时候允许并发的DML；

2）LOCK=SHARED

使用共享锁，表示在DDL时候允许执行DQL，但是不允许DML；

3）LOCK=DEFAULT

使用缺省锁，允许尽量多的并发（包括DQL，DML），不写LOCK语句与LOCK=DEFAULT具有相同的效果；

4）LOCK=EXCLUSIVE

使用排他锁，不允许DQL和DML。一般会在期望该语句能够尽快执行，且DQL和DML不需要的时候；或者为了确保Inno服务是空间的，禁止有DB的访问。


##### 4. 在线DDL的限制                         
1）在alter table时，如果涉及到table copy操作，要确保datadir目录有足够的磁盘空间，能够放的下整张表，因为拷贝表的的操作是直接在数据目录下进行的。                                                                   
2）添加索引无需table copy，但要确保tmpdir目录足够存下索引一列的数据（如果是组合索引，当前临时排序文件一合并到原表上就会删除）            3）在主从环境下，主库执行alter命令在完成之前是不会进入binlog记录事件，如果允许dml操作则不影响记录时间，所以期间不会导致延迟。然而，由于从库是单个SQL Thread按顺序应用relay log，轮到ALTER语句时直到执行完才能下一条，所以从库会在master ddl完成后开始产生延迟。（pt-osc可以控制延迟时间，所以这种场景下它更合适）     

4）在每个在线DDL ALTER TABLE语句中，无论LOCK子句是什么，在表的开始和结束都会有一小段时间需要表上的排他锁(与LOCK= exclusive子句指定的锁类型相同)。因此，如果有一个长时间运行的事务在该表上执行插入、更新、删除或选择更新，那么在线DDL操作可能会在开始之前等待;如果在ALTER表正在进行的时候启动了一个类似的长时间运行的事务，那么在线DDL操作可能会等待到完成。                               

5）在执行一个允许并发DML在线 ALTER TABLE时，结束之前这个线程会应用 online log 记录的增量修改，而这些修改是其它thread里产生的，所以有可能会遇到重复键值错误(ERROR 1062 (23000): Duplicate entry)。  

6）涉及到table copy时，目前还没有机制限制暂停ddl，或者限制IO阀值   

7）在MySQL 5.7.6开始能够通过 performance_schema 观察alter table的进度,一般来说，建议把多个alter语句合并在一起进行，避免多次table rebuild带来的消耗。但是也要注意分组，比如需要copy table和只需inplace就能完成的，应该分两个alter语句。

8）如果DDL执行时间很长，期间又产生了大量的dml操作，以至于超过了innodb_online_alter_log_max_size变量所指定的大小，会引起DB_ONLINE_LOG_TOO_BIG 错误。默认为 128M，特别对于需要拷贝大表的alter操作，考虑临时加大该值，以此获得更大的日志缓存空间                                        

9）执行完 ALTER TABLE 之后，最好 ANALYZE TABLE tb1 去更新索引统计信息

##### 5. Online DDL的实现过程               
online ddl主要包括3个阶段，prepare阶段，ddl执行阶段，commit阶段，rebuild方式比no-rebuild方式实质多了一个ddl执行阶段，prepare阶段和commit阶段类似。下面将主要介绍ddl执行过程中三个阶段的流程。


5.1、Prepare阶段:                                                  
```
①：创建新的临时frm文件(与InnoDB无关)     

②：持有EXCLUSIVE-MDL锁，禁止读写   

③：根据alter类型，确定执行方式(copy,online-rebuild,online-norebuild)假如是Add Index，则选择online-norebuild即INPLACE方式    

④：更新数据字典的内存对象                                      

⑤：分配row_log对象记录增量(仅rebuild类型需要)

⑥：生成新的临时ibd文件(仅rebuild类型需要)     
```

5.2、ddl执行阶段:     

```
①：降级EXCLUSIVE-MDL锁，允许读写   

②：扫描old_table的聚集索引每一条记录record

③：遍历新表的聚集索引和二级索引，逐一处理  

④：根据rec构造对应的索引项      

⑤：将构造索引项插入sort_buffer块排序  

⑥：将sort_buffer块更新到新的索引上  

⑦：记录ddl执行过程中产生的增量(仅rebuild类型需要) 

⑧：重放row_log中的操作到新索引上(no-rebuild数据是在原表上更新的)

⑨：重放row_log间产生dml操作append到row_log最后一个Block
```            

5.3、commit阶段:  

```
①：当前Block为row_log最后一个时，禁止读写，升级到EXCLUSIVE-MDL锁  

②：重做row_log中最后一部分增量   

③：更新innodb的数据字典表    

④：提交事务(刷事务的redo日志)   

⑤：修改统计信息        

⑥：rename临时idb文件，frm文件  

⑦：变更完成 
```


```
Prepare阶段：
- 创建新的临时frm文件
- 持有EXCLUSIVE-MDL锁，禁止读写
- 根据alter类型，确定执行方式(copy,online-rebuild,online-norebuild)
- 更新数据字典的内存对象
- 分配row_log对象记录增量
- 生成新的临时ibd文件

ddl执行阶段：
- 降级EXCLUSIVE-MDL锁，允许读写
- 扫描old_table的聚集索引每一条记录rec
- 遍历新表的聚集索引和二级索引，逐一处理
- 根据rec构造对应的索引项
- 将构造索引项插入sort_buffer块
- 将sort_buffer块插入新的索引
- 处理ddl执行过程中产生的增量(仅rebuild类型需要

commit阶段
- 升级到EXCLUSIVE-MDL锁，禁止读写
- 重做最后row_log中最后一部分增量
- 更新innodb的数据字典表
- 提交事务(刷事务的redo日志)
- 修改统计信息
- rename临时idb文件，frm文件
- 变更完成 
```

##### 若干问题

1.如何实现数据完整性
使用online ddl后，用户心中一定有一个疑问，一边做ddl，一边做dml，表中的数据不会乱吗？这里面关键部件是row_log。row_log记录了ddl变更过程中新产生的dml操作，并在ddl执行的最后将其应用到新的表中，保证数据完整性。

2.online与数据一致性如何兼得

实际上，online ddl并非整个过程都是online，在prepare阶段和commit阶段都会持有MDL-Exclusive锁，禁止读写；而在整个ddl执行阶段，允许读写。由于prepare和commit阶段相对于ddl执行阶段时间特别短，因此基本可以认为是全程online的。Prepare阶段和commit阶段的禁止读写，主要是为了保证数据一致性。Prepare阶段需要生成row_log对象和修改内存的字典；Commit阶段，禁止读写后，重做最后一部分增量，然后提交，保证数据一致。

3.如何实现server层和innodb层一致性

在prepare阶段，server层会生成一个临时的frm文件，里面包含了新表的格式；innodb层生成了临时的ibd文件(rebuild方式)；在ddl执行阶段，将数据从原表拷贝到临时ibd文件，并且将row_log增量应用到临时ibd文件；在commit阶段，innodb层修改表的数据字典，然后提交；最后innodb层和mysql层面分别重命名frm和idb文件。

 4.对innodb表做ddl过程中异常了，为啥再次做ddl报#sql-xxx already exists

这个错误是什么鬼？这个表#sql-xxx实质是做ddl产生的临时表，ddl异常退出后(比如进程被kill，或者机器异常掉电等)，临时文件没有清理。再次执行时，会创建同名的#sql-xxx临时文件，从而导致报错。这里的xxx与table-id强相关，如果是这样，我们把这个讨厌的#sql-xxx临时文件删掉如何呢？再次重做ddl发现还是报同样的错误。这主要原因是，这个临时表信息在innodb的数据字典有残留，通过查询数据字典视图information_schema.innodb_sys_tables，可以发现存在一条#sql-xxx的表记录。
深层次原因：ddl整个过程不是原子的，prepare过程中会新建frm文件，ibd文件，并更新数据字典；然后再进行拷贝全量+重放增量操作；最后再rename frm文件，idb文件，并修改数据字典。由于整个过程涉及到server层和innodb层，并不是一个大事务(每次改数据字典都是单独一个事务)，所以执行过程中如果异常终止，就会导致临时表数据字典残留在系统表内。

影响：虽然临时表信息残留在数据字典内，但不影响用户后续操作。

解决方法：由于临时表与table-id强相关，如何改变table-id是我们需要做的，但表又不能被修改，table-id改变不了。这就成了一个悖论，要做ddl，需要改变table-id；要改变table-id，又需要通过ddl操作。查看源码后发现，对于online ddl，临时表名依赖于变更表的table-id(比如#sql-ib79，79就是变更表的table-id)，而对于copy类型(非online)的ddl，临时表名则不依赖于table-id(由mysqld进程号+连接会话号产生，比如sql-604d_2，604d是mysqld进程号，2是会话号)。因此，我们通过copy类型的ddl，就可以产生表名不一样的临时表了，也就可以完成ddl任务了。比如：alter table test_log add column c88 int, ALGORITHM=copy;

其它：ddl异常结束，会导致重做ddl失败。如果做ddl过程中，kill query，这个时候ddl也会退出，但退出前会做好善后工作，清理数据字典，因此再次做ddl不会存在问题。

参考：

https://blog.51cto.com/fengfeng688/1956827

https://zhuanlan.zhihu.com/p/55082835

https://blog.csdn.net/weixin_34228617/article/details/92121131

https://www.docs4dev.com/docs/zh/mysql/5.7/reference/innodb-online-ddl.html

