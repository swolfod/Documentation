### 阿里云ECS自建Kubernetes集群添加CSI Driver

#### Step 1: 给ECS绑定RAM权限策略


在阿里云后台'RAM访问控制’, 左侧点击'权限管理->权限策略', 点击’创建权限策略‘, 选择’脚本编辑‘，并填入以下权限然后保存。


```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:CreateDisk",
        "ecs:AttachDisk",
        "ecs:DetachDisk",
        "ecs:DeleteDisk",
        "ecs:DescribeDisks",
        "ecs:DescribeDiskAttribute",
        "ecs:ResizeDisk",
        "ecs:CreateSnapshot",
        "ecs:DeleteSnapshot"
      ],
      "Resource": "*"
    }
  ]
}
```


点击'身份管理->角色'，点击创建角色。在’信任主体类型‘中选择’云服务‘，然后在下方’信任主体名称‘中选择’云服务器 ECS (ecs.aliyuncs.com)‘，点击确认创建角色。然后点击进入新创建的角色详情，在下方’权限管理'栏点击‘新增授权’，添加刚创建的权限策略。


前往'云服务器 ECS'后台，找到k8s集群中所有的worker主机。点入每个worker主机的详情页，在’RAM 角色‘下点击’授予 / 收回 RAM 角色‘, 选择授予刚才创建的RAM角色。


#### Step 2: 为Ack One配置集群CSI环境


给集群所有worker节点打上标签


```bash
kubectl get nodes --selector='!node-role.kubernetes.io/master,!node-role.kubernetes.io/control-plane' -o name | xargs -I {} kubectl label {} alibabacloud.com/external=true
```


配置集群授权变量


```bash
# 因为ECS配置了RAM角色权限，所有授权变量都设为空字符串
kubectl create secret generic alibaba-addon-secret \
  --from-literal=access-key-id="" \
  --from-literal=access-key-secret="" \
  -n kube-system

kubectl create secret generic addon.csi.token \
  --from-literal=addon.token.config="" \
  -n kube-system
```


可用ssh连入各worker节点，用以下指令检测是否具有所需权限。


```bash
curl -s http://100.100.100.200/latest/meta-data/ram/security-credentials/
```


#### Step 3: 在Ack One后台安装csi-plugin和csi-provisioner


进去Ack One后台，在左侧菜单栏点击‘运维管理’->‘组件管理’，在‘存储’下点击安装csi-provisioner和csi-plugin。


csi-provisioner启动后，配置集群环境变量使用RAM角色


```bash
kubectl set env deployment/csi-provisioner -n kube-system ALIBABA_CLOUD_AUTH_TYPE=ecs_ram_role
```



