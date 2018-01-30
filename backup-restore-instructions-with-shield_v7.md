# Consignes exploitations SHIELD V7.x avec plugin MySQL et Xtrabackup 

Les 2 plugins ont été améliorés afin de minimiser les opérations d'exploitation durant la restauration.

## Plugin MySQL

Définir les options de MySQLDump dans le manifest

	mysql_host: 127.0.0.1
	mysql_port: "3306"
	mysql_user: root
	mysql_password: ((cf_mysql_mysql_admin_password))
	mysql_options: "--flush-logs --add-drop-database --single-transaction  --opt"
	mysql_bindir: "/var/vcap/packages/mariadb/bin"

## Plugin MySQL: Restauration

Principe 
– Définir le nœud sur lequel on va restaurer
– Stopper les autres nœuds
– Restaurer
– Resynchroniser les autres nœuds

#### Prerequis
**Sur le nœud de restauration **
– Aucune action à faire, le plugin réalise les modifications du paramétrage nécessaires

**Sur les autres nœuds  **

	monit stop mariadb_ctrl

#### Restaurer avec SHIELD

Lancer la restauration depuis shield

#### Post-restauration

Si les vérifications sont correctes
- Sur chacun autres nœuds à tour de rôle  

	rm -rf /var/vcap/store/mysql
	/var/vcap/jobs/mysql/bin/pre-start
	monit start mariadb_ctrl

## Plugin Xtrabackup

### Restauration

Principe 
– Définir le nœud sur lequel on va restaurer
– Stopper les autres nœuds
– Restaurer
– Préparer le redémarrage en BOOTSTRAP
– Démarrer
– Resynchroniser les autres nœuds

#### Prerequis
Sur tous les nœuds :  

	monit stop mariadb_ctrl

#### Restaurer avec SHIELD  

lancer la restauration depuis SHIELD 

#### Post-restauration 

. Sur le nœud de restauration, Redémarrer l'instance en boot-strap  

	echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
	chown vcap:vcap /var/vcap/store/mysql/state.txt
	monit start mariadb_ctrl

. Sur chacun des nœuds à tour de rôle  

	rm -rf /var/vcap/store/mysql
	/var/vcap/jobs/mysql/bin/pre-start
	monit start mariadb_ctrl

## Restauration avec PITR (Point In Time Recovery)
Dans certains cas, il peut être intéressant de restaurer via Xtrabackup et de pouvoir appliquer les logbin produits post sauvegarde jusqu'à une date précise.

### Restauration

Principe 
– définir le nœud sur lequel on va restaurer (ne pas restaurer une sauvegarde d'un nœud sur un autre nœud)
– stopper les nœuds
– copier les binlogs dans un autre répertoire
– Restaurer,
– Redémarrer l'instance en BOOTSTRAP
– Extraire les transactions manquantes des logbins à appliquer
– Appliquer les transactions manquantes 
– Resynchroniser les autres nœuds


#### Prerequis
Sur tous les nœuds :  

	monit stop mariadb_ctrl

Sur le noeud de restauration, copier les binlog :  

	mkdir /tmp/binlog
	cp /var/vcap/store/mysql/mysql-bin.* /tmp/binlog


#### Restaurer avec SHIELD  

lancer la restauration depuis SHIELD 

#### Post-restauration 

. Sur le nœud de restauration, Redémarrer l'instance en boot-strap  

	echo -n "NEEDS_BOOTSTRAP" > /var/vcap/store/mysql/state.txt
	chown vcap:vcap /var/vcap/store/mysql/state.txt
	monit start mariadb_ctrl

. Sur chacun des nœuds à tour de rôle  

	rm -rf /var/vcap/store/mysql
	/var/vcap/jobs/mysql/bin/pre-start
	monit start mariadb_ctrl
