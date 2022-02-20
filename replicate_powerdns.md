# Replication With MySQL on PowerDNS

## MySQL Replication – Master only
These settings apaply to the master only.

In this file:
```
/etc/mysql/my.cnf
```
`my.cnf.template.master`

Then restart MySQL with:
```
systemctl restart mysql
```
Next we need to add the slave user:
```
mysql -u root -p
```
On Mariadb:
```
GRANT REPLICATION SLAVE ON *.* TO 'pdns-secondary'@'<SECONDARY_SERVER_IP>' IDENTIFIED BY '<SECRET_PASSWORD>';
```
On MySQL:
```
CREATE USER 'pdns-secondary'@'<SECONDARY_SERVER_IP>' IDENTIFIED with mysql_native_password BY '<SECRET_PASSWORD>';
GRANT REPLICATION SLAVE ON *.* TO 'pdns-secondary'@'<SECONDARY_SERVER_IP>';
```
Remember to set the secondary IP and the secondary password.
```
FLUSH PRIVILEGES;
```
Lock all tables on data, read only
```
FLUSH TABLES WITH READ LOCK;
```
You can see if its working by running:
```
show master status;
```
Which will give you something like this:
```
+------------------+----------+--------------+------------------+

| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |

+------------------+----------+--------------+------------------+

| mysql-bin.000001 |      634 | powerdns     |                  |

+------------------+----------+--------------+------------------+

1 row in set (0.01 sec)
```
The ‘File’ and ‘Position’ are important, we will come back to them shortly.
## Export & Import database from Master to Slave
### On DB Master:
```
mysqldump -u root -p --databases powerdns > /root/db_dump.sql
```
Copy the db from master to slave:
```
scp /root/db_dump.sql chadmin@192.168.30.4:/home/chadmin/
```
### On DB Slave:
```
cp /home/chadmin/db_dump.sql /root/
mysql -u root -p < /root/db_dump.sql
```
### Unlock DB on Master
```
mysql -u root -p
> UNLOCK TABLES;
> QUIT;
```
## MySQL Replication – Slave server
On the slave server we are again editing the same file:
```
/etc/mysql/my.cnf
```
`my.cnf.template.slave`

Now restart MySQL with:
```
systemtctl restart mysql
```
Now is the time we need those two details from the ‘show master status’ so it would be wise to run it again. Take note of the ‘File’ and the ‘Position’.

On the secondary, log into MySQL.
```
mysql -u root -p
```
Enter the following, its all one line so copy it into notepad, edit it, then paste it into the MySQL console.
```
change master to

master_host='<PRIMARY_SERVER_IP>',

master_user='pdns-secondary',

master_password='<SECRET_PASSWORD>', master_log_file='mysql-bin.000001', master_log_pos=634;
```

Here, change  master_host to the IP of the master server, set the msater_password to the password of the secondary user you created earlier, set the master_log to the ‘file’ from master status, same as the position.

Then start the slave:
```
start slave;
```
You can see the status of the slave with:
```
show slave status\G
```
It will show you an output a bit like this:
```
MariaDB [(none)]> show slave status\G
*************************** 1. row ***************************
Slave_IO_State: Waiting for master to send event
Master_Host: 192.168.0.253
Master_User: pdns-secondary
Master_Port: 3306
Connect_Retry: 60
Master_Log_File: mysql-bin.000002
Read_Master_Log_Pos: 363730
Relay_Log_File: slave-relay-bin.000002
Relay_Log_Pos: 208937
Relay_Master_Log_File: mysql-bin.000002
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Replicate_Do_DB: powerdns
Replicate_Ignore_DB:
Replicate_Do_Table:
Replicate_Ignore_Table:
Replicate_Wild_Do_Table:
Replicate_Wild_Ignore_Table:
Last_Errno: 0
Last_Error:
Skip_Counter: 0
Exec_Master_Log_Pos: 363730
Relay_Log_Space: 209246
Until_Condition: None
Until_Log_File:
Until_Log_Pos: 0
Master_SSL_Allowed: No
Master_SSL_CA_File:
Master_SSL_CA_Path:
Master_SSL_Cert:
Master_SSL_Cipher:
Master_SSL_Key:
Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
Last_IO_Errno: 0
Last_IO_Error:
Last_SQL_Errno: 0
Last_SQL_Error:
Replicate_Ignore_Server_Ids:
Master_Server_Id: 1
Master_SSL_Crl:
Master_SSL_Crlpath:
Using_Gtid: No
Gtid_IO_Pos:
Replicate_Do_Domain_Ids:
Replicate_Ignore_Domain_Ids:
Parallel_Mode: conservative
SQL_Delay: 0
SQL_Remaining_Delay: NULL
Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
Slave_DDL_Groups: 0
Slave_Non_Transactional_Groups: 0
Slave_Transactional_Groups: 182
1 row in set (0.001 sec)
```
