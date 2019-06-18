# Consignes exploitations Plugin SHIELD V6

## Plugin MySQL

Définir les options de MySQLDump dans le manifest

```
mysql_host: 127.0.0.1
mysql_port: "3306"
mysql_user: root
mysql_password: ((cf_mysql_mysql_admin_password))
mysql_options: "--flush-logs --add-drop-database --single-transaction  --opt"
mysql_bindir: "/var/vcap/packages/mariadb/bin"
```

### Restauration avec MySQLDump

Principe 
- définir le nœud sur lequel on va restaurer
- stopper les autres nœuds
- restaurer 
- resynchroniser les autres nœuds

#### Pre-requis  

Sur le nœud de restauration, modifier les variables systeme MySQL  

. connection à MySQL

```sh
mysql -uroot –p<mot de passe>
```
. desactiver les paramètres 

```sql
MariaDB [(none)]> set global enforce_storage_engine=NULL;
MariaDB [(none)]> set global general_log=OFF;
MariaDB [(none)]> set global slow_query_log=OFF;
```

. Vérifier l'espace disque utilisé par les logbin MySQL

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

et purger and précisant le nom du dernier binlog : Attention, pas de récupération possible.

```sql
MariaDB [(none)]> PURGE BINARY LOGS BEFORE now();
Query OK, 0 rows affected (0.02 sec)
```

Sur les autres nœuds  

```sh
monit stop mariadb_ctrl
```

#### Restaurer avec SHIELD
Lancer la restauration

#### Post-restauration

Sur le nœud de restauration, Remettre les variables systeme MySQL à leurs valeurs initiales  

. connection à MySQL

```sh
mysql -uroot –p<mot de passe>
```
. réactiver les paramètres 

```sql
MariaDB [(none)]> Set global enforce_storage_engine=InnoDB;
MariaDB [(none)]> Set global general_log=OFF;
MariaDB [(none)]> Set global slow_query_log=ON;
```

Puis chacun des nœuds à tour de rôle, resynchroniser   

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

## Plugin Xtrabackup
Définir les options de Xtrabackup dans le manifest

```sh
mysql_user: root
mysql_password: ((cf_mysql_mysql_admin_password))
mysql_datadir: "/var/vcap/store/mysql"
mysql_xtrabackup: "/var/vcap/packages/xtrabackup/bin/xtrabackup"
```

if you are using a higher version of cf-mysql-release than 36.11 and Shield 7.x, you must set the socket as follows 
```sh
mysql_xtrabackup: ""/var/vcap/packages/xtrabackup/bin/xtrabackup --socket=/var/vcap/sys/run/mysql/mysqld.sock"
```

### Restauration avec Xtrabackup

Principe 
- Définir le nœud sur lequel on va restaurer
- Stopper les autres nœuds
- Restaurer
- Preparer le redémarrage de l'instance
- Resynchroniser les nœuds

#### Pre-requis 
Sur le nœud de restauration :  

```sh
monit stop mariadb_ctrl
cd /var/vcap/store/mysql/
rm –Rf *
```

Sur les autres nœuds  

```sh
monit stop mariadb_ctrl
```
	
#### restauration SHIELD  

lancer la restauration depuis SHIELD 

#### Post-restauration 

Sur le nœud de restauration  

```sh
echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
chown -R vcap:vcap /var/vcap/store/mysql/*
monit start mariadb_ctrl
```

Sur chacun des nœuds à tour de rôle  

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

