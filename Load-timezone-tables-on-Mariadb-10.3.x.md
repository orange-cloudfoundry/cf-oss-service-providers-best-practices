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

Sur tous les noeuds  modifier les variables systeme MySQL  

. Connection à MySQL

```sh
mysql --defaults-file=/var/vcap/jobs/mysql/config/mylogin.cnf
```
. Désactiver les paramètres 

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

## Réactiver `enforce_storage_engine`

Sur tous les noeuds  modifier les variables systeme MySQL  

. Connection à MySQL

```sh
mysql --defaults-file=/var/vcap/jobs/mysql/config/mylogin.cnf
```
. Réactiver les paramètres 

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
