参考文档：

[https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/offline](https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/offline)

[https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/offline](https://imroc.cc/kubernetes/deploy/kubespray/offline)

在阿里云海外区（例如香港）创建云主机，注意主机系统要和部署环境的主机一致

在操作机上git clone kubespray:
```
git clone https://github.com/kubernetes-sigs/kubespray.git
```
然后切换到要用的版本tag。这里v2.26.0，对应k8s版本1.30:

```
cd kubespray
git checkout v2.26.0
```

在kubespray/contrib/offline目录下，执行generate_list.sh

登录阿里云镜像仓库，然后将images.list中的所有镜像pull下来再上传到阿里云

```
docker login --username=麦克可智能科技 bucket-registry.cn-shanghai.cr.aliyuncs.com

for image in $(cat temp/images.list); do  sudo docker pull ${image}; sudo docker tag ${image} bucket-registry.cn-shanghai.cr.aliyuncs.com/k8s/${image#*/}; sudo docker push bucket-registry.cn-shanghai.cr.aliyuncs.com/k8s/${image#*/}; done
```

然后将files.list中所有文件下载下来，再上传到OSS
```
wget -x -P temp/files -i temp/files.list
```
注意上传的镜像仓库和OSS都要有匿名公共读权限。
