> 【共享表空间】

　　又称为system tablespace系统表空间，a small set of data files (the ibdata files) 。

一、共享表空间ibdata1

1、数据字典(data dictionary)：记录数据库相关信息

2、doublewrite write buffer：解决部分写失败（页断裂）

3、insert buffer：内存insert buffer数据，周期写入共享表空间，防止意外宕机

4、回滚段(rollback segments)

5、undo空间：undo页

二、关于ibdata1的增长考虑

　　影响共享表空间增长的对象：insert buffer、undo空间

1、什么时候undo暴涨：

　　1、大事务为主，例如修改了40万行才提交

　　2、长事务导致的undo持续增加

2、insert buffer空间很大

　　1、索引建立过多

　　2、很多索引不怎么使用

　　3、索引和主键顺序严重不一致：主键的建立选择有问题

所以===

　　ibdata1使用时要注意：在初始化ibdata1时，最好是设置大一些，避免由于高并发突然暴增，影响性能。

三、对ibdata的的设置最佳实践

注意：初始化设置，后期设置就没有意义。

1、空间大小

　　innodb表是按表空间进行存放的，共享表空间ibdata1默认初始化大小是12M。


　　初始化：innodb_data_file_path=ibdata1:10G:ibdata2:10G:autoextend

2、undo分离

　　undo最容易导致ibdata1暴涨，所以需要将undo分离出去，只能在初始化的时候做，初始化之后就没有办法实现undo分离。

处理：

　　在初始化实例之前，只需要设置 innodb_undo_tablespaces 参数，默认是等于0即指单独存放的undo表空间个数为0，也就是undo为分离；建议将该参数设置大于等于3，即可将undo log设置到单独的undo表空间中。


> 【独立表空间】

一、innodb_file_per_table=1

　　MySQL 5.6.6之前的版本，InnoDB 默认会将所有的数据库InnoDB引擎的表数据存储在一个共享空间中：ibdata1，这样就会让管理感觉很难受，增删数据库的时候，ibdata1文件不会自动收缩，单个数据库的备份也将成为问题。通常只能将数据使用mysqldump 导出，然后再导入解决这个问题。

　　在之后的版本，为了优化上述问题，独立表空间innodb_file_per_table参数默认开启：


　　可以修改InnoDB为独立表空间模式，每个数据库的每个表都会生成一个数据空间。

二、独立表空间的性能优越

1、优点：

　　1、每个表都有自已独立的表空间。

　　2、每个表的数据和索引都会存在自已的表空间中。

　　3、可以实现单表在不同的数据库中移动。

　　4、空间可以回收

　　5、对于使用独立表空间的表，不管怎么删除，表空间的碎片不会太严重的影响性能，而且还有机会处理。

2、缺点：

　　单表增加比共享空间方式更大。

3、共享表空间在Insert操作上少有优势，其它都没独立表空间表现好。当启用独立表空间时，合理调整：innodb_open_files(指定InnoDB一次可以保持打开的.ibd文件的最大数目)。
