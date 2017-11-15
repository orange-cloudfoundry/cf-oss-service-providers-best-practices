# Consignes exploitations Plugin SHIELD 

## Plugin MySQL

### Backup MySQL

	mysql_options: "--flush-logs --add-drop-database --single-transaction  --opt"

### Restauration MySQL

Principe 
- définir le nœud sur lequel on va restaurer
- stopper les autres nœuds

Sur le nœud de restauration, modifier les variables avant   

	set global enforce_storage_engine=NULL;
	set global general_log=OFF;
	set global slow_query_log=OFF;
Sur les autres nœuds  

	monit stop all

#### Restaurer avec SHIELD
Lancer la restauration

#### Post-restauration

Sur le nœud de restauration, modifier les variables  
 
	Set global enforce_storage_engine=InnoDB;
	Set global general_log=OFF;
	Set global slow_query_log=ON;

Sur chacun des nœuds à tour de rôle  

	rm -rf /var/vcap/store/mysql
	/var/vcap/jobs/mysql/bin/pre-start
	monit start all

## Plugin Xtrabackup

### Sauvegarde

RAS

### Restauration

Principe 
- définir le nœud sur lequel on va restaurer
- stopper les autres nœuds

Sur le nœud de restauration :  

	monit stop mariadb_ctrl
	cd /var/vcap/store/mysql/
	rm –Rf *

#### Sur les autres nœuds  

	monit stop all
	
#### restauration SHIELD  

#### Post-restauration 

Sur le nœud de restauration  

	chown -R vcap:vcap /var/vcap/store/mysql/*
	monit start mariadb_ctrl

Sur chacun des nœuds à tour de rôle  

	rm -rf /var/vcap/store/mysql
	/var/vcap/jobs/mysql/bin/pre-start
	monit start all

