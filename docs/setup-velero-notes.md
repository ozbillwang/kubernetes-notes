### Running minio container
```
docker pull minio/minio
docker volume create data
docker run -d --name minio -p 9000:9000 -v data:/data minio/minio server /data
```

### Grab access and secret key
/data/.minio.sys/config/config.json
```
docker exec -it minio cat /data/.minio.sys/config/config.json | jq .credentials._[]

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
brew install velero
```

### Create credentials file (Needed for velero initialization)
```
cat <<EOF>> minio.credentials
[default]
aws_access_key_id=minioadmin
aws_secret_access_key=minioadmin
EOF
```

### make sure your kubernetes cluster ready.

It could be minikube, kops, docker desktop with kubernetes enabled, etc.

### Install Velero in the Kubernetes Cluster
```
velero install \
   --provider aws \
   --bucket kubedemo \
   --secret-file ./minio.credentials \
   --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<ip>:9000
```

### Enable tab completion for preferred shell
```
source <(velero completion zsh)
```
