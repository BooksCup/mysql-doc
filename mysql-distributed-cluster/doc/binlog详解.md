- [binlog详解](#binlog详解)  
    - [1 binlog的三种模式](#1-binlog的三种模式)  
        - [1.1 row模式](#11-row模式)  
        - [1.2 statement模式](#12-statement模式)  
        - [1.3 mixed模式](#13-mixed模式)  
        - [1.4 binlog三种模式的总结](#14-binlog三种模式的总结)  
    - [2 查看binlog文件的位置](#2-查看binlog文件的位置)  
    - [3 查看当前mysql的binlog情况](#3-查看当前mysql的binlog情况)  
        - [3.1 通过重启mysql服务生成新的binlog文件](#31-通过重启mysql服务生成新的binlog文件)  
        - [3.2 手动生成新的binlog文件](#32-手动生成新的binlog文件)  
        - [3.3 重置binlog文件](#33-重置binlog文件)  
        - [3.4 查看binlog日志](#34-查看binlog日志)    
    - [4 总结](#4-总结)  

# binlog详解  
## 1 binlog的三种模式
### 1.1 row模式
日志会记录成每一行数据被修改成的形式，然后在slave端再对相同的数据进行修改，只记录要修改的数据，只有value，不会有mysql多表关联的情况。  
优点: 在row模式下，binlog中可以不记录执行的sql语句的上下文相关信息，仅仅需要记录哪一条记录被修改了，修改成什么样了，
所以row的日志内容会非常清楚的记录下每一行数据修改的细节，非常容易理解。
而且不会出现在某些特定情况下的存储过程，函数和触发器在调用或触发时无法被正确复制的问题。  
缺点: 在row模式下，所有执行的语句当记录到日志中的时候，都将以每行记录的修改来记录，这样可能会产生大量的日志内容。
比如有这么一条update语句: update test set name = "tiny"，这条语句不是记录的一条，而是修改的每一条都会记录下来。
***也就是说如果test表有1w条数据，row模式下就会记录1w次***。  

### 1.2 statement模式
每一条修改数据的sql都会记录到master的binlog中，slave在复制的时候sql进程会解析成和原来master端相同的sql再执行。  
优点: 在statement模式下，首先解决了row模式下的缺点，不需要记录每一行日志的变化，减少binlog的日志量，节省了I/O及存储资源，提高了性能。
因为只需要记录在master上所执行的语句的细节以及执行语句时候的上下文信息。  
缺点: 在statement模式下，由于它记录的执行语句，所以，为了让这些语句在slave端也能正确执行，那么它还必须记录每条语句在执行时候的一些相关信息，
也就是上下文信息，以保证所有语句在slave端被执行的时候能够得到在master端执行的结果。另外，由于MySQL现在发展较快，很多新功能不断加入，
使得MySQL的复制遇到了不少的挑战，自然复制的时候涉及到的内容越复杂，BUG也容易出现。  
在statement模式下，目前已经发现不少情况会造成MySQL的复制出现问题，主要是修改数据的时候使用了某些特定的函数或者功能。
比如: sleep()函数在有些版本中就不能直接复制，在存储过程中使用了last_insert_id()函数，可能会使slave和master上得到不一致的id等等。
由于row level是基于每一行来记录的变化，所以不会出现类似的问题。  

### 1.3 mixed模式
从官方文档中看到，之前的MySQL一直都只有基于statement的复制模式，知道5.1.5版本的MySQL才开始支持row模式。
从5.0开始，MySQL的复制已经解决了大量老版本中出现的无法正确复制的问题。
但是由于存储过程的出现，给MySQL replication又带来了更大的挑战。
另外，看到官方文档说，从5.1.8版本开始，MySQL提供了除statement和row之外的第三种模式：mixed，实际上就是前两种模式的结合。
在mixed模式下，MySQL会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也就是在statement和row之间选择一种。
新版本中的statement还是和以前一样，仅仅记录执行的语句。
而新版本的MySQL中对row模式也做了优化，并不是所有的修改都会以row模式来记录，比如遇到表结构变更的时候就会以statement模式来记录，
如果sql语句确实是update或者delete等修改数据的语句，那么还是会记录所有行的变更。  

### 1.4 binlog三种模式的总结
row:  
优点: 记录数据详细(每行)，主从一致  
缺点: 占用大量的磁盘空间，降低了磁盘的性能  
100w条记录  
update test set name = 'tiny';  
binlog里面就有100w条 update test set name = 'tiny';语句  

statement:  
优点: 记录的简单，内容少  
缺点: 可能导致主从不一致  
100w条记录  
update test set name = 'tiny';  
binlog里面就只有1条 update test set name = 'tiny';语句  

mixed:  
100w条记录  
update test set name = 'tiny';  
binlog里面就只有1条 update test set name='tiny';  
对于函数，触发器，存储过程  
会自动的使用row模式  

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