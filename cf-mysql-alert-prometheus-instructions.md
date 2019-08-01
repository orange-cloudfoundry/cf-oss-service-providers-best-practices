# Consignes sur les alertes prometheus pour la release bosh cf-mysql-release

## Alertes Galera Cluster

### MySQLGaleraClusterSize

. Vérification : Taille du cluster inférieure à 3

```
mysql_global_status_wsrep_cluster_size < 3
```

. Message : "Galera Cluster on <deploiement/instances> < 3 nodes during the last 5m"

. Diagnostic : 
- vérifier les VM
- vérifier monitoring
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### MySQLGaleraClusterEvenNodes

. Vérification : Taille du cluster doit être impaire afin d'éviter un split brain figeant le cluster (pas de quorum atteint)

```
mysql_global_status_wsrep_cluster_size % 2 != 1
```

. Message : "Galera Cluster on <deploiement/instances>  has even of nodes during the last 5m "

. Diagnostic : 
- vérifier les VM
- vérifier monitoring
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### MySQLGaleraNotOperational

. Vérification : Etat du noeud galera différent de NON-PRIMARY 
```
mysql_global_status_wsrep_cluster_status != 1
```
. Message : "A Galera Cluster node on <deploiement/instance> had not been operational during the last 5m. It may occur in cases of multiple membership changes that result in a loss of quorum or in cases of split-brain situations"

. Diagnostic : 
- vérifier les VM
- vérifier monitoring
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### MySQLGaleraNotReady

. Vérification : Etat du noeud galera différent OFF (Noeud desynchronisé n'accepte pas de de requête)

```
mysql_global_status_wsrep_ready != 1
```
. Message : "A Galera cluster node on <deploiement/instance> has not been ready during the last 5m"

. Diagnostic : 
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

Si message 
```
#####################################################################################
SST disabled due to danger of data loss. Verify data and run the rejoin-unsafe errand
#####################################################################################"
```
SST bloqué par la release, il faut identifier la raison puis relancer la synchrinisation manuellement

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

### MySQLGaleraNotConnected

. Vérification : Noeud galera non connecté au cluster

```
mysql_global_status_wsrep_connected != 1
```
. Message :  "A Galera cluster node on <deploiement/instance> has not been connected to the cluster during the last 5m"

. Diagnostic : 
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### MySQLGaleraOutOfSync

. Vérification : Noeud non Synced et n'etant pas utilisé pour un SST

```
mysql_global_status_wsrep_local_state != 4 AND mysql_global_variables_wsrep_desync == 0)
```
. Message :  "A Galera cluster node on <deploiement/instance> has not been in sync ) during the last 5m"

. Diagnostic : 
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### MySQLGaleraDonorFallingBehind

. Vérification : Noeud Donor/Desynced avec temps de réponse > 100ms

```
mysql_global_status_wsrep_local_state == 2 AND mysql_global_status_wsrep_local_recv_queue > 100
```

. Message : "A Galera cluster node on <deploiement/instance> is a donor (hotbackup) and has been falling behind (queue size 100) during the last 5m"

. Diagnostic : 
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log


### MySQLGaleraFlowControlPaused

. Vérification : Cluster Galera figé, réplication ne commite pas

```
mysql_global_status_wsrep_flow_control_paused == 1)
```
. Message : "A Galera Cluster node on <deploiement/instance> has been paused due to flow control during the last 5m"

. Diagnostic : 
- identifier le noeud qui bloque via les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### MySQLGaleraFlowControlPauseTooHigh

. Vérification : Cluster Galera ralenti, lag de réplication entre 0.5 et 1

```
mysql_global_status_wsrep_flow_control_paused > 0.5 and mysql_global_status_wsrep_flow_control_paused < 1)
```
. Message : "A Galera Cluster node on <deploiement/instance> had a flow control pause too high during the last 5m"

. Diagnostic : 
- identifier le noeud qui bloque via les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### MySQLGaleraSendQueueLengthTooHigh

. Vérification : replication (send) > 0.01

```
mysql_global_status_wsrep_local_send_queue_avg > 0.01
```
. Message : "Galera Cluster on <deploiement/instance> had a local send queue length too high ({{$value}}) during the last 5m, It may indicate that replication throttling or network throughput issues"

. Diagnostic : 
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log
- vérifier les status
```sh
mysql -uroot –p<mot de passe>
```
si
```sql
MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_local_%_queue';

Variable_name           Value   
----------------------  --------
wsrep_local_recv_queue  0       
wsrep_local_send_queue  0     
```
alors 
```sql
MariaDB [(none)]> flush status;
```

### MySQLGaleraRecvQueueLengthTooHigh
. Vérification : replication (recv) > 0.5

```
mysql_global_status_wsrep_local_recv_queue_avg > 0.5
```
. Message : "Galera Cluster on <deploiement/instance> had a local received queue length too high ({{$value}}) during the last 5m, It may indicate that the node cannot apply write-sets as fast as it receives them, which can lead to replication throttling""

. Diagnostic : 
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log
- vérifier les status
```sh
mysql -uroot –p<mot de passe>
```
si
```sql
MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_local_%_queue';

Variable_name           Value   
----------------------  --------
wsrep_local_recv_queue  0       
wsrep_local_send_queue  0     
```
alors 
```sql
MariaDB [(none)]> flush status;
```

## Alerte Performances

### MySQLInnoDBLogWaits

. Vérification : Moyenne des temps d'écriture dans les InnoDBlog trop important

```
rate(mysql_global_status_innodb_log_waits[15m]) > 10
```

. Message : "the innodb logs at <deploiement/instances> are waiting for disk at a rate of <value>/second"

. Diagnostic : 
- vérifier les taux de remplissage du fs
- vérifier les performances des disques persistents

## Alerte prometheus-mysqld-exporter

### MySQLdExporterScrapeError

. Vérification : 

```
(mysql_exporter_last_scrape_error) != 0
```

. Message : "The mysqld_exporter <deploiement/instances> was unable to scrape metrics during the last 10m

. Diagnostic : 
- vérifier les status
```sh
monit summary

...
Process 'mysqld_exporter'           running
...

```
Si ko, 
```sh
monit start mysqld_exporter
```

Si ok et traces ok dans grafana

```sh
monit restart mysqld_exporter
...
