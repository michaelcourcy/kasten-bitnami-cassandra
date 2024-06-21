# Backing up Cassandra bitnami with Kasten 

This blueprint allow the backup of a cassandra database deployed with the bitnami helm chart.

The specificity of this blueprint is that it only work with Kasten because it store the backup on the same PVC than the data. Each PVC contains its corresponding PVC.

- backupPrehook and backupPosthook action prepare the backup on the PVCs of the statefulset
- kasten backup the PVCs
- the restore action erase the data if they exist and restore from the backup folder


## install cassandra

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install cassandra bitnami/cassandra --namespace=cassandra --version=11.3.1 --create-namespace --set replicaCount=
```

## Create data in cassandra 

Let's connect to the first node of the replica set
```
k exec -it -n cassandra cassandra-0 -- bash 
```

then create data 
```
cqlsh -u cassandra -p $CASSANDRA_PASSWORD cassandra
create keyspace restaurants with replication  = {'class':'SimpleStrategy', 'replication_factor': 3};
# once the keyspace is created let's create a table named guests and some data into that table
create table restaurants.guests (id UUID primary key, firstname text, lastname text, birthday timestamp);
insert into restaurants.guests (id, firstname, lastname, birthday)  values (5b6962dd-3f90-4c93-8f61-eabfa4a803e2, 'Vivek', 'Singh', '2015-02-18');
insert into restaurants.guests (id, firstname, lastname, birthday)  values (5b6962dd-3f90-4c93-8f61-eabfa4a803e3, 'Tom', 'Singh', '2015-02-18');
insert into restaurants.guests (id, firstname, lastname, birthday)  values (5b6962dd-3f90-4c93-8f61-eabfa4a803e4, 'Prasad', 'Hemsworth', '2015-02-18');
# once you have the data inserted you can list all the data inside a table using the command
select * from restaurants.guests;
```

## Create the blueprint and the blueprintbinding for both mongo and cassandra


Apply both the blueprint and the blueprint binding 
```
kubectl apply -f cassandra-blueprint.yaml -n kasten-io
kubectl apply -f cassandra-blueprint-binding.yaml -n kasten-io 
```

Now go on kasten GUI and create a policy with export do not forget to define a kanister profile.


## Test restore 

Delete the namespace and restore from the remote restorepoint.

Check also the data for cassandra 
```
k exec -it -n cassandra cassandra-0 -- bash 
cqlsh -u cassandra -p $CASSANDRA_PASSWORD cassandra-release
select * from restaurants.guests;
```

## How it works 

To backup and restore a Cassandra database, you need to backup and restore the snapshot on each node in the cluster. This is because Cassandra is a distributed database, and data is partitioned across different nodes in the cluster. Each node is responsible for a specific set of data, determined by the partition key and the cluster's replication strategy.

To backup you need to capture 
- The keyspaces schema, something that you obtain with `DESCRIBE keyspaces`, you can do that in any cassandra nodes
- The snapshot on each nodes that you obtain with `nodetool snapshot -t ${HOSTNAME}`, you have to do that on every nodes of the statefulset

To restore you need to restore  
- The keyspaces schema, something that you do with `cqlsh -e "$(cat ${snapshot_prefix}/schema.cql)"`, you can do that in any cassandra nodes
- The snaphot within each nodes by looping on all the ssTable `sstableloader ${HOSTNAME} $table`, you have to do that on every nodes of the statefulset


The specificity of this blueprint is that it only work with Kasten because it store the backup on the same PVC than the data. Each PVC contains its corresponding PVC.

- backupPrehook and backupPosthook action prepare the backup on the PVCs of the statefulset
- kasten backup the PVCs
- the restore action erase the data if they exist and restore from the backup folder

Check the blueprint and adapt it to your need.