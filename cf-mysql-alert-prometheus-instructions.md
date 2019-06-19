# Consignes sur les alertes prometheus pour la release bosh cf-mysql-release

## alertes Galera Cluster

### alert: MySQLGaleraClusterSize

. Vérification : Taille du cluster inférieure à 3

```
mysql_global_status_wsrep_cluster_size < 3
```

. Message : "Galera Cluster on <deploiement/instances> < 3 nodes during the last 5m"

. Diagnostic : 
- vérifier les VM
- vérifier monitoring
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### alert:MySQLGaleraClusterEvenNodes

. Vérification : Taille du cluster doit être impaire afin d'éviter un split brain figeant le cluster (pas de quorum atteint)

```
mysql_global_status_wsrep_cluster_size % 2 != 1
```

. Message : "Galera Cluster on <deploiement/instances>  has even of nodes during the last 5m "

. Diagnostic : 
- vérifier les VM
- vérifier monitoring
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### alert:MySQLGaleraNotOperational

. Vérification : Etat du noeud galera différent de Disconnected 
```
mysql_global_status_wsrep_cluster_status != 1
```
. Message : "A Galera Cluster node on <deploiement/instance> had not been operational during the last 5m. It may occur in cases of multiple membership changes that result in a loss of quorum or in cases of split-brain situations"

. Diagnostic : 
- vérifier les VM
- vérifier monitoring
- vérifier les traces MariaDB sous /var/vcap/sys/log/mysql/mysql.err.log

### alert:MySQLGaleraNotReady

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
SST bloqué par la release, il faut identifier la raison puis

```sh
rm -rf /var/vcap/store/mysql
/var/vcap/jobs/mysql/bin/pre-start
monit start mariadb_ctrl
```

### alert:MySQLGaleraNotConnected

. Vérification : Etat du noeud galera différent OFF (Noeud desynchronisé n'accepte pas de de requête)

```
mysql_global_status_wsrep_connected != 1
```
. Message :  "A Galera cluster node on <deploiement/instance> has not been connected to the cluster during the last 5m"

### alert:MySQLGaleraOutOfSync
        expr: (mysql_global_status_wsrep_local_state != 4 AND mysql_global_variables_wsrep_desync == 0)
        for: <%= p('mysql_alerts.out_of_sync.evaluation_time') %>
        labels:
          service: mysql
          severity: warning
        annotations:
          summary: "Galera cluster node on `{{$labels.instance}}` out of sync"
          description: "A Galera cluster node on `{{$labels.instance}}` has not been in sync ({{$value}} != 4) during the last <%= p('mysql_alerts.out_of_sync.evaluation_time') %>"

### alert:MySQLGaleraDonorFallingBehind
        expr: (mysql_global_status_wsrep_local_state == 2 AND mysql_global_status_wsrep_local_recv_queue > <%= p('mysql_alerts.donor_falling_behind.threshold') %>)
        for: <%= p('mysql_alerts.donor_falling_behind.evaluation_time') %>
        labels:
          service: mysql
          severity: warning
        annotations:
          summary: "Galera xtradb cluster donor node on `{{$labels.instance}}` falling behind"
          description: "A Galera cluster node on `{{$labels.instance}}` is a donor (hotbackup) and has been falling behind (queue size {{$value}}) during the last <%= p('mysql_alerts.donor_falling_behind.evaluation_time') %>"

### alert:MySQLGaleraFlowControlPaused
        expr: (mysql_global_status_wsrep_flow_control_paused == 1)
        for: <%= p('mysql_alerts.flow_control_paused.evaluation_time') %>
        labels:
          service: mysql
          severity: critical
        annotations:
          summary: "Galera Cluster node on `{{$labels.instance}}` paused due to Flow Control"
          description: "A Galera Cluster node on `{{$labels.instance}}` has been paused due to flow control during the last <%= p('mysql_alerts.flow_control_paused.evaluation_time') %>"

### alert:MySQLGaleraFlowControlPauseTooHigh
        expr: (mysql_global_status_wsrep_flow_control_paused > <%= p('mysql_alerts.flow_control_pause.min_threshold') %> and mysql_global_status_wsrep_flow_control_paused < <%= p('mysql_alerts.flow_control_pause.max_threshold') %>)
        for: <%= p('mysql_alerts.flow_control_pause.evaluation_time') %>
        labels:
          service: mysql
          severity: warning
        annotations:
          summary: "Galera Cluster node on `{{$labels.instance}}` flow control pause too high"
          description: "A Galera Cluster node on `{{$labels.instance}}` had a flow control pause too high ({{$value}}) during the last <%= p('mysql_alerts.flow_control_pause.evaluation_time') %>"

### alert:MySQLGaleraSendQueueLengthTooHigh
        expr: (mysql_global_status_wsrep_local_send_queue_avg > <%= p('mysql_alerts.send_queue_length.threshold') %>)
        for: <%= p('mysql_alerts.send_queue_length.evaluation_time') %>
        labels:
          service: mysql
          severity: warning
        annotations:
          summary: "Galera Cluster on `{{$labels.instance}}` send queue length too high"
          description: "Galera Cluster on `{{$labels.instance}}` had a local send queue length too high ({{$value}}) during the last <%= p('mysql_alerts.send_queue_length.evaluation_time') %>, It may indicate that replication throttling or network throughput issues"

### alert:MySQLGaleraRecvQueueLengthTooHigh
        expr: (mysql_global_status_wsrep_local_recv_queue_avg > <%= p('mysql_alerts.recv_queue_length.threshold') %>)
        for: <%= p('mysql_alerts.recv_queue_length.evaluation_time') %>
        labels:
          service: mysql
          severity: warning
        annotations:
          summary: "Galera Cluster on `{{$labels.instance}}` recv queue length too high"
          description: "Galera Cluster on `{{$labels.instance}}` had a local received queue length too high ({{$value}}) during the last <%= p('mysql_alerts.recv_queue_length.evaluation_time') %>. It may indicate that the node cannot apply write-sets as fast as it receives them, which can lead to replication throttling"
