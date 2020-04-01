---
title: "kubeadm搭建部署kubernetes集群"
date: 2020-03-22T01:04:10+08:00
draft: false

categories: ['Code']
tags: ['Kubernetes', 'Docker']
author: "Fancy"
---

<!--more-->

> 零零散散看了不少k8s的文檔，搭建過幾次，每次都有不同的收獲，姿勢也是五花八門，網上關於K8s部署的文檔坑實在是太多了，想著能否好好總結下，學習下部署設計，便於自己查閱以及更好的學習。

目前kubeadm支持：

- Ubuntu 16.04+
- Debian 9+
- CentOS 7
- Red Hat Enterprise Linux (RHEL) 7
- Fedora 25+
- HypriotOS v1.0.1 +
- Container Linux (tested with 1800.6.0)

官方建议至少2G内存，要求双核及以上。并检查网络链接以及用的端口是否开放。更改下防火墙设置放行端口

我这里利用了`斐讯N1`刷的`Armbian`和`Raspberry Pi 3B+`的`HypriotOS`组成最小单元的集成作为示例，之所以选择这两个，主要是因为硬件便宜（家里已经4个N1了），另外想试试HypriotOS这么好用的Docker专用系统，同是debian系的也便于安装命令复用。刚好作为最小集群一个Master节点一个Node节点。

比如Item2 就使用⌘(command) + d 左右分屏，⌘(command) + ⇧(shift) + i 开启多窗口输入命令。省很多功夫。

首先按要求关闭Swap（性能考虑，不关闭会拖慢速度，官方说的是必须的）

```
sudo swapoff -a
```

关闭Selinux：

```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```



确保主机名（hostname）和MAC， product_uuid是否唯一的，这些是用于K8s辨别Node节点唯一性的值，都相同会导致安装失败，可以分别用以下命令查看
毕竟是ARM，我们的armbian内核是没有加载 dmi sysfs的，所以product_uuid无从谈起。

```
cat /etc/hostname 
sudo ifconfig -a
sudo cat /sys/class/dmi/id/product_uuid
```

我的建议是首先更改`Hostname`，后期便利很多。

```
# Master节点
sudo hostnamectl --static set-hostname k8s-master
# Node节点1
sudo hostnamectl --static set-hostname k8s-node1
```



### 修改桥接网络配置

 `lsmod | grep br_netfilter`. 检查br_netfilter是否运行，如果没有的话可以用 `modprobe br_netfilter`显式调用。然后运行防止iptables被绕过：

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

###  安装Docker

截至2020年3月30号，官方推荐的Docker版本是 `19.03.8`。我们通过`docker -v`确保版本是在能稳定运行k8s的`1.13.1, 17.03, 17.06, 17.09, 18.06, 18.09, 19.03.08 `之中。

安装Docker CE老生常谈了，Azure阿里云这些镜像加速毕竟是有N1的程序员我表示根本不需要，网上也有很多参考，这里可以使用一键脚本

```
curl -fsSL "https://get.docker.com/" | sh
systemctl enable --now docker
```

或者Docker官网https://docs.docker.com/install/linux/docker-ce/debian/根据不同发行版和架构安装，我贴的这份仅限Arm架构的Debian安装。

```bash
# 允许apt透过HTTPS使用软件包
apt update && apt install -y \
  apt-transport-https ca-certificates curl software-properties-common gnupg2

# 添加Docker的GPGkey
curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -

# 添加Docker apt软件包
add-apt-repository \
   "deb [arch=arm64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"

# 安装 Docker CE 及 Container.
apt-get update && apt-get install -y \
  containerd.io=1.2.13-1 \
  docker-ce=5:19.03.8~3-0~debian-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.8~3-0~debian-$(lsb_release -cs)
# sudo apt-get install docker-ce docker-ce-cli containerd.io

```

###配置镜像

修改`daemon.json`，配置cgroup驱动，与k8s一致，如果添加镜像加速的话添加多句` "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"],`之类，之后重启，如下：

```bash
# 这里使用systemd作为Docker croup
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# 创建Docker服务目录

mkdir -p /etc/systemd/system/docker.service.d

# 重启Docker，开启开机启动
systemctl daemon-reload
systemctl enable docker
systemctl restart docker

# 检查Docker
docker -v
# >>> Docker version 19.03.8, build afacb8b
docker run hello-world
```



### 安装Runtime

### kubeadm的方案

### 安装kubeadm, kubelet 和 kubectl（Master & Node）

Debian系：

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


systemctl enable kubelet && systemctl start kubelet
```

RedHat系：

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
```

### 配置Cgroup drive

检查docker的cgroup驱动，应与kubelet 的cgroup drive保持一致。

```
docker info | grep -i cgroup
# >>> Cgroup Driver: cgroupfs / systemd
```



### 初始化部署Master（Master）

两种常用的方式，我个人更建议第二种生成文件的部署方式。

- **直接初始化init部署**

  ```bash
  $ sudo kubeadm init \
      --kubernetes-version=v1.18.0 \
      --apiserver-advertise-address=192.168.123.181 \
      --pod-network-cidr=10.244.0.0/16 \
      --service-cidr=10.96.0.0/12
      --image-repository=registry.aliyuncs.com/google_containers
  ```

  参数说明**：

  - **–-pod-network-cidr**：集群所属的Pod子网范围，使用fannel网络必须使用这个CIDR。
  - **–-kubernetes-version**：指定K8S版本，建议与kubeadm, kubelet 和 kubectl一致。
  - **–-apiserver-advertise-address**：作为Master与其他Node通信的IP地址。
  - **--control-plane-endpoint**：标志应该被设置成负载均衡器的地址或 DNS 和端口。会取–-apiserver-advertise-address的值
  - **--image-repository**：指定下载源，例：`registry.aliyuncs.com/google_containers`
  - **--service-cidr**：service网段,负载均衡ip
  - **--ignore-preflight-errors=Swap/all**：忽略 swap/所有 报错

  

- **配置文件部署**

  创建配置文件

  ```bash
  $ kubeadm config print init-defaults --kubeconfig ClusterConfiguration > kubeadm.yml
  ```

  根据情况修改配置文件内容

  ```yaml
  apiVersion: kubeadm.k8s.io/v1beta2
  bootstrapTokens:
  - groups:
    - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef
    ttl: 24h0m0s
    usages:
    - signing
    - authentication
  kind: InitConfiguration
  localAPIEndpoint: 
    advertiseAddress: 192.168.123.181 # 作为Master与其他Node通信的IP地址。
    bindPort: 6443
  nodeRegistration:
    criSocket: /var/run/dockershim.sock
    name: k8s-node1
    taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
  --
  apiServer:
    timeoutForControlPlane: 4m0s
  apiVersion: kubeadm.k8s.io/v1beta2
  certificatesDir: /etc/kubernetes/pki
  clusterName: kubernetes
  controllerManager: {}
  dns:
    type: CoreDNS
  etcd:
    local:
      dataDir: /var/lib/etcd
  imageRepository: k8s.gcr.io    # 指定下载源,例：registry.aliyuncs.com/google_containers
  kind: ClusterConfiguration
  kubernetesVersion: v1.18.0     # 指定K8S版本，建议与kubeadm, kubelet 和 kubectl一致。
  networking:
    dnsDomain: cluster.local
    podSubnet: 192.168.0.0/16    # Pod子网默认网段范围
    serviceSubnet: 10.96.0.0/12  # service网段
  scheduler: {}
  ```

  保存kubeadm.yml后，查看及拉取镜像

  ```bash
  $ kubeadm config images list --config kubeadm.yml
  $ kubeadm config images pull --config kubeadm.yml
  ```

  初始化

  ```bash
  $ kubeadm init --config kubeadm-init.yaml
  ```

  

---

若执行**kubeadm init**出错或强制终止，则再需要执行该命令时，需要先执行**kubeadm reset**重置配置后再进行**kubeadm init**。

如果两种方法没有异常，那么你将在最后看到类似这样的返回：

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.6.163:6443 --token 1lfiq6.tyiic0y0gjybw4vq \
    --discovery-token-ca-cert-hash sha256:a407b3d77b8cb9942c06d32a3aa51ed4ed34e4faec3186d9eabd0c4ef307562e
```



#### node加入集群：(Node)

如以上的最后一句，我们可以使用root权限将Worker添加节点进来集群。

```
sudo kubeadm join 192.168.6.163:6443 --token 1lfiq6.tyiic0y0gjybw4vq \
    --discovery-token-ca-cert-hash sha256:a407b3d77b8cb9942c06d32a3aa51ed4ed34e4faec3186d9eabd0c4ef307562e
```

```
W0401 18:20:37.196454   22044 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "k8s-node1" could not be reached
	[WARNING Hostname]: hostname "k8s-node1": lookup k8s-node1 on 8.8.8.8:53: no such host
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## 配置 kubectl

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config(非root才做)

```



添加完毕后，可以在Master上查看节点状态：

```
$ kubectl get pods --all-namespaces
```





## 使用Dashboard

可以使用这句命令一键部署

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml

$ kubectl get pods --all-namespaces | grep dashboard

```



> 可能自己这篇也不免成为一个新坑吧。

### 附：Raspberry Pi 如何使用 HypriotOS

Etcher在写入之后可以编辑一下根目录的user-data如果你有Wifi连接的需求的话,这个步骤将是必须的.

我尝试过很多官网或者 只有在官方Github博客源代码一个关闭的Issue里找到正解

```yaml
#cloud-config
# vim: syntax=yaml
#

# Set your hostname here, the manage_etc_hosts will update the hosts file entries as well
hostname: black-pearl
manage_etc_hosts: true

# You could modify this for your own user information
users:
  - name: pirate                                                # 用户名
    gecos: "Hypriot Pirate"
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    groups: users,docker,video,input
    plain_text_passwd: hypriot                                  # 明文密码
    lock_passwd: false
    ssh_pwauth: true
    chpasswd: { expire: false }

# # Set the locale of the system
locale: "en_US.UTF-8"

# # Set the timezone
# # Value of 'timezone' must exist in /usr/share/zoneinfo
timezone: "Asia/Harbin"                                         # 时区

# # Update apt packages on first boot
# package_update: true
# package_upgrade: true
# package_reboot_if_required: true
package_upgrade: false

# # Install any additional apt packages you need here
# packages:
#  - ntp

# # WiFi connect to HotSpot
# # - use `wpa_passphrase SSID PASSWORD` to encrypt the psk
write_files:
  - content: |
      allow-hotplug wlan0
      iface wlan0 inet dhcp
      wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
      iface default inet dhcp
    path: /etc/network/interfaces.d/wlan0
  - content: |                                                  # 如果启动wifi则按此编辑
      country=CN
      ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
      update_config=1
# 密码(明文加引号)
      network={
      ssid="YourID"
      psk="12345678"                                            
      key_mgmt=WPA-PSK
      }
    path: /etc/wpa_supplicant/wpa_supplicant.conf

# These commands will be ran once on first boot only
runcmd:
  # Pickup the hostname changes
  - 'systemctl restart avahi-daemon'

#  # Activate WiFi interface
# 开启wifi支持
  - 'ifup wlan0'
bootcmd:
# 开机自动连接wifi
  - [ ifup, wlan0 ]

```

未完待续。。。