# cert-manager

In this example we'll use the Vault PKI as issuer.
Here I deployed a vault operator that uses a etcd backend.

```
kubectl get po -n cert-manager
NAME                                                              READY   STATUS    RESTARTS   AGE
cert-manager-86c45c86c8-9m4kd                                     1/1     Running   13         103d
cert-manager-cainjector-6885996d5-zgfcm                           1/1     Running   54         103d
cert-manager-webhook-59dfddccfd-rdrdn                             1/1     Running   0          103d
vault-operator-5b5c6b64dc-s2wl5                                   1/1     Running   5          103d
vault-operator-etcd-operator-etcd-backup-operator-69b95848dx8mw   1/1     Running   5          103d
vault-operator-etcd-operator-etcd-operator-765667ff65-kdshj       1/1     Running   5          103d
vault-operator-etcd-operator-etcd-restore-operator-8c6bdc9t2ssw   1/1     Running   28         103d
vault-pki-5669bfcb94-7vzn8                                        2/2     Running   0          103d
vault-pki-5669bfcb94-95mh5                                        1/2     Running   0          14d
vault-pki-etcd-6pzz5v5lzx                                         1/1     Running   0          103d
vault-pki-etcd-96t9hpnfv9                                         1/1     Running   0          103d
vault-pki-etcd-wtkkzn2d5l                                         1/1     Running   0          103d
```

Create an issuer in the namespace demo
```
$ kubectl create -f operators/cert-manager/vault-issuer.yaml -n demo
issuer.certmanager.k8s.io/vault-issuer created

$ kubectl describe issuer -n demo vault-issuer
```

Create a certificate
```
$ kubectl create -f operators/cert-manager/certificates/smana-test.yaml -n demo

$ kubectl describe cert -n demo smana-test
```



# percona xtradb cluster operator


**Note**: Official documentation : https://www.percona.com/doc/kubernetes-operator-for-pxc/index.html

### Install the operator
The following command will install a set of kubernetes objects that are required for by the operator

```
$ kubectl apply -f operators/percona-xtradb-cluster-operator/operator-bundle.yaml -n pxc
customresourcedefinition.apiextensions.k8s.io/perconaxtradbclusters.pxc.percona.com unchanged
customresourcedefinition.apiextensions.k8s.io/perconaxtradbclusterbackups.pxc.percona.com unchanged
customresourcedefinition.apiextensions.k8s.io/perconaxtradbclusterrestores.pxc.percona.com unchanged
customresourcedefinition.apiextensions.k8s.io/perconaxtradbbackups.pxc.percona.com configured
role.rbac.authorization.k8s.io/percona-xtradb-cluster-operator unchanged
serviceaccount/percona-xtradb-cluster-operator created
rolebinding.rbac.authorization.k8s.io/service-account-percona-xtradb-cluster-operator created
deployment.apps/percona-xtradb-cluster-operator configured
```

This is worth noting that you have now several custom resource definitions
```
kubectl get crd  | grep percona
perconaxtradbbackups.pxc.percona.com           2019-08-20T12:49:28Z
perconaxtradbclusterbackups.pxc.percona.com    2019-08-20T12:49:28Z
perconaxtradbclusterrestores.pxc.percona.com   2019-08-20T12:49:28Z
perconaxtradbclusters.pxc.percona.com          2019-08-20T12:49:28Z
```

The full name is really long so we'll make use of the short names:
```
kubectl describe crd perconaxtradbclusters.pxc.percona.com | grep -A 4 "Short Names"
    Short Names:
      pxc
      pxcs
    Singular:  perconaxtradbcluster
  Scope:       Namespaced
--
    Short Names:
      pxc
      pxcs
    Singular:  perconaxtradbcluster
  Conditions:
```

### Test secrets

**!!Note** : do not use in a production cluster
```
$ kubectl apply -n demo -f operators/percona-xtradb-cluster-operator/03-tls-secrets.yaml
$ kubectl apply -n demo -f operators/percona-xtradb-cluster-operator/02-secrets.yaml
```

### Install the pxc cluster "cluster1"

$ kubectl apply -n demo -f operators/percona-xtradb-cluster-operator/04-pxc-cluster.yaml
perconaxtradbcluster.pxc.percona.com/cluster1 created
smana@dm-smana[±|master U:3 ?:4 ✗]:~/sources/gitops-flux-demo $ kubectl get po
NAME                                                   READY   STATUS              RESTARTS   AGE
cluster1-proxysql-0                                    4/4     Running             0          27s
cluster1-proxysql-1                                    0/4     ContainerCreating   0          6s
cluster1-pxc-0                                         1/2     Running             0          27s
cluster1-xb-cron-cluster1-20190911000003-kip9z-ldftm   0/1     Completed           0          2d7h
cluster1-xb-cron-cluster1-20190912000009-kip9z-77f52   0/1     Completed           0          31h
cluster1-xb-cron-cluster1-20190913000010-kip9z-bgnc5   0/1     Completed           0          7h6m
percona-xtradb-cluster-operator-f48d549c9-r9vm5        1/1     Running             0          2d11h
pxc-monitoring-0                                       1/1     Running             0          10m



smana@dm-smana[±|master U:3 ?:4 ✗]:~/sources/gitops-flux-demo $ kubectl get statefulset -n pxc
NAME                DESIRED   CURRENT   AGE
cluster1-proxysql   3         1         21s
cluster1-pxc        3         1         22s


```
$ kubectl get po
NAME                                              READY   STATUS    RESTARTS   AGE
cluster1-proxysql-0                               3/3     Running   0          2m28s
cluster1-proxysql-1                               3/3     Running   0          2m2s
cluster1-proxysql-2                               3/3     Running   0          95s
cluster1-pxc-0                                    1/1     Running   0          2m29s
cluster1-pxc-1                                    1/1     Running   0          101s
cluster1-pxc-2                                    1/1     Running   0          44s
percona-xtradb-cluster-operator-f48d549c9-r9vm5   1/1     Running   0          11m
```

```
$ kubectl run -ti --rm percona-client --image=percona:5.7 --restart=Never -- mysql -h cluster1-proxysql -uroot -proot_password
```

### Testing a backup / restore

kubectl run -ti --rm percona-client --image=percona:5.7 --restart=Never -- mysql -h cluster1-proxysql -uroot -proot_password
If you don't see a command prompt, try pressing enter.
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database orleans;
Query OK, 1 row affected (0.01 sec)

mysql> sho
    ->  ^^C
^C
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| orleans            |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql> Bye
pod "percona-client" deleted
smana@dm-smana[±|master U:3 ?:4 ✗]:~/sources/gitops-flux-demo $ kubectl apply -f operators/percona-xtradb-cluster-operator/05-backup.yaml
perconaxtradbclusterbackup.pxc.percona.com/backup1 created
smana@dm-smana[±|master U:3 ?:4 ✗]:~/sources/gitops-flux-demo $ kubectl get perconaxtradbclusterbackup
NAME      CLUSTER    STORAGE   DESTINATION               STATUS    COMPLETED   AGE
backup1   cluster1   fs-pvc    pvc/cluster1-xb-backup1   Running               8s
smana@dm-smana[±|master U:3 ?:4 ✗]:~/sources/gitops-flux-demo $ kubectl get po
NAME                                              READY   STATUS              RESTARTS   AGE
cluster1-proxysql-0                               3/3     Running             0          26m
cluster1-proxysql-1                               3/3     Running             0          25m
cluster1-proxysql-2                               3/3     Running             0          25m
cluster1-pxc-0                                    1/1     Running             0          26m
cluster1-pxc-1                                    1/1     Running             0          25m
cluster1-pxc-2                                    1/1     Running             0          24m
cluster1-xb-backup1-x7ltc                         0/1     ContainerCreating   0          11s
percona-xtradb-cluster-operator-f48d549c9-r9vm5   1/1     Running             0          35m


kubectl get po -l job-name=cluster1-xb-backup1
NAME                        READY   STATUS      RESTARTS   AGE
cluster1-xb-backup1-x7ltc   0/1     Completed   0          76s

kubectl get pvc cluster1-xb-backup1
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cluster1-xb-backup1   Bound    pvc-20649c5e-d404-11e9-bae1-42010a840071   6Gi        RWO            standard       2m5s


kubectl run -ti --rm percona-client --image=percona:5.7 --restart=Never -- mysql -h cluster1-proxysql -uroot -proot_password
If you don't see a command prompt, try pressing enter.

mysql> drop database orleans;
Query OK, 0 rows affected (0.04 sec)

mysql>


```
kubectl logs percona-xtradb-cluster-operator-f48d549c9-r9vm5 -f | grep restore
{"level":"info","ts":1568142889.264852,"logger":"kubebuilder.controller","caller":"controller/controller.go:120","msg":"Starting EventSource","Controller":"perconaxtradbclusterrestore-controller","Source":{"Type":{"metadata":{"creationTimestamp":null},"spec":{"pxcCluster":"","backupName":""},"status":{}}}}
{"level":"info","ts":1568142889.3657274,"logger":"kubebuilder.controller","caller":"controller/controller.go:134","msg":"Starting Controller","Controller":"perconaxtradbclusterrestore-controller"}
{"level":"info","ts":1568142889.4661558,"logger":"kubebuilder.controller","caller":"controller/controller.go:153","msg":"Starting workers","Controller":"perconaxtradbclusterrestore-controller","WorkerCount":1}
{"level":"info","ts":1568142889.4664276,"logger":"controller_perconaxtradbclusterrestore","caller":"pxcrestore/controller.go:75","msg":"backup restore request","namespace":"pxc","restore":"restore1"}
{"level":"info","ts":1568143192.2718492,"logger":"controller_perconaxtradbclusterrestore","caller":"pxcrestore/controller.go:75","msg":"backup restore request","namespace":"pxc","restore":"restore1"}
{"level":"info","ts":1568145232.3128026,"logger":"controller_perconaxtradbclusterrestore","caller":"pxcrestore/controller.go:75","msg":"backup restore request","namespace":"pxc","restore":"restore1"}
{"level":"info","ts":1568145232.3199258,"logger":"controller_perconaxtradbclusterrestore","caller":"pxcrestore/controller.go:158","msg":"stopping cluster","namespace":"pxc","restore":"restore1","cluster":"cluster1"}
{"level":"info","ts":1568145307.3758843,"logger":"controller_perconaxtradbclusterrestore","caller":"pxcrestore/controller.go:170","msg":"starting restore","namespace":"pxc","restore":"restore1","cluster":"cluster1","backup":"backup1"}
{"level":"info","ts":1568145335.475545,"logger":"controller_perconaxtradbclusterrestore","caller":"pxcrestore/controller.go:182","msg":"starting cluster","namespace":"pxc","restore":"restore1","cluster":"cluster1"}
{"level":"info","ts":1568145335.4927638,"logger":"controller_perconaxtradbclusterrestore","caller":"pxcrestore/controller.go:194","msg":"You can view xtrabackup log:\n$ kubectl logs job/restore-job-restore1-cluster1\nIf everything is fine, you can cleanup the job:\n$ kubectl delete pxc-restore/restore1\n","namespace":"pxc","restore":"restore1"}
{"level":"info","ts":1568145335.4999764,"logger":"controller_perconaxtradbclusterrestore","caller":"pxcrestore/controller.go:75","msg":"backup restore request","namespace":"pxc","restore":"restore1"}
```

```
kubectl get po
NAME                                              READY   STATUS              RESTARTS   AGE
cluster1-proxysql-0                               3/3     Running             0          26s
cluster1-proxysql-1                               0/3     ContainerCreating   0          11s
cluster1-pxc-0                                    0/1     Running             0          27s
cluster1-xb-backup1-x7ltc                         0/1     Completed           0          6m22s
percona-xtradb-cluster-operator-f48d549c9-r9vm5   1/1     Running             0          41m
restore-job-restore1-cluster1-944rv               0/1     Completed           0          44s
restore-src-restore1-cluster1                     1/1     Terminating         1          55s
```

```
kubectl run -ti --rm percona-client --image=percona:5.7 --restart=Never -- mysql -h cluster1-proxysql -uroot -proot_password
If you don't see a command prompt, try pressing enter.
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| orleans            |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql>
```