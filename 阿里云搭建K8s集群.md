## 创建云主机

在阿里云上新建1台操作云主机和3台以上集群云主机，注意应尽量让集群主机分属不少于三个的可用区。（以下按6台集群主机，3台master3台worker表述）。
操作用主机配置一个公网ip，集群主机不配置公网ip。
所有主机，以及以下所添加的其他阿里云资源，都需要属于同一个VPC网络。

登录操作主机，创建操作用户（不要用root操作）
编辑/etc/hosts，将所有云主机ip匹配上主机名

```
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

10.10.5.161 terminal
10.10.5.155 master-1
10.10.5.156 master-2
10.10.5.157 master-3
10.10.5.158 worker-1
10.10.5.159 worker-2
10.10.5.160 worker-3
```

将集群主机密钥拷贝到.ssh中，确保操作主机能无密码ssh到各集群主机上。
```
ssh root@master-1
```

## 创建一个私网CLB负载均衡器用于k8s-apiserver

![创建一个私网CLB负载均衡器用于k8s-apiserver](images/k8s-01.PNG?raw=true)

为集群master主机添加一组虚拟服务器

![为集群master主机添加一组虚拟服务器](images/k8s-02.JPEG?raw=true)

添加一项监听，关联刚添加的虚拟服务器组

![添加一项监听，关联刚添加的虚拟服务器组](images/k8s-03.PNG?raw=true)

在「云解析PrivateZone」里，添加一个内网域名，关联至解析，并关联至刚添加的CLB

![在「云解析PrivateZone」里，添加一个内网域名，关联至解析，并关联至刚添加的CLB](images/k8s-04.png?raw=true)

## 创建一个公网NAT网关供node访问外网

集群主机没有公网ip，所以需要在VPC上创建NAT网关来访问外网内容。

![集群主机没有公网ip，所以需要在VPC上创建NAT网关来访问外网内容。](images/k8s-05.png?raw=true)

## 配置kubespray

在操作机上git clone kubespray:
```
git clone https://github.com/kubernetes-sigs/kubespray.git
```

然后切换到要用的版本tag。这里v2.26.0，对应k8s版本1.30.4:
```
cd kubespray
git checkout v2.26.0
```

在kubespray目录下，复制inventory/sample文件夹，并编辑inventory.ini文件
```
cp -rf inventory/sample inventory/smyze
vim inventory/smyze/inventory.ini
```

配置集群node信息
```
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
master-1 ansible_host=10.10.5.155
master-2 ansible_host=10.10.5.156
master-3 ansible_host=10.10.5.157
worker-1 ansible_host=10.10.5.158
worker-2 ansible_host=10.10.5.159
worker-3 ansible_host=10.10.5.160
# node1 ansible_host=95.54.0.12  # ip=10.3.0.1 etcd_member_name=etcd1
# node2 ansible_host=95.54.0.13  # ip=10.3.0.2 etcd_member_name=etcd2
# node3 ansible_host=95.54.0.14  # ip=10.3.0.3 etcd_member_name=etcd3
# node4 ansible_host=95.54.0.15  # ip=10.3.0.4 etcd_member_name=etcd4
# node5 ansible_host=95.54.0.16  # ip=10.3.0.5 etcd_member_name=etcd5
# node6 ansible_host=95.54.0.17  # ip=10.3.0.6 etcd_member_name=etcd6

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube_control_plane]
master-1
master-2
master-3
# node1
# node2
# node3

[etcd]
master-1
master-2
master-3
# node1
# node2
# node3

[kube_node]
worker-1
worker-2
worker-3
# node2
# node3
# node4
# node5
# node6

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

编辑inventory/smyze/group_vars/k8s_cluster下的k8s-cluster.yml，addons.yml，编辑以下内容（[参考文档](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ingress/kube-vip.md)）
```
#k8s-cluster.yml
kube_proxy_strict_arp: true

#addons.yml
kube_vip_enabled: true
kube_vip_arp_enabled: true
kube_vip_controlplane_enabled: true
kube_vip_address: 10.10.3.14   #apiserver负载均衡器的ip
apiserver_loadbalancer_domain_name: "lb-apiserver.k8s-prod.smyze.cn"   #内网域名
loadbalancer_apiserver:
  address: "{{ kube_vip_address }}"
  port: 6443
kube_vip_interface: eth0      #ip link show确认network interface
kube_vip_services_enabled: false
```

国内无法直接下载k8s需要用到的镜像和文件，所以要将这些镜像下载下来放到阿里云的镜像仓库和OSS上。具体做法看这里：[迁移K8S镜像文件到墙内](https://github.com/swolfod/Documentation/blob/main/%E8%BF%81%E7%A7%BBK8S%E9%95%9C%E5%83%8F%E6%96%87%E4%BB%B6%E5%88%B0%E5%A2%99%E5%86%85.md)

编辑inventory/smyze/group_vars/all/offline.yml（注意xxx_download_url要用images.list.template里的内容加上{{ files_repo }}拼接而成

```
## Global Offline settings
### Private Container Image Registry
registry_host: "bucket-registry.cn-shanghai.cr.aliyuncs.com/k8s"
files_repo: "https://smyzek8s.oss-rg-china-mainland.aliyuncs.com"
### If using CentOS, RedHat, AlmaLinux or Fedora
# yum_repo: "http://myinternalyumrepo"
### If using Debian
# debian_repo: "http://myinternaldebianrepo"
### If using Ubuntu
# ubuntu_repo: "http://myinternalubunturepo"

## Container Registry overrides
kube_image_repo: "{{ registry_host }}"
gcr_image_repo: "{{ registry_host }}"
github_image_repo: "{{ registry_host }}"
docker_image_repo: "{{ registry_host }}"
quay_image_repo: "{{ registry_host }}"

## Kubernetes components
kubeadm_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubeadm"
kubectl_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubectl"
kubelet_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubelet"


## Two options - Override entire repository or override only a single binary.

## [Optional] 1 - Override entire binary repository
# github_url: "https://my_github_proxy"
# dl_k8s_io_url: "https://my_dl_k8s_io_proxy"
# storage_googleapis_url: "https://my_storage_googleapi_proxy"
# get_helm_url: "https://my_helm_sh_proxy"

## [Optional] 2 - Override a specific binary
## CNI Plugins
cni_download_url: "{{ files_repo }}/github.com/containernetworking/plugins/releases/download/{{ cni_version }}/cni-plugins-linux-{{ image_arch }}-{{ cni_version }}.tgz"

## cri-tools
crictl_download_url: "{{ files_repo }}/github.com/kubernetes-sigs/cri-tools/releases/download/{{ crictl_version }}/crictl-{{ crictl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"

## [Optional] etcd: only if you use etcd_deployment=host
etcd_download_url: "{{ files_repo }}/github.com/etcd-io/etcd/releases/download/{{ etcd_version }}/etcd-{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"

# [Optional] Calico: If using Calico network plugin
calicoctl_download_url: "{{ files_repo }}/github.com/projectcalico/calico/releases/download/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
# [Optional] Calico with kdd: If using Calico network plugin with kdd datastore
calico_crds_download_url: "{{ files_repo }}/github.com/projectcalico/calico/archive/{{ calico_version }}.tar.gz"

# [Optional] Cilium: If using Cilium network plugin
# ciliumcli_download_url: "{{ files_repo }}/github.com/cilium/cilium-cli/releases/download/{{ cilium_cli_version }}/cilium-linux-{{ image_arch }}.tar.gz"

# [Optional] helm: only if you set helm_enabled: true
helm_download_url: "{{ files_repo }}/get.helm.sh/helm-{{ helm_version }}-linux-{{ image_arch }}.tar.gz"

# [Optional] crun: only if you set crun_enabled: true
crun_download_url: "{{ files_repo }}/github.com/containers/crun/releases/download/{{ crun_version }}/crun-{{ crun_version }}-linux-{{ image_arch }}"

# [Optional] kata: only if you set kata_containers_enabled: true
kata_containers_download_url: "{{ files_repo }}/github.com/kata-containers/kata-containers/releases/download/{{ kata_containers_version }}/kata-static-{{ kata_containers_version }}-{{ ansible_architecture }}.tar.xz"

# [Optional] cri-dockerd: only if you set container_manager: docker
cri_dockerd_download_url: "{{ files_repo }}/github.com/Mirantis/cri-dockerd/releases/download/v{{ cri_dockerd_version }}/cri-dockerd-{{ cri_dockerd_version }}.{{ image_arch }}.tgz"

# [Optional] runc: if you set container_manager to containerd or crio
runc_download_url: "{{ files_repo }}/github.com/opencontainers/runc/releases/download/{{ runc_version }}/runc.{{ image_arch }}"

# [Optional] cri-o: only if you set container_manager: crio
# crio_download_base: "download.opensuse.org/repositories/devel:kubic:libcontainers:stable"
# crio_download_crio: "http://{{ crio_download_base }}:/cri-o:/"
# crio_download_url: "{{ files_repo }}/storage.googleapis.com/cri-o/artifacts/cri-o.{{ image_arch }}.{{ crio_version }}.tar.gz"
# skopeo_download_url: "{{ files_repo }}/github.com/lework/skopeo-binary/releases/download/{{ skopeo_version }}/skopeo-linux-{{ image_arch }}"

# [Optional] containerd: only if you set container_runtime: containerd
containerd_download_url: "{{ files_repo }}/github.com/containerd/containerd/releases/download/v{{ containerd_version }}/containerd-{{ containerd_version }}-linux-{{ image_arch }}.tar.gz"
nerdctl_download_url: "{{ files_repo }}/github.com/containerd/nerdctl/releases/download/v{{ nerdctl_version }}/nerdctl-{{ nerdctl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"

# [Optional] runsc,containerd-shim-runsc: only if you set gvisor_enabled: true
gvisor_runsc_download_url: "{{ files_repo }}/storage.googleapis.com/gvisor/releases/release/{{ gvisor_version }}/{{ ansible_architecture }}/runsc"
gvisor_containerd_shim_runsc_download_url: "{{ files_repo }}/storage.googleapis.com/gvisor/releases/release/{{ gvisor_version }}/{{ ansible_architecture }}/containerd-shim-runsc-v1"

# [Optional] Krew: only if you set krew_enabled: true
krew_download_url: "{{ files_repo }}/github.com/kubernetes-sigs/krew/releases/download/{{ krew_version }}/krew-{{ host_os }}_{{ image_arch }}.tar.gz"

## CentOS/Redhat/AlmaLinux
### For EL8, baseos and appstream must be available,
### By default we enable those repo automatically
# rhel_enable_repos: false
### Docker / Containerd
# docker_rh_repo_base_url: "{{ yum_repo }}/docker-ce/$releasever/$basearch"
# docker_rh_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"

## Fedora
### Docker
# docker_fedora_repo_base_url: "{{ yum_repo }}/docker-ce/{{ ansible_distribution_major_version }}/{{ ansible_architecture }}"
# docker_fedora_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"
### Containerd
# containerd_fedora_repo_base_url: "{{ yum_repo }}/containerd"
# containerd_fedora_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"

## Debian
### Docker
# docker_debian_repo_base_url: "{{ debian_repo }}/docker-ce"
# docker_debian_repo_gpgkey: "{{ debian_repo }}/docker-ce/gpg"
### Containerd
# containerd_debian_repo_base_url: "{{ debian_repo }}/containerd"
# containerd_debian_repo_gpgkey: "{{ debian_repo }}/containerd/gpg"
# containerd_debian_repo_repokey: 'YOURREPOKEY'

## Ubuntu
### Docker
# docker_ubuntu_repo_base_url: "{{ ubuntu_repo }}/docker-ce"
# docker_ubuntu_repo_gpgkey: "{{ ubuntu_repo }}/docker-ce/gpg"
### Containerd
# containerd_ubuntu_repo_base_url: "{{ ubuntu_repo }}/containerd"
# containerd_ubuntu_repo_gpgkey: "{{ ubuntu_repo }}/containerd/gpg"
# containerd_ubuntu_repo_repokey: 'YOURREPOKEY'
```

配置python环境
```
sudo apt install python3.12-venv
###.....

python3 -m venv ~/.venv/k8s
alias k8s-activate='source /home/smyze/.venv/k8s/bin/activate' #将这行加到~/.bashrc中
k8s-activate

pip install -r requirements.txt
```

全部配置完成后，部署k8s集群
```
ansible-playbook -i inventory/smyze/inventory.ini --become --become-user=root --user=root cluster.yml
```

搭建完成后，安装kubectl并从集群主机上复制集群配置到操作主机上，并验证集群状态
```
sudo snap install kubectl --classic
mkdir ~/.kube
scp root@master-1:~/.kube/config ~/.kube/

kubectl get nodes
kubectl get pods -A
```

配置公网负载均衡
安装nginx-ingress
```
kubectl apply -f ingress-nginx.yaml
```

配置文件: [ingress-nginx.yaml](lib/ingress-nginx.yaml)（[来源](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml)，将镜像上传到阿里云并替换）：


查看ingress-nginx负载均衡的端口：
```
kubectl get svc -A

NAMESPACE       NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
#...
ingress-nginx   ingress-nginx-controller             LoadBalancer   10.233.6.160    <pending>     80:30530/TCP,443:31776/TCP   34s
#...
```

在阿里云上创建一个公网负载均衡器：

![在阿里云上创建一个公网负载均衡器](images/k8s-06.png?raw=true)

创建两组虚拟服务器组，分别匹配到集群中所有worker主机的ingress-nginx负载均衡http和https端口

![创建两组虚拟服务器组，分别匹配到集群中所有worker主机的ingress-nginx负载均衡http和https端口](images/k8s-07.png?raw=true)

![创建两组虚拟服务器组，分别匹配到集群中所有worker主机的ingress-nginx负载均衡http和https端口](images/k8s-08.png?raw=true)

创建80和443端口的监听，并匹配到相应的虚拟服务器组

![创建80和443端口的监听，并匹配到相应的虚拟服务器组](images/k8s-09.png?raw=true)

测试负载均衡，添加一条DNS域名解析到公网负载均衡器：

![测试负载均衡，添加一条DNS域名解析到公网负载均衡器](images/k8s-10.png?raw=true)

在集群中启动一个nginx服务，注意将ingress.yaml里的host配置项改为新添加的域名
```
tar zxvf sampleapp.tar.gz
kubectl apply -f sampleapp
```

[测试配置包](lib/sampleapp.tar.gz)

确认服务pod启动后，在浏览器中访问新添加的域名，应该能看到nginx欢迎页面，证明部署成功
![nginx欢迎页面](images/k8s-11.png?raw=true)

## 添加阿里云SLS日志
[如何在自建Kubernetes集群中安装Logtail组件_日志服务(SLS)-阿里云帮助中心](https://help.aliyun.com/zh/sls/user-guide/install-the-logtail-component-self-built-kubernetes-cluster)
