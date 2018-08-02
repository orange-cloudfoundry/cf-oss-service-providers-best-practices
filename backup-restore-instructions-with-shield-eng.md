# Operations instructions for SHIELD plugin V6 
- [Plugin MySQL](#plugin-mySQL) 
- [Plugin Xtrabackup](#plugin-xtrabackup)

## Plugin MySQL 

Define MySQLDump option in manifest

```
mysql_host: 127.0.0.1
mysql_port: "3306"
mysql_user: root
mysql_password: ((cf_mysql_mysql_admin_password))
mysql_options: "--flush-logs --add-drop-database --single-transaction  --opt"
mysql_bindir: "/var/vcap/packages/mariadb/bin"
```

### Restore with MySQLDump

Principles :
- Choose the mysql node for restore
- Stop other nodes
- Restore backup
- Resynchronize the other mysql nodes

#### Pre-requisites  

On restore node, modify MySQL system variables  

```sql
set global enforce_storage_engine=NULL;
set global general_log=OFF;
set global slow_query_log=OFF;
```

Check the disk space used by the MySQL logbin

```sql
MariaDB [(none)]> SHOW BINARY LOGS;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000012 |       409 |
| mysql-bin.000013 | 988995366 |
| mysql-bin.000014 | 859562470 |
| mysql-bin.000015 | 122589435 |
| mysql-bin.000016 | 789955446 |
| mysql-bin.000017 |      1991 |
| mysql-bin.000018 | 998122809 |
| mysql-bin.000019 |       366 |
+------------------+-----------+
```

and purge and specifying the name of the last binlog : Becarefull, no recovery possible.

```sql
MariaDB [(none)]> PURGE BINARY LOGS BEFORE now();
Query OK, 0 rows affected (0.02 sec)
```

On other mysql nodes  

```sh
monit stop mariadb_ctrl
```

#### Restore with SHIELD
Start restore

#### Post Shield restore

On restore node, reset the MySQL system variables to their initial values 
 
```sql
MariaDB [(none)]> Set global enforce_storage_engine=InnoDB;
MariaDB [(none)]> Set global general_log=OFF;
MariaDB [(none)]> Set global slow_query_log=ON;
```

On each other nodes, resynchronize MySQL Gaera Cluster

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

## Plugin Xtrabackup
Define Xtrabackup option in manifest

```sh
mysql_user: root
mysql_password: ((cf_mysql_mysql_admin_password))
mysql_datadir: "/var/vcap/store/mysql"
mysql_xtrabackup: "/var/vcap/packages/xtrabackup/bin/xtrabackup"
```

### Restore with Xtrabackup

Principles :
- Choose the mysql node for restore
- Stop other nodes
- Restore backup
- Prepare the restart of MySQL instance
- Resynchronize the other mysql nodes

#### Pre-requisites  
On the restore node :

```sh
monit stop mariadb_ctrl
cd /var/vcap/store/mysql/
rm –Rf *
```

On other mysql nodes  

```sh
monit stop mariadb_ctrl
```
	
#### Restore with SHIELD
Start restore

#### Post Shield restore

On restore node

```sh
echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
chown -R vcap:vcap /var/vcap/store/mysql/*
monit start mariadb_ctrl
```

On each other nodes, resynchronize MySQL Gaera Cluster

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

