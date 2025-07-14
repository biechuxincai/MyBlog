---
title: k8s离线部署方案-kubeadm部署
toc: true
description: more
tags:
  - k8s
categories:
  - k8s
date: 2025-06-27 16:22:30
---
# Kubernetes 1.29 零线网环境 kubeadm + Docker 部署步骤

## 部分一：联网环境下的准备操作

### 一、环境设置

#### 1. 主机配置

例：

* Master IP: `192.168.0.10`
* Node1 IP: `192.168.0.11`
* 私有 Docker Registry: `192.168.0.100:5000`

#### 2. 版本信息

* Kubernetes: v1.29.0
* Docker 作为 container runtime（需配合额外组件 cri-dockerd）

### 二、网络环境下操作

#### 1. 使用 yum 下载 kubeadm、kubelet、kubectl 到指定位置

```bash
# 设置 Kubernetes 仓库配置文件
sudo mkdir -p /data/k8s-rpms
sudo cp /etc/yum.repos.d /data/k8s-rpms/ -r

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# 下载 RPM 包（不安装）
sudo yum install --downloadonly --downloaddir=/data/k8s-rpms kubelet kubeadm kubectl --disableexcludes=kubernetes
```

#### 2. 下载 Docker 和 cri-dockerd RPM

```bash
# 安装 yum-utils 工具
yum install -y yum-utils --downloadonly --downloaddir=./docker_rpms

# 配置 docker-ce 源（默认只配置，不安装）
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 下载 docker-ce 和相关依赖
yumdownloader --resolve --destdir=./docker_rpms docker-ce docker-ce-cli containerd.io


# 下载 cri-dockerd
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz


# 之后你会得到一堆 .rpm 包，全部打包
tar -czvf docker_rpms.tar.gz ./docker_rpms

```

#### 3. 参照下面的安装步骤先安装docker和kubeadm

```bash
docker pull registry:2
docker save -o registry.tar registry:2
```
#### 4. 下载 Kubernetes 所需镜像

```bash
kubeadm config images list --kubernetes-version=v1.29.0
```

镜像清单：

```bash
registry.k8s.io/kube-apiserver:v1.29.0
registry.k8s.io/kube-controller-manager:v1.29.0
registry.k8s.io/kube-scheduler:v1.29.0
registry.k8s.io/kube-proxy:v1.29.0
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.9-0
registry.k8s.io/coredns/coredns:v1.11.1
```

#### 5. 下载镜像并推送到私有仓库

生成脚本 `images-download-push.sh`：

```bash
#!/bin/bash
REGISTRY=192.168.0.100:5000
K8S_VERSION=v1.29.0
IMAGES=(
  kube-apiserver:$K8S_VERSION
  kube-controller-manager:$K8S_VERSION
  kube-scheduler:$K8S_VERSION
  kube-proxy:$K8S_VERSION
  pause:3.9
  etcd:3.5.9-0
  coredns/coredns:v1.11.1
)
mkdir -p ./images
for image in ${IMAGES[@]}; do
  docker pull registry.k8s.io/$image
  local_image=$REGISTRY/$(basename $image)
  docker tag registry.k8s.io/$image $local_image
  # docker push $local_image #暂时不需要
  docker save -o ./images/$(basename $image).tar $local_image
done
```
```bash
cd ../images

#!/bin/bash

# 确保脚本以 bash 执行，以支持变量替换

# 遍历 kubeadm config images list 列出的每个镜像
for img in $(kubeadm config images list); do
    # 提取镜像名称，用于文件名
    # 例如：registry.k8s.io/kube-apiserver:v1.29.0 -> kube-apiserver_v1.29.0
    # 注意：这里我们直接使用完整的 img 名称作为 tar 文件名的一部分，更准确
    # 或者如果你想更简洁，可以进一步处理字符串
    filename=$(echo "$img" | sed 's/\//_/g' | sed 's/:/_/g') # 将斜杠和冒号替换为下划线，适合文件名

    # 首先尝试拉取镜像（如果本地没有，或者版本不对）
    echo "拉取镜像: $img"
    docker pull "$img"

    # 保存镜像到本地文件
    echo "保存镜像到: ./$filename.tar"
    docker save -o "./$filename.tar" "$img"

    # 如果你需要将这些镜像推送到私有仓库，再添加这部分逻辑
    # 例如：
    # private_repo="192.168.0.10:5000"
    # new_img_name="$private_repo/$(basename "$img")" # 提取原始镜像名，例如 kube-apiserver:v1.29.0
    # docker tag "$img" "$new_img_name"
    # echo "推送镜像到私有仓库: $new_img_name"
    # docker push "$new_img_name"
    # docker rmi "$new_img_name" # 可选：推送后删除本地标签
done

echo "所有镜像处理完成。"
```

#### 6. 下载 Flannel 网络插件

```bash

cd ../manifests
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
sed -i 's|quay.io/coreos/flannel|192.168.0.10:5000/flannel|g'

docker pull flannel/flannel:v0.24.2
docker tag flannel/flannel:v0.24.2 192.168.10.11:5000/flannel:v0.24.2
docker save -o flannel.tar 192.168.10.11:5000/flannel:v0.24.2

```

#### 7. 打包离线安装包

```
k8s-offline-package/
├── rpms/
│   ├── kubeadm-*.rpm
│   ├── kubelet-*.rpm
│   ├── kubectl-*.rpm
│   ├── docker*.rpm
│   └── cri-dockerd*.rpm
├── images/
│   ├── kube-apiserver.tar
│   ├── ...
├── kubeadm-config.yaml
├── kube-flannel.yml
├── install-guide.txt
```

---

## 部分二：离线环境下的部署步骤

### 一、基础环境配置

#### 1. 关闭 swap/selinux/firewalld

```bash
sudo swapoff -a && sudo sed -i '/swap/d' /etc/fstab
sudo setenforce 0 && sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
sudo systemctl disable firewalld --now
```

#### 2. 添加内核参数

```bash
#将桥接的ipv4流量传递到iptables的链
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo modprobe br_netfilter
sudo sysctl --system
```

#### 3. 设置主机名

```bash
cat >> /etc/hosts <<EOF
192.168.0.10 k8s-master
192.168.0.11 k8s-node1
192.168.0.12 k8s-node2
EOF

# 分别在每台上：
hostnamectl set-hostname k8s-master  # 或 k8s-node1、k8s-node2
```

### 二、安装 Docker 和 cri-dockerd

```bash
cd rpms && sudo rpm -ivh docker*.rpm
sudo systemctl enable docker --now

# 配置 /etc/docker/daemon.json 信任私有仓库
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries": ["192.168.0.100:5000"]
}
EOF
sudo systemctl restart docker
```

#### 创建私有 Docker Registry

```bash
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /opt/registry:/var/lib/registry registry:2
```

 配置各节点信任 Registry
在所有节点 /etc/docker/daemon.json：

```bash
{ "insecure-registries": ["192.168.0.10:5000"] }
```

然后重启：

```bash
systemctl daemon-reexec
systemctl restart docker
```

安装cri-dockerd

```bash
sudo rpm -ivh cri-dockerd*.rpm

sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable cri-docker --now
```

方式二
使用二进制压缩包安装，将cri-docker 放到/usr/bin/下，配置cri-docker.service和cri-docker.socket
先启动socket, 
```bash
/usr/bin/cri-dockerd --pod-infra-container-image=10.x.x.x:5000/pause:3.9 --container-runtime-endpoint fd://
```

注意：指定私有仓库的时候要在配置文件中指定， cri-dockerd 中 pause 镜像版本硬编码

```go
const (
    defaultPodSandboxImageName    = "registry.k8s.io/pause"
    defaultPodSandboxImageVersion = "3.10"
)
```

<u>https://github.com/Mirantis/cri-dockerd/issues/97
https://docs.k0sproject.io/v1.32.1+k0s.0/runtime/</u>

后面使用kubelet 遇到拉取默认pause，则创建/etc/cri-dockerd/config.toml

```bash
sandbox_image = "10.x.x.x:5000/pause:3.9"
```

### 三、安装 kubeadm/kubelet/kubectl

```bash
cd rpms && sudo rpm -ivh kubelet-*.rpm kubeadm-*.rpm kubectl-*.rpm
sudo systemctl enable kubelet
```

### 四、导入镜像

```bash
cd images
for img in *.tar; do sudo docker load -i $img; done
```

### 五、初始化 Master

#### 1. 配置 kubeadm 模板

`kubeadm-config.yaml`：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.0.10
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock # Configure connection to cri-dockerd
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0
imageRepository: 192.168.0.100:5000
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: iptables
```

#### 2. 初始化

```bash
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

#### 3. 配置 kubectl 用户

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 六、部署 Flannel 网络

```bash
kubectl apply -f kube-flannel.yml
```

### 七、Node 节点加入

在 Node 节点执行 Master 初始化时输出的 join 指令：

```bash
sudo kubeadm join 192.168.0.10:6443 --token xxx \
  --discovery-token-ca-cert-hash sha256:xxxx \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

确保每个节点都导入了需要镜像，且 docker 配置了信任私有仓库，同时已安装并启动 cri-dockerd

### 八、校验状态

```bash
kubectl get nodes
kubectl get pods -A
```

---

## 注意事项

* Kubernetes 1.24+ 开始弃用 Dockershim，1.29 如继续使用 Docker 需安装 cri-dockerd。

* 私有 registry 必须是可访问的 IP + 端口。

* Docker 必须配置 `insecure-registries`。

* 镜像名称必须和 kubeadm 配置一致。

* 如果不想使用私有仓库，需把镜像 tag 成 registry.k8s.io/格式名称。

* 如使用非 root 用户安装：
  
  * 必须具有 `sudo` 权限，确保能安装软件包、配置 systemd 和访问 `/etc` 等目录。
  * `.kube/config` 拷贝后需执行 `chown`，确保普通用户能使用 kubectl。
  * 建议用 `sudo tee` 和 `sudo systemctl` 替代直接写入配置或启用服务命令。

---

