# Mysql 数据库使用

@(工具)[数据库]


-------------------

[TOC]

## mysqldump 
``` bash
# 导出所有数据库
mysqldump -uroot -proot --all-databases >/tmp/all.sql
# 导出db1, db2的所有数据
mysqldump -uroot -proot --databases db1 db2 >/tmp/user.sql
# 导出db1中的a1、a2表
mysqldump -uroot -proot --databases db1 --tables a1 a2  >/tmp/db1.sql
# 只导出表结构不导出数据
mysqldump -uroot -proot --no-data --databases db1 >/tmp/db1.sql
# 跨服务器导出导入数据
mysqldump --host=192.168.80.137 -uroot -proot -C --databases test |mysql --host=192.168.80.133 -uroot -proot test
``` 