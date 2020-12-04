



# K8S学习

### 前戏

#### 相比传统企业应用，云原生系统有哪些优势呢？

​	传统企业应用很多都是单体应用，每个系统都非常庞大，现在呢很多企业都在做微服务改造，希望把应用拆的散一点，当然拆的力度也是需要控制的，不可能把一个应用拆的非常散，拆分之后有很多好处，每个微服务都可以单独去开发，可以独立的去做升级，不用说是更新一个组件把整个服务都去做更新，然后呢微服务呢就需要使用容器这些技术。



### Docker

https://www.bilibili.com/video/BV1kt411H7p1



![image-20200520230613885](/Users/shaowei/Library/Application Support/typora-user-images/image-20200520230613885.png)



![image-20200520230645228](/Users/shaowei/Library/Application Support/typora-user-images/image-20200520230645228.png)









## 一. K8S介绍

![image-20200425162214818](/Users/shaowei/Desktop/image-20200425162214818.png)



## 二. 安装部署

### 2.1 服务器初始化

配置ip网络

```Bash
# master01
hostnamectl set-hostname k8s-master01
vim /etc/sysconfig/network-scripts/ifcfg-ens33
  BOOTPROTO="static"
  IPADDR=192.168.0.11
  NETMASK=255.255.255.0
  GATEWAY=192.168.0.1
vim /etc/hosts
  192.168.0.11 k8s-master01
  192.168.0.22 k8s-master02
  192.168.0.33 k8s-node01
  192.168.0.44 k8s-node02
scp /etc/hosts root@k8s-master02:/etc/hosts
scp /etc/hosts root@k8s-node01:/etc/hosts
scp /etc/hosts root@k8s-node02:/etc/hosts
scp /etc/hosts root@k8s-node03:/etc/hosts
 
# master02
hostnamectl set-hostname k8s-master02
vim /etc/sysconfig/network-scripts/ifcfg-ens33
  BOOTPROTO="static"
  IPADDR=192.168.0.22
  NETMASK=255.255.255.0
  GATEWAY=192.168.0.1

# node01
hostnamectl set-hostname k8s-node01
vim /etc/sysconfig/network-scripts/ifcfg-ens33
  BOOTPROTO="static"
  IPADDR=192.168.0.33
  NETMASK=255.255.255.0
  GATEWAY=192.168.0.1
  
# node02
hostnamectl set-hostname k8s-node02
vim /etc/sysconfig/network-scripts/ifcfg-ens33
  BOOTPROTO="static"
  IPADDR=192.168.0.44
  NETMASK=255.255.255.0
  GATEWAY=192.168.0.1
  
# node03
hostnamectl set-hostname k8s-node03
vim /etc/sysconfig/network-scripts/ifcfg-ens33
  BOOTPROTO="static"
  IPADDR=192.168.0.44
  NETMASK=255.255.255.0
  GATEWAY=192.168.0.1
```

配置镜像

```BASH
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

k8s-master01上设置免密:

```shell
yum install -y expect

#分发公钥
ssh-keygen -t rsa -P "" -f /root/.ssh/id_rsa

for i in k8s-master01 k8s-master02 k8s-node01 k8s-node02;do
expect -c "
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@$i
        expect {
                \"*yes/no*\" {send \"yes\r\"; exp_continue}
                \"*password*\" {send \"123456\r\"; exp_continue}
                \"*Password*\" {send \"123456\r\";}
        } "
done
```

安装依赖包

```bash
yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat ibseccomp wget vim net-tools git
```

关闭swap

```bash
# 将swap一行注释
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

关闭防火墙和selinux

```bash
systemctl stop firewalld && systemctl disable firewalld
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/sysconfig/selinux && getenforce
```

关闭防火墙

```bash
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
iptables -P FORWARD ACCEPT
```

升级内核

```bash
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

# 安装完后检查 /boot/grub2/grub.cfg 中对应内核 mebuentry 中是否包含 initrd16 配置，如果没有再安装一次
yum --enablerepo=elrepo-kernel install -y kernel-lt

# 设置开机从新内核启动
grub2-set-default "CentOS Linux (4.4.182-1.elrepo.x86_64) 7 (Core)"

# 重启后再次查看内核版本

init 6
uname -r
```

开启ipvs

```bash
modprobe br_netfilter

cat > /etc/sysconfig/modules/ipvs.modules << EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash
/etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

调整内核参数 对于k8s

```bash
cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
fs.inotify.max_user_instances-8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

cp kubernetes.conf /etc/sysctl.d/kubernetes.conf
sysctl -p /etc/sysctl.d/kubernetes.conf
```

设置时区

```bash
timedatectl set-timezone Asia/Shanghai     # 设置系统时区为 中国/上海
timedatectl set-local-rtc 0								 # 将当前的 UTC 时间写入硬盘时钟

#重启依赖于系统时间的服务
systemctl restart rsyslog
systemctl restart crond
```

关闭服务

```bash
systemctl stop postfix && systemctl disable postfix && systemctl status postfix
```

配置环境变量

```bash
echo 'PATH=$PATH:$HOME/bin:/opt/kubernetes/bin' >>/etc/profile
source  /etc/profile
env|grep PATH
```

### 2.2 集群安装

#### 2.2.1 二进制安装 

 [安装地址](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.15.md#downloads-for-v1151)

##### 下载软件包

```bash
mkdir -p /opt/kubernetes/{cfg,bin,ssl,log} && cd /usr/local/src
wget https://dl.k8s.io/v1.15.11/kubernetes-server-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.15.11/kubernetes-node-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.15.10/kubernetes-client-linux-amd64.tar.gz
```

##### 安装证书

下载cfssl命令 (k8s-master01 和 k8s-node01)

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/bin/cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/bin/cfssl-json
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/bin/cfssl-cerrinfo
chmod +x /usr/bin/cfssl*
which cfssl cfssl-json cfssl-certinfo
```

证书配置

```bash
mkdir /usr/local/src/ssl && cd /usr/local/src/ssl
vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "8760h"
      }
    }
  }
}

vim ca-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

cfssl gencert -initca ca-csr.json | cfssl-json -bare ca
```

分发证书

```
cp ca.csr ca.pem ca-key.pem ca-config.json /opt/kubernetes/ssl
SCP证书到k8s-node1和k8s-node2节点
scp ca.csr ca.pem ca-key.pem ca-config.json k8s-master02:/opt/kubernetes/ssl 
scp ca.csr ca.pem ca-key.pem ca-config.json k8s-node01:/opt/kubernetes/ssl
scp ca.csr ca.pem ca-key.pem ca-config.json k8s-node02:/opt/kubernetes/ssl
scp ca.csr ca.pem ca-key.pem ca-config.json k8s-node03:/opt/kubernetes/ssl
```

##### etcd安装

[安装](https://github.com/etcd-io/etcd/releases/download/v3.2.30/etcd-v3.2.30-linux-arm64.tar.gz)

```bash
# k8s-node01 上操作, 将 etcd etcdctl 命令传到其它两个节点

cd /usr/local/src
wget https://github.com/etcd-io/etcd/releases/download/v3.2.30/etcd-v3.2.30-linux-arm64.tar.gz
tar zxf etcd-v3.2.30-linux-arm64.tar.gz
cd etcd-v3.2.30-linux-amd64
cp etcd etcdctl /opt/kubernetes/bin/
scp etcd etcdctl k8s-node02:/opt/kubernetes/bin/
scp etcd etcdctl k8s-node03:/opt/kubernetes/bin/
```

etcd证书配置

```bash
mkdir /usr/local/src/ssl && cd /usr/local/src/ssl
vim etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
	"192.168.0.33",
	"192.168.0.44",
	"192.168.0.55"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
  -ca-key=/opt/kubernetes/ssl/ca-key.pem \
  -config=/opt/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssl-json -bare etcd

cp etcd*.pem /opt/kubernetes/ssl
scp etcd*.pem 192.168.0.44:/opt/kubernetes/ssl
scp etcd*.pem 192.168.0.55:/opt/kubernetes/ssl
```

etcd 配置文件

```bash
vim /opt/kubernetes/cfg/etcd.conf
#[member]
ETCD_NAME="etcd-node01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_SNAPSHOT_COUNTER="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://192.168.0.33:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.0.33:2379,https://127.0.0.1:2379"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.0.33:2380"
# if you use different ETCD_NAME (e.g. test),
# set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd-node01=https://192.168.0.33:2380,etcd-node02=https://192.168.0.44:2380,etcd-node03=https://192.168.0.55:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.0.33:2379"
#[security]
CLIENT_CERT_AUTH="true"
ETCD_CA_FILE="/opt/kubernetes/ssl/ca.pem"
ETCD_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
ETCD_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_CA_FILE="/opt/kubernetes/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
```

etcd 服务

```bash
vim /etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/lib/etcd
EnvironmentFile=-/opt/kubernetes/cfg/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /opt/kubernetes/bin/etcd"
Type=notify

[Install]
WantedBy=multi-user.target
```

同步etcd配置文件

```bash
scp /opt/kubernetes/cfg/etcd.conf 192.168.0.44:/opt/kubernetes/cfg/
scp /etc/systemd/system/etcd.service 192.168.0.44:/etc/systemd/system/

scp /opt/kubernetes/cfg/etcd.conf 192.168.0.55:/opt/kubernetes/cfg/
scp /etc/systemd/system/etcd.service 192.168.0.55:/etc/systemd/system/

```

修改k8s-node02 和k8s-node03 的配置文件

```bash
vim /opt/kubernetes/cfg/etcd.conf
ETCD_NAME="etcd-node02"
ETCD_LISTEN_PEER_URLS="https://192.168.0.44:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.0.44:2379,https://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.0.44:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.0.44:2379"

vim /opt/kubernetes/cfg/etcd.conf
ETCD_NAME="etcd-node03"
ETCD_LISTEN_PEER_URLS="https://192.168.0.55:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.0.55:2379,https://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.0.55:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.0.55:2379"

```

所有节点启动服务

```bash
mkdir /var/lib/etcd
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd

# 验证集群状态，发现3个都是健康的
etcdctl --endpoints=https://192.168.0.33:2379 --ca-file=/opt/kubernetes/ssl/ca.pem --cert-file=/opt/kubernetes/ssl/etcd.pem --key-file=/opt/kubernetes/ssl/etcd-key.pem cluster-health
  member a87d6aefcab03dba is healthy: got healthy result from https://192.168.0.33:2379
  member de871cd73a363bbe is healthy: got healthy result from https://192.168.0.44:2379
  member f3acc78943e7fac8 is healthy: got healthy result from https://192.168.0.55:2379
  cluster is healthy
```

##### master安装

```bash
cd /usr/local/src/kubernetes
cp server/bin/kube-apiserver /opt/kubernetes/bin/
cp server/bin/kube-controller-manager /opt/kubernetes/bin/
cp server/bin/kube-scheduler /opt/kubernetes/bin/
```



```bash
cd /usr/local/src/ssl
vim kubernetes-csr.json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.0.11",
    "10.1.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes kubernetes-csr.json | cfssl-json -bare kubernetes

cp kubernetes*.pem /opt/kubernetes/ssl/
scp kubernetes*.pem 192.168.0.22:/opt/kubernetes/ssl/
scp kubernetes*.pem 192.168.0.33:/opt/kubernetes/ssl/
scp kubernetes*.pem 192.168.0.44:/opt/kubernetes/ssl/
scp kubernetes*.pem 192.168.0.55:/opt/kubernetes/ssl/
```

创建 kube-apiserver 使用的客户端 token 文件

```bash
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
441e3efb9bd7a27e3c5ba638df77c0d0 
vim /opt/kubernetes/ssl/bootstrap-token.csv
441e3efb9bd7a27e3c5ba638df77c0d0,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

创建基础用户名/密码认证配置

```bash
vim /opt/kubernetes/ssl/basic-auth.csv
admin,admin,1
readonly,readonly,2
```

apiserver配置文件

```bash
vim /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kubernetes/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --bind-address=192.168.0.11 \
  --insecure-bind-address=127.0.0.1 \
  --authorization-mode=Node,RBAC \
  --runtime-config=rbac.authorization.k8s.io/v1 \
  --kubelet-https=true \
  --anonymous-auth=false \
  --basic-auth-file=/opt/kubernetes/ssl/basic-auth.csv \
  --enable-bootstrap-token-auth \
  --token-auth-file=/opt/kubernetes/ssl/bootstrap-token.csv \
  --service-cluster-ip-range=10.1.0.0/16 \
  --service-node-port-range=20000-40000 \
  --tls-cert-file=/opt/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/opt/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/opt/kubernetes/ssl/ca.pem \
  --etcd-certfile=/opt/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/opt/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://192.168.0.33:2379,https://192.168.0.44:2379,https://192.168.0.55:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/opt/kubernetes/log/api-audit.log \
  --event-ttl=1h \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

启动

```bash
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

安装controller

```bash
vim /usr/lib/systemd/system/kube-controller-manager.service

[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.1.0.0/16 \
  --cluster-cidr=10.2.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启动

```bash
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

安装scheduler

```bash
vim /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启动

```bash
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

##### kubectl安装

```bash
cd /usr/local/src/ && tar xf kubernetes-client-linux-amd64.tar.gz
cd /usr/local/src/kubernetes/client/bin
cp kubectl /opt/kubernetes/bin/
```

安装证书

```bash
cd /usr/local/src/ssl/
vim admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

# 生成admin证书和私钥


cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes admin-csr.json | cfssl-json -bare admin
  
# 设置集群参数

kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.0.11:6443
   
# 设置客户端认证参数

kubectl config set-credentials admin \
   --client-certificate=/opt/kubernetes/ssl/admin.pem \
   --embed-certs=true \
   --client-key=/opt/kubernetes/ssl/admin-key.pem
   
# 设置上下文参数

kubectl config set-context kubernetes \
   --cluster=kubernetes \
   --user=admin
kubectl config use-context kubernetes

cat ~/.kube/config

# 查看健康状态

kubectl get cs
  NAME                 STATUS    MESSAGE             ERROR
  controller-manager   Healthy   ok                  
  scheduler            Healthy   ok                  
  etcd-1               Healthy   {"health":"true"}   
  etcd-2               Healthy   {"health":"true"}   
  etcd-0               Healthy   {"health":"true"} 
```

##### kubelet安装

创建角色绑定

```bash
#  k8s-master 上操作

kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

创建 kubelet bootstrapping kubeconfig 文件 设置集群参数        

```bash
#  k8s-master 上操作

cd /usr/local/src/ssl/
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.0.11:6443 \
   --kubeconfig=bootstrap.kubeconfig
```

设置客户端认证参数

```bash
#  k8s-master 上操作

kubectl config set-credentials kubelet-bootstrap --token=441e3efb9bd7a27e3c5ba638df77c0d0 --kubeconfig=bootstrap.kubeconfig
```

设置上下文参数

```bash
#  k8s-master 上操作

kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
```

选择默认上下文

```bash
#  k8s-master 上操作

kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

cp bootstrap.kubeconfig /opt/kubernetes/cfg
scp bootstrap.kubeconfig 192.168.0.33:/opt/kubernetes/cfg
scp bootstrap.kubeconfig 192.168.0.44:/opt/kubernetes/cfg
scp bootstrap.kubeconfig 192.168.0.55:/opt/kubernetes/cfg
```

node节点安装 kubelet kube-proxy

```bash
# K8s-node01 操作

cd /usr/local/src
tar -zxf kubernetes-node-linux-amd64.tar.gz
cd kubernetes/node/bin/
cp kubelet kube-proxy /opt/kubernetes/bin/
scp kubelet kube-proxy root@k8s-node02:/opt/kubernetes/bin/
scp kubelet kube-proxy root@k8s-node03:/opt/kubernetes/bin/
```

设置CNI支持

```bash
# K8s-node01 k8s-node02 k8s-node03

mkdir -p /etc/cni/net.d
vim /etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}
```

创建kubelet数据

```bash
# K8s-node01 k8s-node02 k8s-node03
# 之后数据放在此目录下
mkdir /var/lib/kubelet
```



```bash
# K8s-node01

vim /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \
  --address=192.168.0.33 \
  --hostname-override=192.168.0.33 \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
  --cert-dir=/opt/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/kubernetes/bin/cni \
  --cluster-dns=10.1.0.2 \
  --cluster-domain=cluster.local. \
  --hairpin-mode hairpin-veth \
  --fail-swap-on=false \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5

## 拷贝并修改相应ip
scp /usr/lib/systemd/system/kubelet.service 192.168.0.44:/usr/lib/systemd/system/kubelet.service
scp /usr/lib/systemd/system/kubelet.service 192.168.0.55:/usr/lib/systemd/system/kubelet.service

# k8s-node02 上修改配置文件

vim /usr/lib/systemd/system/kubelet.service
  --address=192.168.0.44
  --hostname-override=192.168.0.44
  
# k8s-node03 上修改配置文件

vim /usr/lib/systemd/system/kubelet.service
  --address=192.168.0.55
  --hostname-override=192.168.0.55
```

启动服务

```bash
# K8s-node01 k8s-node02 k8s-node03

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

master 上查看状态

```bash
[root@k8s-master01 ssl]#kubectl get csr
NAME                                                   AGE   REQUESTOR           CONDITION
node-csr-_QqF2VlG6XUI_JqmpzROD3s0Sts9WPAfpq7rP7dCgP0   19s   kubelet-bootstrap   Pending
node-csr-kiPZ4YSRJeIRkyEiOR4cv8xXMUwe-xME11ruR3zfAr8   42s   kubelet-bootstrap   Pending
node-csr-zd18cFbrHELmdqtggr6KdhqP0c_iO6ZuGf34Sf2oXTA   12s   kubelet-bootstrap   Pending


# 批准kubelet 的 TLS 证书请求
kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve

# 再次查看状态
[root@k8s-master01 ssl]# kubectl get csr
NAME                                                   AGE     REQUESTOR           CONDITION
node-csr-_QqF2VlG6XUI_JqmpzROD3s0Sts9WPAfpq7rP7dCgP0   4m41s   kubelet-bootstrap   Approved,Issued
node-csr-kiPZ4YSRJeIRkyEiOR4cv8xXMUwe-xME11ruR3zfAr8   5m4s    kubelet-bootstrap   Approved,Issued
node-csr-zd18cFbrHELmdqtggr6KdhqP0c_iO6ZuGf34Sf2oXTA   4m34s   kubelet-bootstrap   Approved,Issued
```

##### kube-proxy安装

```bash
yum install -y ipvsadm ipset conntrack
cd /usr/local/src/ssl/
vim kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```



```bash
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes  kube-proxy-csr.json | cfssl-json -bare kube-proxy

cp kube-proxy*.pem /opt/kubernetes/ssl/
scp kube-proxy*.pem 192.168.0.33:/opt/kubernetes/ssl/
scp kube-proxy*.pem 192.168.0.44:/opt/kubernetes/ssl/
scp kube-proxy*.pem 192.168.0.55:/opt/kubernetes/ssl/
```

创建kube-proxy配置文件

```bash
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.0.11:6443 \
   --kubeconfig=kube-proxy.kubeconfig
   
kubectl config set-credentials kube-proxy \
   --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
   --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
   --embed-certs=true \
   --kubeconfig=kube-proxy.kubeconfig
   
kubectl config set-context default \
   --cluster=kubernetes \
   --user=kube-proxy \
   --kubeconfig=kube-proxy.kubeconfig
   
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

分发kubeconfig配置文件

```
cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
scp kube-proxy.kubeconfig 192.168.0.33:/opt/kubernetes/cfg/
scp kube-proxy.kubeconfig 192.168.0.44:/opt/kubernetes/cfg/
scp kube-proxy.kubeconfig 192.168.0.55:/opt/kubernetes/cfg/
```

创建kube-proxy服务配置

```bash
# K8s-node01

mkdir /var/lib/kube-proxy
vim /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \
  --bind-address=192.168.0.33 \
  --hostname-override=192.168.0.33 \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig \
--masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target


## 拷贝并修改相应ip
scp /usr/lib/systemd/system/kube-proxy.service root@k8s-node02:/usr/lib/systemd/system/kube-proxy.service
scp /usr/lib/systemd/system/kube-proxy.service root@k8s-node03:/usr/lib/systemd/system/kube-proxy.service

# k8s-node02 上修改配置文件

vim /usr/lib/systemd/system/kube-proxy.service
  --address=192.168.0.44
  --hostname-override=192.168.0.44
  
# k8s-node03 上修改配置文件

vim /usr/lib/systemd/system/kube-proxy.service
  --address=192.168.0.55
  --hostname-override=192.168.0.55
```

查看状态

```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
ipvsadm -L -n
```

##### Flannel安装

```bash
cd /usr/local/src/ssl
vim flanneld-csr.json
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

生成证书

```bash
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes flanneld-csr.json | cfssl-json -bare flanneld
```

同步证书

```bash
cp flanneld*.pem /opt/kubernetes/ssl/
scp flanneld*.pem 192.168.0.22:/opt/kubernetes/ssl/
scp flanneld*.pem 192.168.0.33:/opt/kubernetes/ssl/
scp flanneld*.pem 192.168.0.44:/opt/kubernetes/ssl/
scp flanneld*.pem 192.168.0.55:/opt/kubernetes/ssl/
```

安装

```bash
cd /usr/local/src
wget
 https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
tar zxf flannel-v0.10.0-linux-amd64.tar.gz
cp flanneld mk-docker-opts.sh /opt/kubernetes/bin/
scp flanneld mk-docker-opts.sh 192.168.0.22:/opt/kubernetes/bin/
scp flanneld mk-docker-opts.sh 192.168.0.33:/opt/kubernetes/bin/
scp flanneld mk-docker-opts.sh 192.168.0.44:/opt/kubernetes/bin/
scp flanneld mk-docker-opts.sh 192.168.0.55:/opt/kubernetes/bin/

vim remove-docker0.sh
#!/usr/bin/env bash
set -e

rc=0
ip link show docker0 >/dev/null 2>&1 || rc="$?"
if [[ "$rc" -eq "0" ]]; then
  ip link set dev docker0 down
  ip link delete docker0
fi

#cd /usr/local/src/kubernetes/cluster/centos/node/bin/
cp remove-docker0.sh /opt/kubernetes/bin/
scp remove-docker0.sh 192.168.0.22:/opt/kubernetes/bin/
scp remove-docker0.sh 192.168.0.33:/opt/kubernetes/bin/
scp remove-docker0.sh 192.168.0.44:/opt/kubernetes/bin/
scp remove-docker0.sh 192.168.0.55:/opt/kubernetes/bin/
```

配置flannel

```bash
vim /opt/kubernetes/cfg/flannel
FLANNEL_ETCD="-etcd-endpoints=https://192.168.0.33:2379,https://192.168.0.44:2379,https://192.168.0.55:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/opt/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/opt/kubernetes/ssl/flanneld.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/opt/kubernetes/ssl/flanneld-key.pem"

scp /opt/kubernetes/cfg/flannel 192.168.0.22:/opt/kubernetes/cfg/flannel
scp /opt/kubernetes/cfg/flannel 192.168.0.33:/opt/kubernetes/cfg/flannel
scp /opt/kubernetes/cfg/flannel 192.168.0.44:/opt/kubernetes/cfg/flannel
scp /opt/kubernetes/cfg/flannel 192.168.0.55:/opt/kubernetes/cfg/flannel
```

设置Flannel系统服务

```bash
vim /usr/lib/systemd/system/flannel.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
Before=docker.service

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/flannel
ExecStartPre=/opt/kubernetes/bin/remove-docker0.sh
ExecStart=/opt/kubernetes/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker

Type=notify

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service

scp /usr/lib/systemd/system/flannel.service 192.168.0.22:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/flannel.service 192.168.0.33:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/flannel.service 192.168.0.44:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/flannel.service 192.168.0.55:/usr/lib/systemd/system/
```

flannel CNI集成

```bash
cd /usr/local/src
wget https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz
mkdir /opt/kubernetes/bin/cni			# 5台服务器都创建此目录
tar zxf cni-plugins-amd64-v0.7.1.tgz -C /opt/kubernetes/bin/cni
scp -r /opt/kubernetes/bin/cni/* 192.168.0.22:/opt/kubernetes/bin/cni/
scp -r /opt/kubernetes/bin/cni/* 192.168.0.33:/opt/kubernetes/bin/cni/
scp -r /opt/kubernetes/bin/cni/* 192.168.0.44:/opt/kubernetes/bin/cni/
scp -r /opt/kubernetes/bin/cni/* 192.168.0.55:/opt/kubernetes/bin/cni/
```

创建Etcd的key

```bash
/opt/kubernetes/bin/etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file /opt/kubernetes/ssl/flanneld-key.pem \
      --no-sync -C https://192.168.0.33:2379,https://192.168.0.44:2379,https://192.168.56.55:2379 \
mk /kubernetes/network/config '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' >/dev/null 2>&1

```

启动flannel

```bash
systemctl daemon-reload && systemctl restart flannel
systemctl enable flannel
chmod +x /opt/kubernetes/bin/*
systemctl start flannel
systemctl status flannel
```

配置docker

```bash
vim /usr/lib/systemd/system/docker.service
[Unit] #在Unit下面修改After和增加Requires
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service] #增加EnvironmentFile=-/run/flannel/docker
Type=notify
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS

scp /usr/lib/systemd/system/docker.service 192.168.0.22:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/docker.service 192.168.0.33:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/docker.service 192.168.0.44:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/docker.service 192.168.0.55:/usr/lib/systemd/system/
```

重启docker

```bash
systemctl daemon-reload
systemctl restart docker
```

##### coredns安装



##### dashboard安装



#### 2.2.2 kubeadm安装

iptables并设置空规则

```bash
yum -y install iptables-services
systemctl start iptables && systemctl enable iptables && iptables -F && service iptables save
```

### 三. kubernetes

#### 3.1 资源组件

##### 3.1.1 api-server





##### 3.1.2 controller manager



##### 3.1.3 controller



##### 3.1.4 kube-proxy

用来实现service包括Clusip包括一个负载均衡



##### 3.1.5 kubelet

管理和启动一个容器的生命周期



##### 3.1.6 pod

**pod分类：**

​	两类pod的最大区别就是 生命周期被管理的机制不一致

- 自主式pod

  ​		只要是pod退出了，就不会被重启，此pod没有管理者。

- 控制器pod

  ​		在控制器的的生命周期里，始终要维持pod的副本数目。



		- 自主式pod
			- pod退出了，此类的pod不会被创建
		- 控制器管理pod
			- 在控制器的的生命周期里，始终要维持pod的副本数目

**容器分类**
	Infrastructure Container：基础容器
		用户不可见，无需感知
		维护整个Pod网络空间
	InitContainers: 初始化容器，一般用于服务等待处理以及注册Pod信息等
		先于业务容器开始执行
		顺序执行，执行成功退出，全部执行成功后开始启动业务容器
	Containers：业务容器
		并行启动，启动成功后一直Runing



##### 3.1.7 etcd



##### 3.1.8 flannel



##### 3.1.9 coredns

**3.2.0 service**

Kubernetes Service 定义了这样一种抽象: 一个pod的逻辑分组，一种可以访问它们的策略--通常称为微服务，这一组pod能够被Service访问到，通常是通过Label Selector，只能提供4层的负载均衡，只能通过ip和端口进行转发。

**service的四种类型:**

- ClusterIp: 默认类型，自动分配一个仅Cluster内部可以访问的虚拟IP
- NodePort: 在node上开一个端口，将向该端口的流量导入到kube-proxy，然后又kube-proxy进一步到对应的pod
- LoadBalancer: 在NodePort的基础上，借助cloud provider创建一个外部负载均衡器，并将请求转发到NodeIp:NodePort
- Externalname: 把集群外部的服务引入到集群内部来，在集群内部直接使用，没有任何类型代理被创建，这只有kuberneres1.7或更高版的kube-dns才支持

**流程：**

- api server通过监控kube-proxy，去进行服务和端点的发现或监控（watch Services and Endpoints）
- kube-proxy通过pod的标签判断信息是否写入到ipvs的规则里去，当客户端访问svc其实就是访问ipvs的规则，从而调度到后端的pod。





#### 3.2 资源清单

##### 3.2.1 资源类型

- 名称空间级别

  ```bash
  	- 工作负载型资源:
  		- pod  共享网络栈，共享存储卷
  		- Replicaset  		# 副本集控制器，通过标签的选择控制pod的数量。
  		- Deployment       # 生命式控制器，只能控制无状态的应用。通过控制RS的创建去创建Pod
  		- StatefulSet 		 # 有状态的副本集
  		- Daemonset				 # 每个节点都一个运行一个组件
  		- Job 					   # 为批处理而生		
  		- Cronjob				   # 轮训任务，为批处理而生
  		
  	- 服务发现及负载均衡型资源(service discovery loadbalance):
  		- Service
  		- ingress 				 # 这两者都是为了将我们的服务暴露出去
  		
  	- 配置与存储资源:
  		- Volume				   # 存储卷
  		- CSI					     # 容器存储接口，可以扩展各种各样的第三方存储卷
  		
  	- 特殊类型的存储卷:
  		- ConfigMap				 # 当配置中心来使用的资源类型
  		- Secret 			     # 保存敏感数据
  		- DownwardAPI		   # 把外部环境中的信息输出给容器
  ```

- 集群级别

- 元数据型

##### 3.2.2 资源清单

```bash
	- apiVerersion(必)					 # 版本
		- group/version
	- kind (必)    					   # 资源类别和角色 eg: pod
	- metadata (必) 					 	 # 元数据
		- name(必)								 # 元数据的名称
		- namespace      		    	# 定义名称空间，默认default
		- lables     			      	# 标签
		- annotations			      	# 注解 
	- spec(必)						       # 详细定义对象 期望状态
		- containers(必)
			- name                  # 容器名字
			- image(必)						 # 镜像名称
			- imagePullpolicy 		  # 镜像的下载策略
				- Always							# 总是到仓库中查找 默认always
				- Never			 					# 永远用本地
				- ifNotPresent	 			# 本地如果没有就下载
			- command								# 相当于docker cmd
			- args									# 启动命令的一些参数 比如上面ls 加一个 -a
			- workingDir						# 进入容器的工作目录
			- volumeMounts					# 指定容器内的存储卷配置
				- name								# 指定可以被容器挂载的存储卷名称
				- mountPath						# 指定可以被容器挂载的存储卷的路径
				- readOnly						# 存储卷路径的读写模式，true或者false  默认是读写模式
		  - ports 									# 指定容器用到的端口列表
			  - name 									# 端口名称
		  	- containerPort 				# 容器监听的端口
			  - hostPort							# 主机监听的post，默认和containerpost相同
			  - protocol 							# 指定端口协议，默认为TCP
		  - env
		    - name									# 环境变量名称
		    - value									# 环境变量值
		  - resources  							# 指定资源限制和资源请求的值
		  	- limits								# 指定容器运行时资源的运行上限
		  		- cpu									# CPU限制 单位为core数
		  		- memory							# 指定MEM内存的限制，单位为MIB GIB
		  		- requests						# 指定容器启动和调度时的限制设置
		  			- cpu								# CPU请求，单位为core数，容器启动时初始化可用数
		  			- memory
		- restartPolicy
			- Always 								# 总是重启，也有限制
			- OnFailure 						# 如果推出的状态码为非0就重启
			- Never									# 从不重启
		- nodeSelector 						# 选择node节点运行
		- imagePullSecret					# 
		- hostNetwork							# 定义使用主机网络模式，默认为false，设置为true表示使用宿主机网络，不使用docker网桥，
```

##### 3.2.3 pod生命周期

![image-20200428234722783](/Users/shaowei/Library/Application Support/typora-user-images/image-20200428234722783.png)

###### 执行过程

1. 首先kubctl向api server发送指令后，api server创建controller，将目标状态保存到etcd中。
2. api server接下来请求schedule请求调度，并且把调度的结果保存到etcd中。
   - 比如调度到那个node，将信息保存到etccd中的pod状态信息当中。
3. 随后目标node节点的kubelet通过api server中的状态变化有任务给自己，所以此时此kubelet通过api server 拿到清单在当前节点去创建pod， 去操作CRI，CRI完成容器的初始化，
   - 初始化会先启动一个pause的基础容器，负责同一个pod中的网络和存储的共享。
    - 接着依次进行多个init c的初始化，也可以没有init c
     - 如果执行失败，k8s会不断重启pod知道init容器成功为止，当然根据重启策略有关restartPolicy，如果为Nerver，就不会重启
    - 随后进入主容器的运行，运行前后分别会执行start和stop的命令或脚本。

在过程中会有readmess和liveless的参与，可以在多少秒后执行readless和liveless的探测，如果readless没有执行pod的状态是不可能running，liveless会伴随着这个容器的生命周期，当主容器的进程和liveless的探测不一致时会重启或暂停。

######  探针

探针是由kubelet对容器执行的定期诊断，要执行诊断，kubelet调用由容器实现的Handler，

**有三种类型的处理程序：**

- ExecAction：在容器内执行命令，如果命令退出时返回码为0，则认为诊断成功。
- TCPSocketAction：对指定端口上的容器的IP地址进行TCP检查，如果端口打开，则诊断认为成功。
- HttpGetAction：执行HTTP Get请求，如果响应的状态码大于等于200且小于400，则诊断认为成功。

**探测方式：**

- readinessprobe: 指示容器是否准备好服务请求，如果就绪探测失败，端口控制器将从与pod匹配的所有Server的端点中删除Pod的IP地址，初始延迟之前的就绪状态默认为Failure，如果容器不提供就绪探针，则默认状态为Success

- livenessProbe: 指示容器是否正在运行，如果存活探针失败，则kubelet会杀死容器，并且容器将受到其重启策略的影响，如果容器不提供存活探针，则默认为Success。

  

**就绪检测（readinessProbe）**

readinessProbe -  HttpGetAction

```yaml
$ vim pod.yaml
apiVersion: v1
kind: Pod
metadata:
	name: readiness-http-pod
	namespace: default
spec:
	containsers:
	- name: readiness-httpger-container
	  image: golangav.com/library/myapp:v1
	  imagePullPolicy: IfNotPresent
	  readinessProbe:
	  	httpGet:
	  	  port: 80
	  	  path: /index.html
	  	initialDelaySeconds: 1													# 启动1秒后开始探测
	  	periodSeconds: 3							    							# 重试循环时间为3秒
	  	
$ kubectl create -f pod.yaml
$ kubectl get pod 																		# 发现STATUS为Running，但READY为0/1
$ kubectl exec readiness-http-pod -it -- /bin/sh 			# 进入容器
$  echo "Hello word" >> /usr/share/nginx/html/index.html
$ kubectl get pod 																		# 再次查看容器正常
```

**存活检测 （livenessProbe）**

libenessProbe - ExecAction

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: liveness-exec-pod
	namespace: default
spec:
	containsers:
	- name: liveness-exec-container
	  image: golangav.com/library/myapp:v1
	  imagePullPolicy: IfNotPresent
	  command: ["/bin/sh","-c","touch /tmp/live; sleep 60; rm -rf /tmp/live; sleep 3600"]
	  livenessProbe:
	  	exec:
	  		command: ["test","-e","/tmp/live"]
	  	initialDelaySeconds: 1						# 启动1秒后开始探测
	  	periodSeconds: 3							    # 重试循环时间为3秒
```

libenessProbe - Httpget

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: liveness-exec-pod
	namespace: default
spec:
	containsers:
	- name: liveness-exec-container
	  image: golangav.com/library/myapp:v1
	  imagePullPolicy: IfNotPresent
		ports:
		- name: http
		  containerPort: 80
	  livenessProbe:
	  	httpGet:
	  	  port: http
	  	  path: /index.heml
	  	initialDelaySeconds: 1						# 启动1秒后开始探测
	  	periodSeconds: 3							    # 重试循环时间为3秒
	  	timeoutSeconds: 10
```

libenessProbe - tcp

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: liveness-exec-pod
	namespace: default
spec:
	containsers:
	- name: liveness-exec-container
	  image: golangav.com/library/myapp:v1
	  livenessProbe:
			tcpSocker:
			  port: 80
	  	initialDelaySeconds: 5						# 启动1秒后开始探测
	  	periodSeconds: 3							    # 重试循环时间为3秒
	  	timeoutSeconds: 10
```

###### 启动退出动作

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: liveness-exec-pod
	namespace: default
spec:
	containsers:
	- name: liveness-exec-container
	  image: golangav.com/library/myapp:v1
		lifecycle:
		  postStart:
		    exec:
		      command: ["/bin/sh","-c","echo start > /usr/share/message" ]
		  preStop:
		    exec:
		      command: ["/bin/sh","-c","echo stop > /usr/share/message" ]
```



#### 3.2 资源控制器

**什么事控制器？**

​	Kubernetes中内建了很多controller（控制器），这些相当于一个状态机，用来控制pod的状态和行为

**控制器的类型：**

- Replicaset 

  ```bash
  - 查缺补漏，确保容器应用的副本始终保持在用户定义的副本数，如果容器有异常退出，会自动创建新的pod来替代，而如果异常多出来的会自动回收。
  - ReplicaSet支持集合式的selector
  ```

- Deployment 

  ```bash
  - Deployment并不直接管理pod，而是通过RS来对Pod进行管理
  - Deployment为pod和Repliceset提供了一个声明式(declarative)方法，来替代以前ReplicationController来方便管理应用
  - 典型的应用场景包括：
  	- 定义Deployment来创建Pod和RepliSet
  	- 滚动升级和回滚应用
  	- 扩容和缩容
  	- 暂停和继续Deploment
  ```

- StatefulSet

  ```bash
  StatufulSet作为Controller为Pod提供唯一个标识。它可以保证部署和scale的顺序
  应用场景：
  	- 稳定的持久化存储，即Pod重新调度后还是能访问到相应的持久化数据，基于PVC实现
  	- 稳定的网络标志，即Pod重新调度后其PodNmade和HosrName不变，基于Headless Service（即没有Cluster IP的Server）来实现
  	- 有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次进行（即从0到N-1,在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于 init containers来实现。
  	- 有序收缩，有序顺序（即从N-1到0）
  ```

- Daemonset

  ```bash
  指定Pod运行在哪个Node上，也就是k8s的调度的选择可以认为控制（标签选择和污点去控制），每个Node上只能运行一个
  使用DeemonSet的一些典型用法:
  	- 运行集群存储daemon，例如在每个Node上运行glusterd，ceph
  	- 在每个Node上运行日志手机daemon，例如fluentd，logstash
  	- 在每个Mode上运行监控daemon，例如Promentheus Node Exporter、colletcd、Datadog代理、New Relic代理、Ganlia gmond
  ```

- Job 

  ```bash
  job负责批处理任务，即仅执行一个的任务，它保证批处理任务的一个或多个Pod成功结束
  ```

- Cronjob

  ```bash
  在给定时间点只运行一次
  周期性地在给定时间点运行
  典型的应用实例：
  	- 在给定时间点调度job运行
  	- 创建周期性运行的job，例如： 数据库备份、发送邮件。
  ```

- Horizontal Pod Autoscaling

  ```bash
  应用的资源使用率通常都有高峰和低谷的时候，如何消峰填谷，提高集群的整体资源利用率，让service的Pod个数自动调整呢？使用Pod水平自动缩放
  用来管理其它Pod
  例如CPU使用率大于百分之80，就恢复到100个Pod
  ```

**RS演示**

```yaml
apiVersion: extension/v1beta1
kind: ReplicaSet
metedata:
   name: frontend
spec:
	replicas: 3
	selector:
	   matchLabels:
	      tier: fronted
	template:
	   matadata:
	     labels:
	        tier: frontend
	    spec:
	      containers:
	      - name: php-redis
	        image: golangav.com/google_samples/gb-fronted:v3
	        env:
	        - name: GET_HOST_FROM
	          value: dns
	        ports:
        	- containerPort: 80
```

**Deployment演示**

扩容 回滚

```yaml
$ vim deployment.yaml
apiVersion: extension/v1beta1
kind: Deployment
metedata:
   name: nginx-deplyment
spec:
	replicas: 3
	template:
	   matadata:
	     labels:
	        app: nginx
	    spec:
	      containers:
	      - name: nginx
	        image: nginx:1.7.9
			ports:
			- containerPort: 80   

$ kubectl apply -f  deployment.yaml --record												# 加日志
$ kubectl get deployment    
$ kubectl get rs
$ kubectl get pod
$ kubectl scale deployment nginx-deplyment --replicas 10     				# 扩容
$ kubectl set image deployment/nginx-deplyment nginx=nginx:1.9.1		# 更新镜像
$ kubectl rollout undo deployment/nginx-deployment 						      # 回滚
$ kubectl get pod -w wide
$ kubectl rollout status deployment/nginx-deployment						    # 查看回滚状态
$ kubectl rollout history deployment/nginx-deployment
```

**Daemonset演示**

**Daemonset演示**

每个node上运行一个pod

```yaml
$ vim daemonset.yaml
	apiVersion: app/v1
	kind: DaemonSet
	metedata:
	   name: deamonset-example 				# 和下面名字一致
	   labels:
	     app: daemonset
	spec:
		selector:
		   macheLabels:
		      name: deamonset-example     	# 和上面名字一致
		template:
		   matadata:
		     labels:
		        app: deamonset-example
		    spec:
		      containers:
		      - name: deamonset-example
		        image: golangav.com/myapp:v1

$ kubectl create -f daemonset.yaml
$ kubectl get pod -o wide 				   # 发现每个node上创建一个pod
```

**Job**

```yaml
- 执行per的一个命令，从不重启
$ vim job.yaml
	apiVersion: batch/v1
	kind: Job
	metedata:
	   name: p1
	spec:
		template:
		   matadata:
		     name: p1
		    spec:
		      containers:
		      - name: p1
		        image: per1
				command: ["perl","-Mbignum=bpi","-wle","print bpi(2000)"]
			  restartPolicy: Never	    
$ kubectl get job
```



#### 3.3 存储

##### 3.3.1 ConfigMap

- 许多应用程序从配置文件、命令行参数或环境变量中读取配置信息，ConfigMap Api给我们提供了向容器中注入配置信息的机制

- CongigMap可以被用来保存单个属性，也可以用来保存整个配置文件或者JSON二进制大对象。

###### **ConfigMap**的创建

**使用目录创建**

```bash
$ ls docs/user-guide/configmap/kubectl/
game.properties
ui.properties

$ cat docs/user-guide/configmap/kubectl/game.properties
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
enemies.code.passphrase=UUDDRLRBABAS
enemies.code.allowed=true
enemies.lives=30

$ cat docs/user-guide/configmap/kubectl/ui.pro=perties
color.good=purple
color.bad=yellow
color.textmode=true
color.to.look=fairlyNice

$ cubectl create configmap game-config --from-file=docs/user-guide/configmap/kubectl

$ kubectl get cm
$ kubectl descrebe cm game-config
$ kubectl get cm game-config -o yaml
```

**使用文件创建**

```bash
$ cubectl create configmap game-config --from-file=docs/user-guide/configmap/kubectl/game.properties

$ kubectl get cm
$ kubectl descrebe cm game-config
$ kubectl get cm game-config -o yaml
```

**使用字面值创建**

```bash
$ kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm

$ kubectl get cm special-config -o yaml
```

###### ConfigMap使用案例

**1.使用ConfigMap替代环境变量**

创建两个pod，注入到另一个pod的环境变量中

```bash
$ vim special.yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: special-config
	namespace: default
data:
    special.how: very
    special.type: charm
$ kubectl apply -f special.yaml	

$ vim env.yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: env-config
	namespace: default
data:
	log_level: INFO	
$ kubectl apply -f env.yaml	

$ vim configmap_pod.yaml
apiVersion: v1
kind: Pod
metadata:
   name: depi-test-pod
spec:
   containers:
     - name: test-container
       image: golangav.com/library/myapp:v1
       command: ["/bin/sh","-c","env"]
       env:
         - name: SPECIAL_LEVEL_KEY
	       valueFrom:
	         configMapKeyRef:
	            name: special-config
	            key: special.how
         - name: SPECIAL_TYPE_KEY
           valueFrom:
	         configMapKeyRef:
	            name: special-config
	            key: special.type
	    envfrom:
	      - configMapRef:
	          name: env-config
	restartPolicy: Never
$ kubectl create -f configmap_pod.yaml
$ kubectl get pod
$ kubectl log depi-test-pod
```

**2.使用ConfigMap设置为命令行参数**

```bash
$ vim special.yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: special-config
	namespace: default
data:
    special.how: very
    special.type: charm
$ kubectl apply -f special.yaml	

$ vim configmap_pod.yaml
apiVersion: v1
kind: Pod
metadata:
   name: depi-test-pod
spec:
   containers:
     - name: test-container
       image: golangav.com/library/myapp:v1
       command: ["/bin/sh","-c","echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)"]
       env:
         - name: SPECIAL_LEVEL_KEY
	       valueFrom:
	         configMapKeyRef:
	            name: special-config
	            key: special.how
         - name: SPECIAL_TYPE_KEY
           valueFrom:
	         configMapKeyRef:
	            name: special-config
	            key: special.type
	restartPolicy: Never
```

**3. 通过数据卷插件使用ConfigMap**

```bash
$ vim special.yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: special-config
	namespace: default
data:
    special.how: very
    special.type: charm
$ kubectl apply -f special.yaml	
# 键就是文件名，值就是文件内容
$ vim configmap_pod.yaml
apiVersion: v1
kind: Pod
metadata:
   name: depi-test-pod
spec:
   containers:
     - name: test-container
       image: golangav.com/library/myapp:v1
       command: ["/bin/sh","-c","sleep 600s"]
       volumeMounts:
       - name: config-volume
         mountPath: /etc/config
   volumes:
     - name: config-volume
       configMap:
         name: special-config
	restartPolicy: Never
$ kubectl create -f configmap_pod.yaml
$ kubectl get pod
$ kubectl exec depi-test-pod -it -- /bin/sh
$ cat /etc/config/special.how
	very
$ cat /etc/config/special.type
	charm
```

##### 3.3.2 Secret

secret 解决了密码、token、秘钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者Pod Spec中，Secret可以以Volume或者环境变量的方式使用

###### Secret 三种类型

- Service Account: 用来访问kubernets API，由kubernets自动创建，并且会自动挂载到Pod的/run/secrets/kubernets.io/serviceaccount目录中

- Opaque: base64编码格式Secret，用来存储密码、秘钥等，数据是map类型

- kubernets.io/dockerconfigjson: 用来存储私有 decker registrv的认证信息



###### Sceret的创建

**Service Account**

```bash
$ kubectl run nginx --image nginx
$ kubectl get pods
$ kubectl exec nginx ls /run/secrets/kubernets.io/serviceaccount
	ca.crt
	namespace
	token
```

**Opaque Secret**

````bash
# 用户名和密码加密
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "123456" | base64
MTIzNDU2

$ vim secret.yaml
apiVersion: v1
kind: Secret
metadata:
   name: mysecret
type: Opaque
data:
   username: YWRtaW4=
   password: MTIzNDU2
$ kubectl create -f secret.yaml

# 将Secret挂载到Volume中

$ vim secret-pod.yaml
apiVersion: v1
kind: Pod
metadata:
   name: secret-test
spec:
   volumes:
     - name: secret
       secret:
         secretName: mysecret
   containers:
     - name: db
       image: golangav.com/library/myapp:v1
       volumeMounts:
       - name: secrets
         mountPath: /etc/secrets				         
		 readOnly: true
		 
$ kubectl create -f secret-pod.yaml
$ kubectl get pod
$ kubectl exec secret-test la /etc/secrets
	username  password
	
# 将Secret导入到环境变量中

$ vim pod.yaml
apiVersion: v1
kind: Deployment
metadata:
   name: pod-deployment
spec:
   replicas: 2
   template:
     metadata:
       labels:
         app: pod-deployment
     spec:
	   containers:
	     - name: pod-1
	       image: golangav.com/library/myapp:v1
	       ports:
	       - containerPort: 80
	       env:
	         - name: TEST_USER
		       valueFrom:
		         SecretKeyRef:
		            name: mysecret
		            key: username
	         - name: TEST_PASSWORD
		       valueFrom:
		         SecretKeyRef:
		            name: mysecret
		            key: password
		            

# 通过仓库拉取镜像
$ kubectl creat sectret docker-registry myregistrykey --docker-server=hub.goalngav.com --docker-username=admin --docker-passwoed=123456 --docker-email=golangav.com@163.com
$ vim pod.yaml
	apiVersion: v1
	kind: Pod
	metadata:
	  name: foo
	spec:
	  containers:
	    - name: foo
	      image: hub.golangac.com/test/myapp:v2
	  imagePullSecrets:
	    - name: myregistrykey
$ kubectl create -f pod.yaml 
````

##### 3.3.3 volume

为pod提供一个存储卷的能力，例如NFS共享，本地某个目录共享。

**kubernetes支持的卷:**

```bash
aesElasticBlockStore
azureDisk
azureFile
cephfs
csi
downwardAPI
emptyDir
fc
flocker
gcPersistenDisk
gitRepo
glusterfs
hostPath
iscsi
local
nfs
persistenVolumeClaim
projected
portworxVolume
quobyte
rbd
scaleIO
secret
storageos
vsphereVolume
emptyDir
```

**emptyDir**

emptyDir 使用场景：

- 暂存空间，例如用于基于磁盘的合并排序
- 用作长时间计算崩溃恢复时的检查点
- web服务器容器提供数据时，保存内容管理器提取的文件

```bash
$ vim pod.yaml
  apiVersion: v1
  kind: Pod
  metadata:
     name: test-pod
  spec:
	    containers:
	    - name: test-container
	      image: golangav.com/library/webserver
	      volumeMount:
	      - mountPath: /cache
	        name: cache-volume
	    volumes:
	    - name: cache-volume
	    emptyDir: {}
$ kubectl apply -f pod.yaml
$ kubectl exec test-pod ls /cache/
```

##### 3.3.4 pvc

Perrsistent Volume（pv 持久卷）








### 3.4 Scheduler

	- 调度过程：
		- 调度分为几个部分，首先是过滤掉不满足条件的节点，这个过程称为predicate，如果没有合适的节点pod会一直在pending状态，不断充实调度知道有节点满足条件，经过这个步骤，如果有多个满足条件，就继续priorities过程；
		- 然后对通过的节点按照优先级排序，这个是priority 优选；
		- 最后从中选择优先级最高的节点。如果中间任何一步骤有序错误，就直接返回错误
	
		- predicate 有一系列的算法可以使用：
			- PodFitsResources: 节点剩余的资源是否大于pod的请求资源
			- PodFitsHost: 如果pod制定了NodeNmae，检查节点名称是否和NodeName匹配
			- PodFitsHostPorts: 节点已经使用的port是否和pod申请的port冲突
			- podSelectorMatches: 过滤掉和pod指定的label不匹配的节点
			- NoDiskConflict: 已经mount的volume和pod指定的volume不冲突，除非它们都是只读
		- priority 
			- 优先级由一系列键值组成，键是优先级的名称，值是它的权重，优先级选项包括：
				- LeastRequestedPriority  通过计算CPU和Memory的使用率来决定权重，使用率越低权重越高
				- BalanceResourceAllocation 节点上CPU和Memory使用率越接近，权重越高，这个应该和上面的一起使用，不应该单独使用
				- ImageLocalityPriority 倾向于已经有要使用镜像的节点，镜像总大小值越大，权重越高
	- 调度的亲和性
		- requiredDuringSchedulingIgnoreDuringExecution   硬策略
		- preferredDuringSchedulingIgnoreDuringExecution  软策略
		
		- 节点亲和
			- 运行一个pod，不让在node2上运行
			vim pod.yaml
				apiVersion: v1
				kind: Pod
				metadata:
				  name: affinity
				  labels:
				    app: node-affinity-pod
				spec:
				  containers:
				  - name: with-node-affinity
				    iamge: golangac.com/myapp:v1
				  affinity:
				    nodeAffinifty:
				      requiredDuringSchedulingIgnoreDuringExecution:
				        nodeSelectorTerms:
				        -  matchExpressions:
				           - key: kubernets.io/hostname
				             operator: NotIn
				             values:
				             - kus-node02


四. Helm

Helm本质是让k8s的应用管理（Deployment Service等）可配置，能动态生成，通过动态生成k8s资源清单文件（deployment.yaml service.yaml）,然后调用Kubectl自动执行k8s资源部署

Helm是官方提供的类似于YUM的包管理器，是部署环境的流程封装，Helm有两个重要概念：

- chart是创建一个应用的信息结合，包括各种Kubernetes对象得配置模板，参数定义、依赖关系、文档说明等。chart是应用部署的自包含逻辑单元。可以将chart想象成apt、yum中的软件安装包
- release 是chart的运行实例，代表了一个正在运行的应用，当chart被安装到Kubernetes集群，就生成一个release，chart能够多次安装到同一个集群，每次安装都是一个release



## Prometheus

### Prometheus机构图

![image-20200519204055698](/Users/shaowei/Library/Application Support/typora-user-images/image-20200519204055698.png)

### Prometheus存储设计

![image-20200519205758713](/Users/shaowei/Library/Application Support/typora-user-images/image-20200519205758713.png)

- 以一段时间间隔(默认两小时)作为一个时间块存储，分为多个块，其中第一块mutable存储在内存中，存储当前的数据。
- 存储块的数据格式：
  	chunks: 里面包含多个块，每个块里面存储具体数据。
    	index：存储索引
    	meta.json：存储元数据相关信息 名字、tag
    	tombstons：标记删除
- 查询时指点时间的快进行查询







![image-20200519211311823](/Users/shaowei/Library/Application Support/typora-user-images/image-20200519211311823.png)



### Metrics种类

​	Counter(计数器)
​		始终增加
​		http请求数
​		下单数
​	Gauge(测量仪)
​		当前值得一次快照(snapshot)测量，可增可减
​		磁盘使用率
​		当前同时在线用户数
​	Histogram(直方图)
​		通过分桶方式统计样本分布
​		比如延迟多少在0-20ms,多少在20-40ms
​	Summary(汇总)
​		客户端计算好，根据样本统计出百分位