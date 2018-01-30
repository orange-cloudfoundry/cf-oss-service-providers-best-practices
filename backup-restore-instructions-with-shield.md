# Consignes exploitations Plugin SHIELD 

## Plugin MySQL

Définir les options de MySQLDump dans le manifest

	mysql_host: 127.0.0.1
	mysql_port: "3306"
	mysql_user: root
	mysql_password: ((cf_mysql_mysql_admin_password))
	mysql_options: "--flush-logs --add-drop-database --single-transaction  --opt"
	mysql_bindir: "/var/vcap/packages/mariadb/bin"

### Restauration MySQL

Principe 
- définir le nœud sur lequel on va restaurer
- stopper les autres nœuds
- restaurer 
- resynchroniser les autres nœud

#### Pre-requis  

Sur le nœud de restauration, modifier les variables avant   

	set global enforce_storage_engine=NULL;
	set global general_log=OFF;
	set global slow_query_log=OFF;

Vérifier la place prise par les logbin

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
et purger and précisant le nom du dernier binlog : Attention, pas de récupération possible.

	MariaDB [(none)]> PURGE BINARY LOGS BEFORE now();
	Query OK, 0 rows affected (0.02 sec)
	

Sur les autres nœuds  

	monit stop mariadb_ctrl

#### Restaurer avec SHIELD
Lancer la restauration

#### Post-restauration

Sur le nœud de restauration, Remettre les variables à leur valeurs initiales  
 
	Set global enforce_storage_engine=InnoDB;
	Set global general_log=OFF;
	Set global slow_query_log=ON;

Puis chacun des nœuds à tour de rôle, resynchroniser   

	rm -rf /var/vcap/store/mysql
	/var/vcap/jobs/mysql/bin/pre-start
	monit start mariadb_ctrl

## Plugin Xtrabackup

### Restauration Xtrabackup

Principe 
- définir le nœud sur lequel on va restaurer
- stopper les autres nœuds
- restaurer
- preparer le redémarrage de l'instance
- resynchroniser les nœuds

#### Pre-requis 
Sur le nœud de restauration :  

	monit stop mariadb_ctrl
	cd /var/vcap/store/mysql/
	rm –Rf *

Sur les autres nœuds  

	monit stop mariadb_ctrl
	
#### restauration SHIELD  

lancer la restauration depuis SHIELD 

#### Post-restauration 

Sur le nœud de restauration  

	echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
	chown -R vcap:vcap /var/vcap/store/mysql/*
	monit start mariadb_ctrl

Sur chacun des nœuds à tour de rôle  

	rm -rf /var/vcap/store/mysql
	/var/vcap/jobs/mysql/bin/pre-start
	monit start mariadb_ctrl

