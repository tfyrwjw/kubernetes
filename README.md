# kubernetes
kubernetes dashboard images

下载版本为安装k8s二进制安装的1.16.0版本
二进制部署k8s集群
这里所有的文件均在文档同目录下，可以直接使用，二进制包根据提供的地址去下载。
全部下载地址：https://github.com/tfyrwjw/kubernetes/releases/download/k8s-deploy/k8s-deploy.zip
这里包含了1.16.0的二进制版本
3.1 服务器规划
角色	IP	组件
k8s-master1	192.168.31.63	kube-apiserver
kube-controller-manager
kube-scheduler
etcd
k8s-master2	192.168.31.66	kube-apiserver
kube-controller-manager
kube-scheduler

k8s-node1	192.168.31.64	kubelet
kube-proxy
docker
etcd
k8s-node2	192.168.31.65	kubelet
kube-proxy
docker
etcd
3.2 系统初始化
关闭防火墙：
# systemctl stop firewalld
# systemctl disable firewalld

关闭selinux：
# setenforce 0 # 临时
# sed -i 's/enforcing/disabled/' /etc/selinux/config # 永久
关闭swap：
# swapoff -a  # 临时
# vim /etc/fstab  # 永久
同步系统时间：
# ntpdate time.windows.com
添加hosts：
# vim /etc/hosts
192.168.31.63 k8s-master1
192.168.31.64 k8s-master2
192.168.31.65 k8s-node1
192.168.31.66 k8s-node2
修改主机名：
hostnamectl set-hostname k8s-master1
hostnamectl set-hostname k8s-master2
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2
3.3 Etcd集群
可在任意节点完成以下操作。
1.生成etcd证书
# cd TLS/etcd
安装cfssl工具：
# ./cfssl.sh
修改请求文件中hosts字段包含所有etcd节点IP：
# vi server-csr.json 
{
    "CN": "etcd",
    "hosts": [
        "192.168.31.63",
        "192.168.31.64",
        "192.168.31.65"
        ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
# ./generate_etcd_cert.sh
# ls *pem
ca-key.pem  ca.pem  server-key.pem  server.pem
2.部署三个Etcd节点
# tar zxvf etcd.tar.gz
# cd etcd
# cp TLS/etcd/ssl/{ca,server,server-key}.pem ssl
分别拷贝到Etcd三个节点：
# scp –r etcd root@192.168.31.63:/opt 
# scp etcd.service root@192.168.31.63:/usr/lib/systemd/system
登录三个节点修改配置文件、名称和IP：
# vi /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.31.63:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.31.63:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.31.63:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.31.63:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.31.63:2380,etcd-2=https://192.168.31.64:2380,etcd-3=https://192.168.31.65:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

# systemctl start etcd
# systemctl enable etcd
3.查看集群状态
# /opt/etcd/bin/etcdctl \
> --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem \
> --endpoints="https://192.168.31.63:2379,https://192.168.31.64:2379,https://192.168.31.65:2379" \
> cluster-health
member 37f20611ff3d9209 is healthy: got healthy result from https://192.168.31.63:2379
member b10f0bac3883a232 is healthy: got healthy result from https://192.168.31.64:2379
member b46624837acedac9 is healthy: got healthy result from https://192.168.31.65:2379
cluster is healthy
3.4 部署Master Node
1.生成apiserver证书
# cd TLS/k8s
修改请求文件中hosts字段包含所有etcd节点IP：
# vi server-csr.json 
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local",
      "192.168.31.60",
      "192.168.31.61",
      "192.168.31.62",
      "192.168.31.63",
      "192.168.31.64",
      "192.168.31.65",
      "192.168.31.66"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}

# ./generate_k8s_cert.sh
# ls *pem
ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  server-key.pem  server.pem
2.部署apiserver,controller-manager和scheduler
在Master节点完成以下操作。二进制包下载地址：https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.16.md#v1161
二进制文件位置：kubernetes/serverr/bin
# tar zxvf k8s-master.tar.gz
# cd kubernetes
# cp TLS/k8s/ssl/*.pem ssl
# cp –rf kubernetes /opt
# cp kube-apiserver.service kube-controller-manager.service kube-scheduler.service /usr/lib/systemd/system
修改对应配置文件里面的地址，启动，设置开机启动
# systemctl start kube-apiserver
# systemctl start kube-controller-manager
# systemctl start kube-scheduler
# systemctl enable kube-apiserver
# systemctl enable kube-controller-manager
# systemctl enable kube-scheduler
3.启用TLS Bootstrapping
为kubelet TLS Bootstrapping 授权：
# cat /opt/kubernetes/cfg/token.csv 
c47ffb939f5ca36231d9e3121a252940,kubelet-bootstrap,10001,"system:node-bootstrapper"
格式：token,用户,uid,用户组
给kubelet-bootstrap授权：
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap

token也可自行生成替换：
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
但apiserver配置的token必须要与node节点bootstrap.kubeconfig配置里一致。
3.5 部署Worker Node
1.安装Docker
二进制包下载地址：https://download.docker.com/linux/static/stable/x86_64/
# tar zxvf k8s-node.tar.gz
# tar zxvf docker-18.09.6.tgz
# mv docker/* /usr/bin
# mkdir /etc/docker
# mv daemon.json /etc/docker
# mv docker.service /usr/lib/systemd/system
# systemctl start docker
# systemctl enable docker
2.部署kubelet和kube-proxy
拷贝证书到Node：
# cd TLS/k8s
# scp ca.pem kube-proxy*.pem root@192.168.31.65:/opt/kubernetes/ssl/
# cp kube-apiserver.service kube-controller-manager.service kube-
# tar zxvf k8s-node.tar.gz
# mv kubernetes /opt
# cp kubelet.service kube-proxy.service /usr/lib/systemd/system

修改以下三个文件中IP地址：
# grep 192 *
bootstrap.kubeconfig:    server: https://192.168.31.63:6443
kubelet.kubeconfig:    server: https://192.168.31.63:6443
kube-proxy.kubeconfig:    server: https://192.168.31.63:6443

修改以下两个文件中主机名：
# grep hostname *
kubelet.conf:--hostname-override=k8s-node1 \
kube-proxy-config.yml:hostnameOverride: k8s-node1

# systemctl start kubelet
# systemctl start kube-proxy
# systemctl enable kubelet
# systemctl enable kube-proxy
3.允许给Node颁发证书
# kubectl get csr
# kubectl certificate approve node-csr-MYUxbmf_nmPQjmH3LkbZRL2uTO-_FCzDQUoUfTy7YjI
# kubectl get node
3.5 部署CNI网络
二进制包下载地址：https://github.com/containernetworking/plugins/releases
# mkdir /opt/cni/bin /etc/cni/net.d
# tar zxvf cni-plugins-linux-amd64-v0.8.2.tgz –C /opt/cni/bin
确保kubelet启用CNI：
# cat /opt/kubernetes/cfg/kubelet.conf 
--network-plugin=cni

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

在Master执行：
kubectl apply –f kube-flannel.yaml
# kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
kube-flannel-ds-amd64-5xmhh   1/1     Running   6          171m
kube-flannel-ds-amd64-ps5fx   1/1     Running   0          150m
3.6 授权apiserver访问kubelet
为提供安全性，kubelet禁止匿名访问，必须授权才可以。
# kubectl apply –f apiserver-to-kubelet-rbac.yaml
3.7 Master高可用
部署Master组件（与Master1一致）
拷贝master1/opt/kubernetes和service文件：
# scp –r /opt/kubernetes root@192.168.31.66:/opt
# scp –r /opt/etcd/ssl root@192.168.31.66:/opt/etcd
# scp /usr/lib/systemd/system/{kube-apiserver,kube-controller-manager,kube-scheduler}.service root@192.168.31.66:/usr/lib/systemd/system

修改apiserver配置文件为本地IP：

# cat /opt/kubernetes/cfg/kube-apiserver.conf 
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=https://192.168.31.63:2379,https://192.168.31.64:2379,https://192.168.31.65:2379 \
--bind-address=192.168.31.66 \
--secure-port=6443 \
--advertise-address=192.168.31.66 \
……

# systemctl start kube-apiserver
# systemctl start kube-controller-manager
# systemctl start kube-scheduler
# systemctl enable kube-apiserver
# systemctl enable kube-controller-manager
# systemctl enable kube-scheduler
