# Consignes exploitations SHIELD V7.x avec plugin MySQL et Xtrabackup 

Les 2 plugins ont été améliorés afin de minimiser les opérations d'exploitation durant la restauration.

## Plugin MySQL

Définir les options de MySQLDump dans le manifest

```sh
mysql_host: 127.0.0.1
mysql_port: "3306"
mysql_user: root
mysql_password: ((cf_mysql_mysql_admin_password))
mysql_options: "--flush-logs --add-drop-database --single-transaction  --opt"
mysql_bindir: "/var/vcap/packages/mariadb/bin"
```

## Plugin MySQL: Restauration

Principe

- Définir le nœud sur lequel on va restaurer
- Stopper les autres nœuds
- Restaurer
- Resynchroniser les autres nœuds

#### Prerequis  

Sur le nœud de restauration  

Aucune action à faire, le plugin réalise les modifications nécessaires des variables systeme MySQL 

Sur les autres nœuds

```sh
monit stop mariadb_ctrl
```

#### Restaurer avec SHIELD

Lancer la restauration depuis shield

#### Post-restauration

Si les vérifications sont correctes
- Sur chacun autres nœuds à tour de rôle  

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

## Plugin Xtrabackup

### Restauration

Principe 

- Définir le nœud sur lequel on va restaurer
- Stopper les autres nœuds
- Restaurer
- Préparer le redémarrage en BOOTSTRAP
- Démarrer
- Resynchroniser les autres nœuds

#### Prerequis
Sur tous les nœuds :  

```sh
monit stop mariadb_ctrl
```

#### Restaurer avec SHIELD  

lancer la restauration depuis SHIELD 

#### Post-restauration 

. Sur le nœud de restauration, Redémarrer l'instance en boot-strap  

```sh
echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
chown vcap:vcap /var/vcap/store/mysql/state.txt
monit start mariadb_ctrl
```

Si les vérifications sont correctes :  
. Sur chacun des nœuds à tour de rôle  

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

## Restauration avec PITR (Point In Time Recovery)
Dans certains cas, il peut être intéressant de restaurer jusqu'à une date précise, ceci grâce à Xtrabackup et aux logbin disponibles sur la VM.

### Restauration Xtrabackup PITR

Principe 

- Définir le nœud sur lequel on va restaurer (ne pas restaurer une sauvegarde d'un nœud sur un autre nœud)
- Stopper les nœuds
- Copier les binlogs dans un autre répertoire
- Restaurer,
- Redémarrer l'instance en BOOTSTRAP
- Extraire les transactions manquantes des logbins à appliquer
- Appliquer les transactions manquantes 
- Resynchroniser les autres nœuds


#### Prerequis
Sur tous les nœuds :  

```sh
monit stop mariadb_ctrl
```

Sur le noeud de restauration, copier les binlog :  

```sh
mkdir /tmp/binlog
cp /var/vcap/store/mysql/mysql-bin.* /tmp/binlog
```

#### Restaurer avec SHIELD  

lancer la restauration depuis SHIELD 

#### Appliquer les transactions manquantes des logbins

. Sur le nœud de restauration, Redémarrer l'instance en boot-strap  

```sh
echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
chown vcap:vcap /var/vcap/store/mysql/state.txt
monit start mariadb_ctrl
```

. Récupérer la dernière transaction (binlog_pos) appliquée sur la sauvegarde

```sh
grep binlog_pos /var/vcap/store/mysql/xtrabackup_info
```
par exemple

```sh
binlog_pos = filename 'mysql-bin.000022', position '3606', GTID of the last change '0-1-1397001'
```

. Génération du fichier `/tmp/mybinlog.sql` pour remettre à niveau l'instance mysql

– Début des transactions : `'mysql-bin.000022'`, position `'3606'`
– Définir fin des transactions à appliquer (date/heure) : par exemple `2018-01-26 16:20:00`

```sh
/var/vcap/packages/mariadb/bin/mysqlbinlog -uroot /tmp/binlog/mysql-bin.000022 --start-position=3606 --stop-datetime="2018-01-26 16:20:00" > /tmp/mybinlog.sql
```

- Puis pour chaque binlog disponibles

```sh
/var/vcap/packages/mariadb/bin/mysqlbinlog -uroot /tmp/binlog/mysql-bin.000023 --stop-datetime="2018-01-26 16:20:00" >> /tmp/mybinlog.sql
```

…etc…
```sh
/var/vcap/packages/mariadb/bin/mysqlbinlog -uroot /tmp/binlog/mysql-bin.000027 --stop-datetime="2018-01-26 16:20:00" >> /tmp/mybinlog.sql
```

. Appliquer les transactions manquantes 

connection à la base

```sh
mysql -uroot –p<mot de passe>
```

désactiver les logbin
```sql 
MariaDB [(none)]> set sql_log_bin=off
```
appliquer les transactions

```sql 
MariaDB [(none)]> source /tmp/mybinlog.sql
```
Réactiver les logbin

```sql
MariaDB [(none)]>  set sql_log_bin=on
```

#### Post-restauration 

. Sur chacun des nœuds à tour de rôle  

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```
