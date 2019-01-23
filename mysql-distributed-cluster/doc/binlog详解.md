# binlog详解  
## 1 概念
### 1.1 binlog的三种模式
#### 1.1.1 ROW模式
日志会记录成每一行数据被修改成的形式，然后在slave端再对相同的数据进行修改，只记录要修改的数据，只有value，不会有mysql多表关联的情况。  
优点: 在row模式下，binlog中可以不记录执行的sql语句的上下文相关信息，仅仅需要记录哪一条记录被修改了，修改成什么样了，
所以row的日志内容会非常清楚的记录下每一行数据修改的细节，非常容易理解。
而且不会出现在某些特定情况下的存储过程，函数和触发器在调用或触发时无法被正确复制的问题。  
缺点: 在row模式下，所有执行的语句当记录到日志中的时候，都将以每行记录的修改来记录，这样可能会产生大量的日志内容。
比如有这么一条update语句: update user set name = "zhangsan"，这条语句不是记录的一条，而是修改的每一条都会记录下来。
***也就是说如果user表有1w条数据，row模式下就会记录1w次***。  

#### 1.1.2 statement模式
每一条修改数据的sql都会记录到master的binlog中，slave在复制的时候sql进程会解析成和原来master端相同的sql再执行。  
优点: 在statement模式下，首先解决了row模式下的缺点，<font color='red'>不需要记录每一行日志的变化</font>，减少binlog的日志量，节省了I/O及存储资源，提高了性能。
因为只需要记录在master上所执行的语句的细节以及执行语句时候的上下文信息。


## 2 查看binlog文件的位置
```mysql
show variables like '%log_bin%';
```
| Variable_name | Value | 
| :--- | :--- |   
| log_bin | ON | 
| log_bin_basename | /var/lib/mysql/mysql-bin | 
| log_bin_index | /var/lib/mysql/mysql-bin.index | 
| log_bin_trust_function_creators | OFF | 
| log_bin_use_v1_row_events | OFF | 
| sql_log_bin | ON |  

## 3 查看当前mysql的binlog情况
```mysql
show master status;
```
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :--- | :--- | :--- | :--- | :--- |
| mysql-bin.000001 | 154 |  |  |  |  

### 3.1 通过重启mysql服务生成新的binlog文件
每当我们重启mysql服务一次，会自动生成一个binlog文件，我们重启完毕之后执行相同的命令
```mysql
show master status;
```
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :--- | :--- | :--- | :--- | :--- |
| mysql-bin.000002 | 154 |  |  |  |  

存放binlog的目录下也多了这么一个文件。

### 3.2 手动生成新的binlog文件
我们也可以手动刷新binlog文件，通过flush logs来创建一个binlog文件。  
```mysql
flush logs;  
show master status;
```
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :--- | :--- | :--- | :--- | :--- |
| mysql-bin.000003 | 154 |  |  |  |  

### 3.3 重置binlog文件  
如果我们想把这些文件全部清空，可以使用reset master来处理。  
```mysql
reset master;  
show master status;
```
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :--- | :--- | :--- | :--- | :--- |
| mysql-bin.000001 | 154 |  |  |  |  

### 3.4 查看binlog日志
```
mysqlbinlog mysql-bin.000001
```  

## 4 总结  
1.binlog文件会随着服务的启动而创建一个新文件  
2.通过flush logs可以手动刷新日志，生成一个新的binlog文件  
3.通过show master status可以查看binlog的状态    
4.通过reset master可以清空binlog日志文件  
5.通过mysqlbinlog工具可以查看binlog日志的内容  
6.通过执行dml, mysql会自动记录binlog  