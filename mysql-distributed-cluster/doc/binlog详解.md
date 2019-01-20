# binlog详解  
## 1.查看binlog文件的位置
```sql
show variables like '%log_bin%';
```
| Variable_name | Value | 
| :---         |     :---      |   
| log_bin | ON | 
| log_bin_basename | /var/lib/mysql/mysql-bin | 
| log_bin_index | /var/lib/mysql/mysql-bin.index | 
| log_bin_trust_function_creators | OFF | 
| log_bin_use_v1_row_events | OFF | 
| sql_log_bin | ON |