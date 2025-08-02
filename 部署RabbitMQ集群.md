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

配置rabbitmq cluster.yaml并apply。

```
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-cluster
  namespace: smyze # Or your desired namespace
spec:
  replicas: 3 # Number of RabbitMQ nodes in the cluster
  image: bucket-registry.cn-shanghai.cr.aliyuncs.com/package/rabbitmq/rabbitmq:4.1.1-management
  
  # Resource requests and limits for RabbitMQ pods
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi
      
  # Persistent storage configuration
  persistence:
    storageClassName: rabbitmq-storage
    storage: 20Gi # Requested storage size for each RabbitMQ node

    
  # Enable management plugin (optional, but highly recommended for dashboard)
  rabbitmq:
    additionalPlugins:
      - rabbitmq_management
      - rabbitmq_peer_discovery_k8s # Essential for Kubernetes clustering
    additionalConfig: |
      default_user = {username}
      default_pass = {password}

  override:
    service:
      spec:
        ports:
          - name: http
            port: 15672
            targetPort: 15672
          - name: amqp
            port: 5672
            targetPort: 5672
```
