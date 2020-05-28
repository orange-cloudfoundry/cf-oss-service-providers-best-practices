# Contexte de la release cf-mysql 37.1
Pour les nouveaux déploiements avec la release cf-mysql 37.1 incluant MariaDB 10.3.22, les tables timezones sont automatiquement pré-chargées. 

Cependant pour les déploiements (SHARED et DEDICATED) cf-mysql 36.9, nécessitant d'être upgradés, les tables timezone doivent être chargées manuellement si nécessaire. 

Ce document décrit le mode opératoire pour les alimenter 


# Mode opératoire

Principe  
- sur tous les noeuds, désactiver le paramètre `enforce_storage_engine` 
- sur 1 seul noeud, charger les tables timezone
- sur tous les noeuds, réactiver le paramètre  `enforce_storage_engine`

## Désactiver `enforce_storage_engine`  

Sur tous les noeuds  modifier la variable systeme MySQL  

. Connection à MySQL

```sh
mysql --defaults-file=/var/vcap/jobs/mysql/config/mylogin.cnf
```
. Désactiver le paramètre

```sql
MariaDB [(none)]> set global enforce_storage_engine=NULL;
```

. Vérifier 

```sql
MariaDB [(none)]> show global variables like 'enforce%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| enforce_storage_engine |       |
+------------------------+-------+
```

## Charger les tables timezone 

Sur un seul noeud  

. Chargement

```sh
/var/vcap/packages/mariadb/bin/mysql_tzinfo_to_sql /usr/share/zoneinfo  | /var/vcap/packages/mariadb/bin/mysql --defaults-file="/var/vcap/jobs/mysql/config/mylogin.cnf" mysql


Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as time zone. Skipping it.
```

. Vérifier 

```sql
MariaDB [(none)]> select count(*) from mysql.time_zone;
+----------+
| count(*) |
+----------+
|     1820 |   
+----------+
```
. Vérifier heure UTC

```sql
MariaDB [mysql]> select now();
+---------------------+
| now()               |
+---------------------+
| 2020-05-28 08:20:19 |
+---------------------+
```
. Vérifier heure Paris

```sql
MariaDB [mysql]> SET time_zone = 'Europe/Paris';
Query OK, 0 rows affected (0.021 sec)

MariaDB [mysql]> select now();
+---------------------+
| now()               |
+---------------------+
| 2020-05-28 10:20:42 |
+---------------------+
```

## Réactiver `enforce_storage_engine`

Sur tous les noeuds  modifier la variable systeme MySQL  

. Connection à MySQL

```sh
mysql --defaults-file=/var/vcap/jobs/mysql/config/mylogin.cnf
```
. Réactiver le paramètre

```sql
MariaDB [(none)]> set global enforce_storage_engine=innodb;
```

. Vérifier 

```sql
MariaDB [(none)]> show global variables like 'enforce%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| enforce_storage_engine |InnoDB |
+------------------------+-------+
```
