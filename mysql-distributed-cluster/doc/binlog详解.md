# binlog详解  
## 1.查看binlog文件的位置
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

## 2.查看当前mysql的binlog情况
```mysql
show master status;
```
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :--- | :--- | :--- | :--- | :--- |
| mysql-bin.000001 | 154 |  |  |  |  

### 2.1 通过重启mysql服务生成新的binlog文件
每当我们重启mysql服务一次，会自动生成一个binlog文件，我们重启完毕之后执行相同的命令
```mysql
show master status;
```
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :--- | :--- | :--- | :--- | :--- |
| mysql-bin.000002 | 154 |  |  |  |  

存放binlog的目录下也多了这么一个文件。

### 2.2 手动生成新的binlog文件
我们也可以手动刷新binlog文件，通过flush logs来创建一个binlog文件。  
```mysql
flush logs;  
show master status;
```
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :--- | :--- | :--- | :--- | :--- |
| mysql-bin.000003 | 154 |  |  |  |  

### 2.3 重置binlog文件  
如果我们想把这些文件全部清空，可以使用reset master来处理。  
```mysql
reset master;  
show master status;
```
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :--- | :--- | :--- | :--- | :--- |
| mysql-bin.000001 | 154 |  |  |  |  