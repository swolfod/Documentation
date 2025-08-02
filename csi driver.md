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


#### Step 2: 
