## Docker 安装

```shell
https://docs.docker.com/engine/install/ubuntu/
```



## go安装

1. 下载go

   ```shell
   wget https://studygolang.com/dl/golang/go1.20.2.linux-amd64.tar.gz
   ```

2. 解压

   ```shell
   sudo tar -zxvf go1.20.2.linux-amd64.tar.gz -C /usr/local/lib/	
   ```

3. 配置go环境变量

   ```shell
   tee -a ~/.bashrc << EOF
   export GOROOT=/usr/local/lib/go/
   export GOPATH=/home/${USER}/go
   export PATH=\$PATH:\$GOROOT/bin:\$GOPATH/bin
   EOF
   ```

   ```shell
   source ~/.bashrc
   ```

4. go env 设置

   ```shell
   go env -w GO111MODULE=on
   ```
   
   ```shell
   go env -w GOPROXY=https://goproxy.io
   ```
   
   

## cri-dockerd安装

1. 安装

   ```shell
   https://github.com/Mirantis/cri-dockerd
   ```

2. 设置ExecStart

   ```shell
   vim /etc/systemd/system/cri-docker.service
   ```

3. 修改参数

   ```shell
   ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/cloudneedle/pause:3.9
   ```



## host 准备

```shell
vim /etc/hosts
```



## 关闭swap

临时关闭: 

```shell
sudo swapoff -a
```

永久关闭：

```shell
 vi /etc/fstab 注释 swap所在行，即swap所在行前增加 # 即可
```



## 开启k8s所需防火墙断端口

```shell
https://kubernetes.io/zh-cn/docs/reference/networking/ports-and-protocols/
```



## 加载ipvs模块

检查ipvs是否启用

```shell
lsmod | grep ip_vs
```

临时启用

```shell
for i in $(ls /lib/modules/$(uname -r)/kernel/net/netfilter/ipvs|grep -o "^[^.]*");do echo $i; /sbin/modinfo -F filename $i >/dev/null 2>&1 && /sbin/modprobe $i; done
```

永久启用

```shell
ls /lib/modules/$(uname -r)/kernel/net/netfilter/ipvs|grep -o "^[^.]*" >> /etc/modules
```



## 内核参数修改

```shell
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF
```



## ip_forward转发功能

1. 生效

   ```shell
   modprobe br_netfilter
   sysctl -p /etc/sysctl.d/k8s.conf
   ```

2. 配置模块加载永久生效(重启也生效)

   ```shell
   cat <<EOF | tee /etc/modules-load.d/k8s.conf
   br_netfilter
   EOF
   ```



## 增加一些工具

```shell
apt install -y ipvsadm ipset
```



## Kubeadm kubelet and kubectl 安装

1. 添加阿里云k8s源

   ```shell
   curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
   ```

   ```shell
   vi /etc/apt/sources.list
   ```

   ```shell
   deb https://mirrors.aliyun.com/kubernetes/apt/  kubernetes-xenial main
   ```

   ```shell
   apt update -y
   ```

2. 查看可用版本

   ```shell
   apt-cache madison kubeadm
   ```

3. 安装最新版本

   ```shell
   apt -y install kubectl kubelet kubeadm
   ```

4. 安装指定版本

   ```shell
   apt install -y kubelet=1.26.2-00 kubeadm=1.26.2-00 kubectl=1.26.2-00
   ```

## 启动kubelet

1. 修改cgroup

   ```shell
   cat > /etc/default/kubelet <<EOF
   KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --fail-swap-on=false
   EOF
   ```

1. 重新加载

   ```shell
   systemctl daemon-reload
   ```
   
   ```shell
   systemctl start kubelet.service
   ```
   
   ```shell
   systemctl enable kubelet.service
   ```
   
   ```shell
   systemctl status kubelet.service
   ```
   
   
   
   
   
   
   
   
