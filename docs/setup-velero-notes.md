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

### velero backup

