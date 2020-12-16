### Running minio container
```
$ helm repo add minio https://helm.min.io/
$ helm install --set accessKey=minioadmin,secretKey=minioadmin --generate-name minio/minio
```
Get its service IP

```
$ kubectl get service
service/minio-1608073655   ClusterIP   10.96.205.247   <none>        9000/TCP   16m
```
So the IP address is `10.96.205.247` in this case.

### install velero on MacOS
```
$ brew install velero
```
For other systems, go through its official online document https://velero.io/docs/main/basic-install/

### Create credentials file (Needed for velero initialization)
```
$ cat <<EOF>> minio.credentials

[default]
aws_access_key_id=minioadmin
aws_secret_access_key=minioadmin
EOF
```

### make sure your kubernetes cluster ready.

It could be minikube, kops, docker desktop with kubernetes enabled, AWS EKS, Azure AKS, etc.

```
$ velero version
Client:
	Version: v1.5.2
	Git commit: -
<error getting server version: no matches for kind "ServerStatusRequest" in version "velero.io/v1">
```
The reason is, velero server is not installed yet to kubernetes cluster.

### Install Velero in the Kubernetes Cluster

Replace s3url with real IP you get above.

```
velero install \
   --provider aws \
   --plugins velero/velero-plugin-for-aws:v1.0.0 \
   --bucket kubedemo \
   --secret-file ./minio.credentials \
   --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://10.96.205.247:9000
```
Velero server is installed in namespace `velero`
```
$ kk -n velero get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/velero-84b455d4db-8j8j7   1/1     Running   0          21h

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/velero   1/1     1            1           21h

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/velero-84b455d4db   1         1         1       21h
```

Now you are fine to check velero version
```
$ velero version
Client:
	Version: v1.5.2
	Git commit: -
Server:
	Version: v1.5.2
```
### check velero backup location

```
$ velero backup-location get

NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED                   ACCESS MODE
default   aws        kubedemo        Available   2020-12-16 10:25:48 +1100 AEDT   ReadWrite
```

Now we are ready to do the backup and restore.

### deploy nginx applicaiton

```
$ kubectl create ns testing
$ kubectl -n testing run nginx --image nginx --replicas 2
$ kubectl -n testing get all
```
### create velero backup

```
$ velero backup create firstbackup --include-namespaces testing

# Make sure status is completed.
$ velero backup get

NAME          STATUS      ERRORS   WARNINGS   CREATED                          EXPIRES   STORAGE LOCATION   SELECTOR
firstbackup   Completed   0        0          2020-12-16 10:37:35 +1100 AEDT   29d       default            <none>

$ velero backup describe firstbackup
Name:         firstbackup
Namespace:    velero
Labels:       velero.io/storage-location=default
Annotations:  velero.io/source-cluster-k8s-gitversion=v1.18.8
              velero.io/source-cluster-k8s-major-version=1
              velero.io/source-cluster-k8s-minor-version=18

Phase:  Completed

Errors:    0
Warnings:  0

Namespaces:
  Included:  testing
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        <none>
  Cluster-scoped:  auto

Label selector:  <none>


Storage Location:  default

Velero-Native Snapshot PVs:  auto

TTL:  720h0m0s

Hooks:  <none>

Backup Format Version:  1.1.0

Started:    2020-12-16 10:37:35 +1100 AEDT
Completed:  2020-12-16 10:37:36 +1100 AEDT

Expiration:  2021-01-15 10:37:35 +1100 AEDT

Total items to be backed up:  28
Items backed up:              28

Velero-Native Snapshots: <none included>

$ $ kubectl -n velero get backups
NAME          AGE
firstbackup   5m44s
```
### Test with restore

Delete the namespace `testing`
```
$ kubectl delete ns testing

$ kubectl get ns
```

Restore from backup
```
$ velero restore create firstrestore --from-backup firstbackup

$ kubectl -n testing get all
```
You get everything back in namespace `testing`

### schedule backup

```
$ velero schedule create test --schedule="@every 1m" --include-namespaces testing

$ velero schedule get
```
