### Running minio container
```
$ docker pull minio/minio
$ docker volume create data
$ docker run -d --name minio -p 9000:9000 -v data:/data minio/minio server /data
```

### Grab access and secret key
/data/.minio.sys/config/config.json
```
$ docker exec -it minio cat /data/.minio.sys/config/config.json | jq .credentials._[]

{
  "key": "access_key",
  "value": "minioadmin"
}
{
  "key": "secret_key",
  "value": "minioadmin"
}
```

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
```
velero install \
   --provider aws \
   --plugins velero/velero-plugin-for-aws:v1.0.0 \
   --bucket kubedemo \
   --secret-file ./minio.credentials \
   --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<ip>:9000
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
### Enable tab completion for preferred shell
```
source <(velero completion zsh)
```
