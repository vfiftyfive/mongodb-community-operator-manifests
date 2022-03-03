# Deploy MongoDB with Ondat

The YAML manifests in this repo help you to perform the following tasks:
- Deploy MongoDB with the Community Operator. It creates a 3-node MongoDB cluster as a Kubernetes `StatefulSet` with 3 replicas.
- Encrypt the data volume of the MongoDB nodes with private keys stored as `Secrets`.

## Create an Ondat StorageClass with Encryption enabled
Through the CSI interface, Ondat provides data services such as encryption, replication and performance optimization. You can enable encryption by adding a new parameter to the `StorageClass` definition. Let's create a new `StorageClass` with encryption enabled:
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  labels:
    app: storageos
  name: ondat-encrypted
parameters:
  storageos.com/encryption: "true"
  storageos.com/replicas: 1
  csi.storage.k8s.io/fstype: xfs
  csi.storage.k8s.io/secret-name: storageos-api
  csi.storage.k8s.io/secret-namespace: storageos
provisioner: csi.storageos.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
EOF
```
## Deploy MongoDB Cluster
First, deploy the MongoDB Custom Resource Definition:
```
kubectl apply -f mongo-crd.yaml
```

Deploy the clusterwide RBAC resources:
```
kubectl apply -f clusterwide/
```

Create a namespace for the operator:
```
kubectl create ns mongo-operator
```

Deploy namespace RBAC resources in the operator namespace:
```
kubectl apply -k rbac/ -n mongo-operator
```

Deploy namespace RBAC resources in the default namespace (where the database will be deployed):
```
kubectl apply -k rbac/
```

Deploy the Operator:
```
kubectl apply -f manager.yaml -n mongo-operator
```

Check the Operator has correctly been deployed and wait for it to be running:
```
watch kubectl get pods -n mongo-operator
```
Result:
```
Every 2.0s: kubectl get pods -n mongo-operator

NAME                                           READY   STATUS    RESTARTS   AGE
mongodb-kubernetes-operator-6d46dd4b74-xxxwh   1/1     Running   0          92s
```

Press Ctrl+C to cancel the command.

Finally, deploy the MongoDB cluster:
```
kubectl apply -f mongodb-config.yaml
```

Check that the cluster is deploying:
```
watch kubectl get pods
```
Result:
```
Every 2.0s: kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
mongodb-0   2/2     Running   0          3m29s
mongodb-1   2/2     Running   0          2m25s
mongodb-2   1/2     Running   0          61s
```

## Create a New Database
Now that you have a running MongoDB cluster, let's create a database and a collection. A collection in MongoDB NoSQL architecture is comparable to a table in a relational database.

You are going to create a JSON document in the `shakespeare` database, within the `quotes` collection. This document has the following format:
```json
{
  "publication": "Romeo and Juliet",
  "text": "What\'s in a name? That which we call a rose by any other word would smell as sweet..."
}
```

When deploying the `StatefulSet`, the Operator creates a MongoDB Replica Set, where one `Pod` holds the primary data set and the other ones hold replicas. Let's first verify this by running the following command:

```
kubectl exec -it mongodb-0 -- mongosh --username admin --password mongo --quiet --eval 'printjson(rs.status())'
```

Identify the primary MongoDB node. You should see a JSON Key/Value pair like below:
```
stateStr: 'PRIMARY'
```

The following MongoDB commands must be run from the primary node. Replace the `Pod` ordinal number accordingly so you can create the database and populate the collection.

```
kubectl exec -it mongodb-0 -- mongosh admin < /root/ondat/mongo.sh
```

Result:
```
...
mongodb [direct: primary] admin> { ok: 1 }
mongodb [direct: primary] admin> switched to db shakespeare
mongodb [direct: primary] shakespeare> { ok: 1 }
mongodb [direct: primary] shakespeare> {
  acknowledged: true,
  insertedId: ObjectId("6209a09540c27d9ba2907b76")
}
```

Let's query the database to check that the document has been correctly created.
```
kubectl exec -it mongodb-0 -- mongosh admin < /root/ondat/checkmongo.sh
```

Result:

```
...
mongodb [direct: primary] admin> { ok: 1 }
mongodb [direct: primary] admin> switched to db shakespeare
mongodb [direct: primary] shakespeare> [
  {
    _id: ObjectId("6209a09540c27d9ba2907b76"),
    publication: 'Romeo and Juliet',
    text: "What's in a name? That which we call a rose by any other word would smell as sweet..."
  }
]
```

All good! You have deployed a new database that contains a document. Now, Let's verify that the database is encrypted!

## Verify Volume Encryption
First, let's find the Ondat volumes associated with the MongoDB `Pods` by executing the following commands:
```
#Enable Ondat CLI first
 kubectl port-forward svc/storageos -n storageos 5705 &
 storageos get volumes
 ```

Result:
```
NAMESPACE  NAME                                      SIZE     LOCATION          ATTACHED ON  REPLICAS  AGE
default    pvc-723f1945-5c99-441d-a8a3-f43cc6d0c668  1.0 GiB  worker1 (online)  worker1      0/0       6 minutes ago
default    pvc-afbb7b28-a8f3-4238-966f-76b2a3ebcf9c  1.0 GiB  worker2 (online)  worker2      0/0       7 minutes ago
default    pvc-f70ac39b-b74f-4b2e-b47c-9b49e14ae25b  1.0 GiB  worker2 (online)  worker2      0/0       7 minutes ago
default    pvc-85179a6f-44d2-49c6-8b11-d96bd91009b2  1.0 GiB  worker3 (online)  worker3      0/0       8 minutes ago
default    pvc-3ee3c186-a7a7-4035-893f-e66607e73c2e  1.0 GiB  worker1 (online)  worker1      0/0       6 minutes ago
default    pvc-fabbdce2-9b42-4e4f-a719-ee18e56c1195  1.0 GiB  worker3 (online)  worker3      0/0       8 minutes ago
```

Get the list of `PVCs` to map them out to the Ondat volumes displayed above:
```
kubectl get pvc
```
Result:
```
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
data-volume-mongodb-0   Bound    pvc-85179a6f-44d2-49c6-8b11-d96bd91009b2   1Gi        RWO            ondat-encrypted   11m
data-volume-mongodb-1   Bound    pvc-f70ac39b-b74f-4b2e-b47c-9b49e14ae25b   1Gi        RWO            ondat-encrypted   10m
data-volume-mongodb-2   Bound    pvc-3ee3c186-a7a7-4035-893f-e66607e73c2e   1Gi        RWO            ondat-encrypted   9m46s
logs-volume-mongodb-0   Bound    pvc-fabbdce2-9b42-4e4f-a719-ee18e56c1195   1Gi        RWO            ondat-encrypted   11m
logs-volume-mongodb-1   Bound    pvc-afbb7b28-a8f3-4238-966f-76b2a3ebcf9c   1Gi        RWO            ondat-encrypted   10m
logs-volume-mongodb-2   Bound    pvc-723f1945-5c99-441d-a8a3-f43cc6d0c668   1Gi        RWO            ondat-encrypted   9m46s
```

In the example above, you can see that `mongodb-0` data volume PVC ID is ending with `9b2`. If you check this ID in the Ondat volumes list, you can also find this volume ending with `9b2`. This information allows you to identify the node where the data volume is attached to. In this example, the node is `worker3` (use the column `ATTACHED ON`).

Let's check that there's no trace of the string `rose` in the encrypted data volume of `mongodb-0`. You can now choose any other MongoDB `Pod` if you fancy! The data is replicated to the other two nodes. Just make sure you target the right Kubernetes worker node (ie. where the Ondat volume is attached to).

But first, verify that the string rose is present within the database binary file located under /data by executing the following command:
```
kubectl exec -it mongodb-0 -- grep -r rose /data
```
Result:
```
Binary file /data/journal/WiredTigerLog.0000000002 matches
Binary file /data/collection-16-6361445414969153344.wt matches
Binary file /data/collection-6--5148538780104397675.wt matches
Binary file /data/collection-10--5148538780104397675.wt matches
```

You can notice that there is a match with multiple files.

Now, let's verify if the data is encrypted when accessing the raw volume. This is simulating an attacker who leveraged privilege escalation to get access to the storage device. The string `rose` should not be visible when the attacker tries to access the encrypted data. Let's verify this.

Connect to the node where the Ondat volume is attached to (column `ATTACHED ON` above). In our case, this is node `worker3`. Run the following command:
```
ssh root@worker3 strings /var/lib/storageos/data/dev1/vol.*.blob | grep rose
```
The result should be empty. This validates that the volume is encrypted!
