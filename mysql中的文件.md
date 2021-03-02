mysql一般目录结构大概如下：
```
.
├── bin
├── COPYING
├── data
├── docs
├── include
├── lib
├── man
├── README
├── share
└── support-files
```

data目录主要是用来存储表结构和表数据，其中特有的文件有:
```
ib_logfile0 日志文件
ib_logfile1
ibdata1  系统表空间文件，存储InnoDB系统信息和用户数据库表数据和索引，所有表共用
ibtmp1   非压缩的innodb临时表的独立表空间
XX数据库1   
XX数据库2
```

每个数据库目录下的内容如下：
```
表1.MYI （MyISAM数据库表文件，即MY Data,表数据文件）
表1.MYD （MyISAM数据库表文件，即MY Index,索引文件）
表1.frm    表结构文件
表1.ibd     表数据文件
表2.frm
表2.ibd
db.opt  是mysql建库过程中自动生成的，用来存储当前数据库的默认字符集和字符校验规则
```

```
cat db.opt
default-character-set=latin1
default-collation=latin1_swedish_ci
```

通过.ibd和.frm恢复mysql数据
https://www.cnblogs.com/meitian/p/9886654.html 
