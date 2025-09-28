### 准备

```
# Verify cluster requirements
kubectl version --short  # Kubernetes 1.25-1.29
kubectl get nodes       # At least 3 nodes for production
kubectl get pods -n kube-system | grep csi  # CSI driver installed

# Create namespaces
kubectl create namespace elastic-system
kubectl create namespace elastic-stack
kubectl label namespace elastic-stack name=elastic-stack

# Verify ACK CSI driver is installed (should be present by default)
kubectl get pods -n kube-system | grep csi
```

### Step 1: 准备镜像

拉取Elasticsearch镜像，安装ik插件，然后上传到ACR
```
# Dockerfile
FROM docker.elastic.co/elasticsearch/elasticsearch:8.11.4

# Install IK Analysis plugin
RUN bin/elasticsearch-plugin install --batch \
    https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v8.11.4/elasticsearch-analysis-ik-8.11.4.zip

# Optional: Add custom dictionaries
# COPY dictionaries/custom_dict.txt config/analysis-ik/custom/
# COPY dictionaries/stopwords.txt config/analysis-ik/custom/

# Health check
HEALTHCHECK --interval=10s --timeout=5s --start-period=60s \
  CMD curl -f http://localhost:9200/_cluster/health || exit 1
```

### Step 2: 安装ECK Operator

从Elasticsearch官方地址下载[csr.yaml](https://download.elastic.co/downloads/eck/3.1.0/crds.yaml)和[operator.yaml](https://download.elastic.co/downloads/eck/3.1.0/operator.yaml)，注意将operator的image上传至国内容器库，并修改配置文件镜像地址。


```
# Install CRDs
kubectl create -f crds.yaml

# Install operator
kubectl apply -f operator.yaml

# Wait for operator
kubectl -n elastic-system wait --for=condition=ready pod --selector=control-plane=elastic-operator --timeout=300s

# Verify
kubectl -n elastic-system get pods
```

### Step 3: 创建Storageclass

storageclass.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elasticsearch-storage
provisioner: diskplugin.csi.alibabacloud.com
parameters:
  type: cloud_essd
  performanceLevel: PL1  # Options: PL0(10K), PL1(50K), PL2(100K) IOPS
  fsType: ext4
reclaimPolicy: Retain  # Protect data from accidental deletion
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

bash
```
kubectl apply -f storageclass.yaml
```

### Step 4: 部署生产环境ES集群

cluster.yaml
```
# elasticsearch-prod.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: elastic-stack
spec:
  version: 8.11.4
  image:  bucket-registry.cn-shanghai.cr.aliyuncs.com/k8s/elasticsearch/elasticsearch:8.11.4-ik
  
  # No security configuration
  http:
    service:
      spec:
        type: ClusterIP
    tls:
      selfSignedCertificate:
        disabled: true  # No TLS
  
  nodeSets:
  - name: default
    count: 3
    config:
      # Basic settings
      node.store.allow_mmap: false
      
      # DISABLE ALL SECURITY
      xpack.security.enabled: false
      xpack.security.enrollment.enabled: false
      xpack.security.http.ssl.enabled: false
      xpack.security.transport.ssl.enabled: false
      
      # Disable unused features
      xpack.monitoring.collection.enabled: false
      xpack.ml.enabled: false
      xpack.graph.enabled: false
      xpack.watcher.enabled: false
      
      # Performance tuning
      thread_pool.search.size: 2
      thread_pool.search.queue_size: 20
      thread_pool.write.size: 1
      thread_pool.write.queue_size: 20
      thread_pool.get.size: 1
      thread_pool.get.queue_size: 10
      
      # Cluster settings
      cluster.routing.allocation.disk.threshold_enabled: true
      cluster.routing.allocation.disk.watermark.low: "85%"
      cluster.routing.allocation.disk.watermark.high: "90%"
      cluster.routing.allocation.disk.watermark.flood_stage: "95%"
      
      # Recovery settings
      indices.recovery.max_bytes_per_sec: 100mb
      cluster.routing.allocation.node_concurrent_recoveries: 2
    
    podTemplate:
      metadata:
        labels:
          app: elasticsearch
      spec:
        imagePullSecrets:
        - name: acr-secret
        
        # Pod distribution across zones
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  elasticsearch.k8s.elastic.co/cluster-name: elasticsearch
              topologyKey: kubernetes.io/hostname
        
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 8Gi
              cpu: 1
            limits:
              memory: 8Gi
              cpu: 2
          env:
          - name: ES_JAVA_OPTS
            value: "-Xms4g -Xmx4g"
          readinessProbe:
            httpGet:
              path: /_cluster/health
              port: 9200
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
    
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: elasticsearch-storage
        resources:
          requests:
            storage: 100Gi
```

bash
```
kubectl apply -f elasticsearch-prod.yaml   # For production

# Monitor deployment, could take a while (over 20 minutes)
watch kubectl get elasticsearch,pods -n elastic-stack
```

### Step 5: 启动Service和NetworkPolicy

service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elastic-stack
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 9200
    targetPort: 9200
  selector:
    elasticsearch.k8s.elastic.co/cluster-name: elasticsearch
```

network-policy.yaml
```
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: elasticsearch-isolation
  namespace: elastic-stack
spec:
  podSelector:
    matchLabels:
      elasticsearch.k8s.elastic.co/cluster-name: elasticsearch
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow ECK operator
  - from:
    - namespaceSelector:
        matchLabels:
          name: elastic-system
    ports:
    - port: 9200
  
  # Allow your application namespaces
  - from:
    - namespaceSelector:
        matchLabels:
          name: smyze  # Label your app namespaces with this
    ports:
    - port: 9200
  
  egress:
  - {}  # Allow all outbound
```

bash
```
# Apply network configuration
kubectl apply -f services.yaml
kubectl apply -f network-policy.yaml

# Label your application namespace to allow access
kubectl label namespace smyze name=smyze
```
