### Step 1: [安装CSI驱动](https://github.com/swolfod/Documentation/blob/main/%E5%AE%89%E8%A3%85CSI%E9%A9%B1%E5%8A%A8.md)

### Step 2: 创建StorageClass


创建storageclass.yaml

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rabbitmq-storage
provisioner: diskplugin.csi.alibabacloud.com
parameters:
  type: cloud_essd
  performanceLevel: PL1
  fsType: ext4
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

然后在集群创建StorageClass

```bash
kubectl apply -f storageclass.yaml
```

从[RabbitMQ官方github](https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml)下载cluster-operator.yml，将image上传至国内容器库，并修改配置文件镜像地址后apply。

```bash
kubectl apply -f cluster-operator.yml
```

