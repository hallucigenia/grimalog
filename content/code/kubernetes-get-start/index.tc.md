---
title: "kubeadm搭建部署kubernetes集群"
date: 2020-03-22T01:04:10+08:00
draft: false

categories: ['Code']
tags: ['Kubernetes', 'Docker']
author: "Fancy"
resizeImages: false

---

<!--more-->

> 零零散散看了不少k8s的文檔，搭建過幾次，每次都有不同的收獲，部署根據不同的應用場景比較靈活，對應的姿勢也是五花八門，網上關於K8s部署的文檔坑實在是太多了，想著能否好好總結下，學習下部署設計，便於自己查閱以及更好的理解k8s。

目前kubeadm支持：

- Ubuntu 16.04+
- Debian 9+
- CentOS 7
- Red Hat Enterprise Linux (RHEL) 7
- Fedora 25+
- HypriotOS v1.0.1 +
- Container Linux (tested with 1800.6.0)

官方建議至少2G內存，要求雙核及以上。並檢查網絡鏈接以及用的端口是否開放。更改下防火墻設置放行端口

我這裏利用了`斐訊N1`刷的`Armbian`和`Raspberry Pi 3B+`的`HypriotOS`組成最小單元的集成作為示例，之所以選擇這兩個，主要是因為硬件便宜（家裏已經4個N1了），另外想試試HypriotOS這麽好用的Docker專用系統，同是debian系的也便於安裝命令復用。剛好作為最小集群壹個Master節點壹個Node節點。

比如Item2 就使用⌘(command) + d 左右分屏，⌘(command) + ⇧(shift) + i 開啟多窗口輸入命令。省很多功夫。

首先按要求關閉Swap（性能考慮，不關閉會拖慢速度，官方說的是必須的）

```
sudo swapoff -a
```

關閉Selinux：

```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```



確保主機名（hostname）和MAC， product_uuid是否唯壹的，這些是用於K8s辨別Node節點唯壹性的值，都相同會導致安裝失敗，可以分別用以下命令查看
畢竟是ARM，我們的armbian內核是沒有加載 dmi sysfs的，所以product_uuid無從談起。

```
cat /etc/hostname 
sudo ifconfig -a
sudo cat /sys/class/dmi/id/product_uuid
```

我的建議是首先更改`Hostname`，後期便利很多。

```
# Master節點
sudo hostnamectl --static set-hostname k8s-master
# Node節點1
sudo hostnamectl --static set-hostname k8s-node1
```



### 修改橋接網絡配置

 `lsmod | grep br_netfilter`. 檢查br_netfilter是否運行，如果沒有的話可以用 `modprobe br_netfilter`顯式調用。然後運行防止iptables被繞過：

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

###  安裝Docker

截至2020年3月30號，官方推薦的Docker版本是 `19.03.8`。我們通過`docker -v`確保版本是在能穩定運行k8s的`1.13.1, 17.03, 17.06, 17.09, 18.06, 18.09, 19.03.08 `之中。

安裝Docker CE老生常談了，Azure阿裏雲這些鏡像加速畢竟是有N1的程序員我表示根本不需要，網上也有很多參考，這裏可以使用壹鍵腳本

```
curl -fsSL "https://get.docker.com/" | sh
systemctl enable --now docker
```

或者Docker官網https://docs.docker.com/install/linux/docker-ce/debian/根據不同發行版和架構安裝，我貼的這份僅限Arm架構的Debian安裝。

```bash
# 允許apt透過HTTPS使用軟件包
apt update && apt install -y \
  apt-transport-https ca-certificates curl software-properties-common gnupg2

# 添加Docker的GPGkey
curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -

# 添加Docker apt軟件包
add-apt-repository \
   "deb [arch=arm64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"

# 安裝 Docker CE 及 Container.
apt-get update && apt-get install -y \
  containerd.io=1.2.13-1 \
  docker-ce=5:19.03.8~3-0~debian-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.8~3-0~debian-$(lsb_release -cs)
# sudo apt-get install docker-ce docker-ce-cli containerd.io

```

###配置鏡像

修改`daemon.json`，配置cgroup驅動，與k8s壹致，如果添加鏡像加速的話添加多句` "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"],`之類，之後重啟，如下：

```bash
# 這裏使用systemd作為Docker croup
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

# 創建Docker服務目錄

mkdir -p /etc/systemd/system/docker.service.d

# 重啟Docker，開啟開機啟動
systemctl daemon-reload
systemctl enable docker
systemctl restart docker

# 檢查Docker
docker -v
# >>> Docker version 19.03.8, build afacb8b
docker run hello-world
```



### 安裝Runtime

### kubeadm的方案

### 安裝kubeadm, kubelet 和 kubectl（Master & Node）

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

檢查docker的cgroup驅動，應與kubelet 的cgroup drive保持壹致。

```
docker info | grep -i cgroup
# >>> Cgroup Driver: cgroupfs / systemd
```



### 初始化部署Master（Master）

如果熟悉之後，有更多的靈活性可以按需調整，我個人更建議第二種生成文件的部署方式。

- **直接初始化init部署**

  ```bash
  $ sudo kubeadm init \
      --kubernetes-version=v1.18.0 \
      --apiserver-advertise-address=192.168.123.181 \
      --pod-network-cidr=10.244.0.0/16 \
      --service-cidr=10.96.0.0/12
      --image-repository=registry.aliyuncs.com/google_containers
  ```

  參數說明：

  - **–-pod-network-cidr**：集群所屬的Pod子網範圍，使用fannel網絡必須使用這個CIDR。
  - **–-kubernetes-version**：指定K8S版本，建議與kubeadm, kubelet 和 kubectl壹致。
  - **–-apiserver-advertise-address**：作為Master與其他Node通信的IP地址。
  - **--control-plane-endpoint**：標誌應該被設置成負載均衡器的地址或 DNS 和端口。會取–-apiserver-advertise-address的值
  - **--image-repository**：指定下載源，例：`registry.aliyuncs.com/google_containers`
  - **--service-cidr**：service網段,負載均衡ip
  - **--ignore-preflight-errors=Swap/all**：忽略 swap/所有 報錯

  

- **配置文件部署**

  創建配置文件

  ```bash
  $ kubeadm config print init-defaults --kubeconfig ClusterConfiguration > kubeadm.yml
  ```

  根據情況修改配置文件內容，kube-proxy 的 IPVS 模式在大規模下會有更好的性能表現，我們

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
    advertiseAddress: 192.168.123.181 # 作為Master與其他Node通信的IP地址。
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
  imageRepository: k8s.gcr.io    # 指定下載源,例：registry.aliyuncs.com/google_containers
  kind: ClusterConfiguration
  kubernetesVersion: v1.18.0     # 指定K8S版本，建議與kubeadm, kubelet 和 kubectl壹致。
  networking:
    dnsDomain: cluster.local
    podSubnet: 192.168.0.0/16    # Pod子網默認網段範圍
    serviceSubnet: 10.96.0.0/12  # service網段
  scheduler: {}
  ```

  

  保存kubeadm.yml後，查看及拉取鏡像

  ```bash
  $ kubeadm config images list --config kubeadm.yml
  $ kubeadm config images pull --config kubeadm.yml
  ```

  初始化

  ```bash
  $ kubeadm init --config kubeadm-init.yaml
  ```

  

---

若執行**kubeadm init**出錯或強制終止，則再需要執行該命令時，需要先執行**kubeadm reset**重置配置後再進行**kubeadm init**。

如果兩種方法沒有異常，那麽妳將在最後看到類似這樣的返回：

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

kubeadm join 192.168.123.181:6443 --token 1lfiq6.tyiic0y0gjybw4vq \
    --discovery-token-ca-cert-hash sha256:a407b3d77b8cb9942c06d32a3aa51ed4ed34e4faec3186d9eabd0c4ef307562e
```

最後壹句務必要先記錄下來。



## 配置 kubectl 環境（Master）

如果是root賬戶不必最後壹句chown賦予權限操作

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



## 使用Dashboard（Master）

如果使用Dashboard，應在配置 kubectl 環境後，Node Join前部署，否則：

```bash
W0401 18:20:37.196454   22044 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
```

可以使用這句命令壹鍵部署

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml

$ kubectl get pods --all-namespaces | grep dashboard

```



#### node加入集群：(Node)

如以上的最後壹句，我們可以使用root權限將Worker添加節點進來集群。

```bash
$ sudo kubeadm join 192.168.123.181:6443 --token 1lfiq6.tyiic0y0gjybw4vq \
    --discovery-token-ca-cert-hash sha256:a407b3d77b8cb9942c06d32a3aa51ed4ed34e4faec3186d9eabd0c4ef307562e
```

```
[preflight] Running pre-flight checks
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



添加完畢後，可以在Master上查看節點狀態：

```
$ kubectl get pods --all-namespaces
```



> k8s部署下來，理解也不是很深入，如果有問題歡迎共同討論。可能自己這篇也不免成為壹個新坑吧。另外樹莓派3B+做master節點不太合適，這裏用的N1作為主節點，往後計劃試試k3s部署，相較之下更加適合Arm架構以及樹莓派這種低效能的服務器集群吧。這裏先用HypriotOS做個過渡。



### 附：Raspberry Pi 如何使用 HypriotOS

Etcher在寫入之後，可以編輯壹下根目錄的user-data

如果妳有Wifi連接的需求的話,這個步驟將是必須的。

嘗試過很多官網或者帖子上的內容，官方Github博客源代碼壹個關閉的Issue裏可以找到很好的解法

```yaml
#cloud-config
# vim: syntax=yaml
#

# Set your hostname here, the manage_etc_hosts will update the hosts file entries as well
hostname: black-pearl
manage_etc_hosts: true

# You could modify this for your own user information
users:
  - name: pirate                                                # 用戶名
    gecos: "Hypriot Pirate"
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    groups: users,docker,video,input
    plain_text_passwd: hypriot                                  # 明文密碼
    lock_passwd: false
    ssh_pwauth: true
    chpasswd: { expire: false }

# # Set the locale of the system
locale: "en_US.UTF-8"

# # Set the timezone
# # Value of 'timezone' must exist in /usr/share/zoneinfo
timezone: "Asia/Harbin"                                         # 時區

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
  - content: |                                                  # 如果啟動wifi則按此編輯
      country=CN
      ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
      update_config=1
# 密碼(明文加引號)
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
# 開啟wifi支持
  - 'ifup wlan0'
bootcmd:
# 開機自動連接wifi
  - [ ifup, wlan0 ]

```