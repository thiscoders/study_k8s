# Kubeadm 安装 k8s 步骤

*使用centos7系统*

## 1. 准备步骤
1. 关闭防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
```
2. 关闭swap
```bash
# 删除fstab关于swap的配置
/dev/mapper/centos-swap swap swap defaults 0 0
# 修改文件
echo vm.swappiness=0 >> /etc/sysctl.conf
# 重启
reboot
```

## 2. 安装k8s
```bash
# 修改主机名
hostnamectl set-hostname newhostname
# 更新系统
yum update -y
# 安装第三方库
yum install epel-release -y
# 安装工具软件
yum install htop vim tree unzip wget curl axel -y
# 停止防火墙
systemctl stop firewalld
# 关闭防火墙开机自启
systemctl disable firewalld
# 下载阿里云的docker-ce仓库
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -P /etc/yum.repos.d
# 列出docker版本
yum list docker-ce --showduplicates | sort -r
# 安装docker
yum install docker-ce -y
# 非root用户执行docker命令不输入sudo
sudo gpasswd -a ${USER} docker # ${USER} 
# 开启docker服务
systemctl start docker
# docker服务开机自启
systemctl enable docker
# 导入k8s的yum源
echo '[kubernetes]
name=Kubernetes Repo
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
enabled=1' > /etc/yum.repos.d/kubernetes.repo
# 下载证书
wget https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
wget https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
# 导入证书
rpm --import yum-key.gpg
rpm --import rpm-package-key.gpg
# 安装k8s的三大组件
yum install kubelet-1.11.1-0  kubectl-1.11.1-0 kubeadm-1.11.1-0 -y
# 便于docker创建iptables规则
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
echo 'net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1' >> /etc/sysctl.conf 
# k8s开机自启
systemctl enable kubelet
systemctl start kubelet

# 部署flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 3. 启动集群

1. 预先拉取镜像
```bash
echo "begin pull k8s images" \
&& docker pull mirrorgooglecontainers/kube-apiserver-amd64:v1.11.1 \
&& docker pull mirrorgooglecontainers/kube-controller-manager-amd64:v1.11.1 \
&& docker pull mirrorgooglecontainers/kube-scheduler-amd64:v1.11.1 \
&& docker pull mirrorgooglecontainers/kube-proxy-amd64:v1.11.1 \
&& docker pull mirrorgooglecontainers/pause:3.1 \
&& docker pull mirrorgooglecontainers/etcd-amd64:3.2.18 \
&& docker pull coredns/coredns:1.1.3 \
&& echo "images pull complete!" \
&& docker tag docker.io/mirrorgooglecontainers/kube-proxy-amd64:v1.11.1 k8s.gcr.io/kube-proxy-amd64:v1.11.1 \
&& docker tag docker.io/mirrorgooglecontainers/kube-scheduler-amd64:v1.11.1 k8s.gcr.io/kube-scheduler-amd64:v1.11.1 \
&& docker tag docker.io/mirrorgooglecontainers/kube-apiserver-amd64:v1.11.1 k8s.gcr.io/kube-apiserver-amd64:v1.11.1 \
&& docker tag docker.io/mirrorgooglecontainers/kube-controller-manager-amd64:v1.11.1 k8s.gcr.io/kube-controller-manager-amd64:v1.11.1 \
&& docker tag docker.io/mirrorgooglecontainers/etcd-amd64:3.2.18  k8s.gcr.io/etcd-amd64:3.2.18 \
&& docker tag docker.io/mirrorgooglecontainers/pause:3.1  k8s.gcr.io/pause:3.1 \
&& docker tag coredns/coredns:1.1.3  k8s.gcr.io/coredns:1.1.3 \
&& echo "images rename complete!" \
&& docker rmi mirrorgooglecontainers/kube-apiserver-amd64:v1.11.1 \
&& docker rmi mirrorgooglecontainers/kube-controller-manager-amd64:v1.11.1 \
&& docker rmi mirrorgooglecontainers/kube-scheduler-amd64:v1.11.1 \
&& docker rmi mirrorgooglecontainers/kube-proxy-amd64:v1.11.1 \
&& docker rmi mirrorgooglecontainers/pause:3.1 \
&& docker rmi mirrorgooglecontainers/etcd-amd64:3.2.18 \
&& docker rmi coredns/coredns:1.1.3 \
&& echo "images remove complete!" \
&& echo "k8s images pull complete!"
```

2. 启动集群
```bash
# 启动集群
kubeadm init --kubernetes-version=v1.11.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12
# 新建普通用户并添加配置文件
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```