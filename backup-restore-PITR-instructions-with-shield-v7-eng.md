# Operations instructions for SHIELD V7.34 with MySQL / Xtrabackup plugin

The 2 plugins have been improved to minimize the exploitation operations during restore

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

#### Prerequis  

On the restore node : No action to be taken, the plugin makes the necessary modifications of the MySQL system variables.

On other mysql nodes  :

```sh
monit stop mariadb_ctrl
```

#### Restore with SHIELD

Start restore

#### Post Shield restore

- Verify if your database is correct.
- On each other nodes, resynchronize MySQL Galera Cluster

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

## Plugin Xtrabackup

### Restore

Principles :

- Choose the mysql node for restore
- Stop other nodes
- Restore backup
- Prepare the restart of MySQL instance
- Start MySQL
- Resynchronize the other mysql nodes

#### Pre-requisites  

On all the nodes

```sh
monit stop mariadb_ctrl
```

#### Restore with SHIELD

Start restore


#### Post Shield restore

On restore node, start MariaDB Cluster in bootstrap  

```sh
echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
chown vcap:vcap /var/vcap/store/mysql/state.txt
monit start mariadb_ctrl
```

- Verify if your database is correct.
- On each other nodes, resynchronize MySQL Galera Cluster

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

## Point In Time Recovery

In some cases, it can be interesting to restore until a specific date, thanks to Xtrabackup and the logbin available on the VM.

### Restore Xtrabackup PITR

Principles :

- Choose the mysql node for restore
- Stop other nodes
- Save binlogs in other directory
- Restore backup
- Prepare the restart of MySQL instance
- Start MySQL
- Extract missing transactions from logbins to apply
- Apply missing transactions
- Resynchronize the other mysql nodes


#### Pre-requisites  

On all the nodes

```sh
monit stop mariadb_ctrl
```
On the restore node, copy the binlogs:

```sh
mkdir /tmp/binlog
cp /var/vcap/store/mysql/mysql-bin.* /tmp/binlog
```

#### Restore with SHIELD

lancer la restauration depuis SHIELD 

#### Apply missing transactions

- On restore node, start MySQL Instance in BootStrap

```sh
echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
chown vcap:vcap /var/vcap/store/mysql/state.txt
monit start mariadb_ctrl
```

- Retrieve the last transaction (binlog_pos) applied on the backup

```sh
grep binlog_pos /var/vcap/store/mysql/xtrabackup_info
```
For example

```sh
binlog_pos = filename 'mysql-bin.000022', position '3606', GTID of the last change '0-1-1397001'
```

- Generate transactions file `/tmp/mybinlog.sql` to upgrade MySQL data

. Beginning of transactions : `'mysql-bin.000022'`, position `'3606'`
. Define end of transactions to apply (date / time): for example `2018-01-26 16:20:00`

```sh
/var/vcap/packages/mariadb/bin/mysqlbinlog -uroot /tmp/binlog/mysql-bin.000022 --start-position=3606 --stop-datetime="2018-01-26 16:20:00" > /tmp/mybinlog.sql
```

- Then for each binlog available

```sh
/var/vcap/packages/mariadb/bin/mysqlbinlog -uroot /tmp/binlog/mysql-bin.000023 --stop-datetime="2018-01-26 16:20:00" >> /tmp/mybinlog.sql
```

…etc…
```sh
/var/vcap/packages/mariadb/bin/mysqlbinlog -uroot /tmp/binlog/mysql-bin.000027 --stop-datetime="2018-01-26 16:20:00" >> /tmp/mybinlog.sql
```

- Apply missing transactions 

. connect to mysql

```sh
mysql -uroot –p<mot de passe>
```

 . disable logbin
```sql 
MariaDB [(none)]> set sql_log_bin=off
```
 . Apply transactions

```sql 
MariaDB [(none)]> source /tmp/mybinlog.sql
```
 . enable logbin

```sql
MariaDB [(none)]>  set sql_log_bin=on
```

#### Post Shield restore

- On each other nodes, resynchronize MySQL Galera Cluster

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```
