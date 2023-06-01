### 环境

###### 服务器：ubuntu 22.04 TSL

###### K8S版本：1.27.x

###### Docker版本: 23.x



### 安装与设置

###### Docker

> 参照官网文档进行配置安装

```shell
https://docs.docker.com/engine/install/ubuntu/
```

> 配置cgroupdriver=systemd和镜像源

```shell
sudo cat > /etc/docker/daemon.json << EOF
{
"registry-mirrors": [
"https://i7m337pd.mirror.aliyuncs.com",
"https://docker.mirrors.ustc.edu.cn",
"https://hub-mirror.c.163.com",
"https://reg-mirror.qiniu.com",
"https://registry.docker-cn.com"
],
"exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
```

> 重启Docker

```shell
systemctl restart docker.service 
```



###### ubuntu

> 关闭swap(所有节点)

```shell
sudo swapoff -a  
```

```shell
sudo sed -i '/swap/s/^/#/' /etc/fstab
```

> 开启IPv4转发

```shell
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```shell
modprobe overlay
modprobe br_netfilter
```

```shell
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

```shell
sudo sysctl --system
```

> IPVS 开启

- 安装ipset和ipvsadm：

```shell
apt install -y ipset ipvsadm
```

- 开启ipvs 转发

```shell
sudo modprobe br_netfilter
```

- 配置加载模块

```shell
cat > /etc/modules-load.d/ipvs.conf << EOF
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF
```

- 临时加载

```shel
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
```

- 开机加载配置，将ipvs相关模块加入配置文件中

```shell
cat >> /etc/modules <<EOF
ip_vs_sh
ip_vs_wrr
ip_vs_rr
ip_vs
nf_conntrack
EOF
```

- 查看 ip_vs 是否加载进入内核中

```shell
lsmod | grep -e ip_vs -e nf_conntrack
```

> 防火墙： ubuntu默认是关闭的，如果要开启防火墙，必须开放K8S所需端口，具体端口参照官方文档：
>
> https://kubernetes.io/zh-cn/docs/reference/networking/ports-and-protocols/

查看防火墙状态

```shell
sudo ufw status
```

> 安装cri-dockerd

下载软件包，直接从github下载速度较慢，这里使用了代理加速：

```shell
wget https://ghproxy.com/https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb
```

安装软件包

```shell
sudo dpkg -i cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb
```

调整启动参数

```shell
sudo sed -i -e 's#ExecStart=.*#ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9#g' /lib/systemd/system/cri-docker.service
```

```shell
sudo systemctl daemon-reload
```

设置开启自启动

```shell
sudo systemctl enable cri-docker
```



###### K8S

> 配置阿里云镜像站点，（阿里镜像下载）也可以用官方方式下载

```shell
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat >/etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
```

```shell
apt-get update
```

查看可用版本

```shell
apt-cache madison kubeadm|head
```

 安装最新版本

```shell
apt -y install kubectl kubelet kubeadm
```

> 安装指定版本
>
> ```shell
> apt install -y kubelet=1.26.3-00 kubeadm=1.26.3-00 kubectl=1.26.3-00
> ```

> ##### kubeadm init 初始化

生成默认配置，便于修改

```shell
kubeadm config print init-defaults > kubeadm.yaml
```

文件修改对应如下

```shell
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef # 可以自定义，正则([a-z0-9]{6}).([a-z0-9]{16})
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4 # kubernetes主节点IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/cri-dockerd.sock # docker
  imagePullPolicy: IfNotPresent
  name: k8s-matser # 节点的hostname
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers # 镜像仓库
kind: ClusterConfiguration
kubernetesVersion: 1.27.1 # 指定版本
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.1.0.0/16  # 增加指定pod的网段
scheduler: {}
---
# 使用ipvs
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
# 指定cgroup
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

使用文件配置方式初始化

```shell
kubeadm init --config ./kubeadm.yaml 
```

根据显示提示进行操作(要开始使用集群，需要以普通用户身份运行以下命令)

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 安装Pod网络插件Calico(主节点)

```shell
wget https://ghproxy.com/https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
```

```shell
kubectl create -f tigera-operator.yaml
```

```shell
wget https://ghproxy.com/https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
```

修改cidr配置

```shell
vim custon-resources.yaml
```

将spec.calicoNetwork.ipPools.cidr值设置为初始化集群时--pod-network-cidr参数指定的网段：

```shell
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16 # 修改此处
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

