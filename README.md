# kubernetes在CentOS7下二进制文件方式安装、离线安装
## 一、安装和配置 docker（master和node节点部署） 
**1.1 下载最新的 docker 二进制文件**

```
https://download.docker.com/linux/static/stable/x86_64/docker-18.03.1-ce.tgz
tar -xvf docker-18.03.1-ce.tgz
cp docker/docker* /usr/bin
```

**1.2 创建 docker 的 启动文件**

```
cat /usr/lib/systemd/system/docker.service 
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
Environment="PATH=/root/local/bin:/bin:/sbin:/usr/bin:/usr/sbin"
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd --log-level=error $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
```

**1.3 启动 dockerd**
```
systemctl daemon-reload 
systemctl stop firewalld 
systemctl disable firewalld 
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat 
systemctl enable docker 
systemctl start docker
```

**1.4 检查 docker 服务**
```
service docker status
```

## 二、k8s部署准备 

**2.1 下载Kubernetes(简称K8S)二进制文件**
```
https://github.com/kubernetes/kubernetes/releases
```

**2.2 从上边的网址中选择相应的版本，本文以1.11.2版本为例，从 CHANGELOG页面 下载二进制文件**
![图片名称](https://img-blog.csdn.net/20180809215039897?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xqeDE1Mjg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  

**2.3 组件选择：选择Service Binaries中的kubernetes-server-linux-amd64.tar.gz **

`该文件已经包含了 K8S所需要的全部组件，无需单独下载Client等组件`
![图片名称](https://img-blog.csdn.net/20180809215112902?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xqeDE1Mjg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


## 三、安装规划
**3.1 下载K8S解压，把每个组件依次复制到/usr/bin目录文件下，然后创建systemd服务文见，最后启动该组件 **
**3.1.1 本例：以三个节点为例。具体节点安装组件如下 **

`
节点IP地址 角色 安装组件名称
192.168.1.180 Master（管理节点） etcd、kube-apiserver、kube-controller-manager、kube-scheduler
192.168.1.181 Node1（计算节点） docker 、kubelet、kube-proxy
`

## 四、Master节点部署 

**4.1 kubernetes master 节点包含的组件：**
```
kube-apiserver
kube-scheduler
kube-controller-manager
etcd
flannel
docker
```

**4.2 部署注意事项**
```
注意：在CentOS7系统 以二进制文件部署，所有组件都需要4个步骤： 
1）复制对应的二进制文件到/usr/bin目录下 
2）创建systemd service启动服务文件 
3）创建service 中对应的配置参数文件 
4）将该应用加入到开机自启
```

**4.3 etcd数据库安装**
```
下载：K8S需要etcd作为数据库。以 v3.2.9为例，下载地址如下： 
wget https://github.com/coreos/etcd/releases/download/v3.2.9/etcd-v3.2.9-linux-amd64.tar.gz 
https://github.com/coreos/etcd/releases/ 
tar xf etcd-v3.2.9-linux-amd64.tar.gz 
cd etcd-v3.2.9-linux-amd64/ 
cp etcd etcdctl /usr/bin/
```

**4.4 设置 etcd.service服务文件 **
**4.4.1 在/etc/systemd/system/目录里创建etcd.service，其内容如下：**
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service] 
Type=notify
WorkingDirectory=/var/lib/etcd/
#etcd配置文件路径
EnvironmentFile=-/etc/etcd/etcd.conf
 
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\""
Restart=on-failure
LimitNOFILE=65536

[Install] 
WantedBy=multi-user.target
# 说明：其中WorkingDirectory为etcd数据库目录，需要在etcd**安装前创建**
```

**4.4.2 创建配置/etc/etcd/etcd.conf文件**
```
[root@k8s-master]# cat /etc/etcd/etcd.conf 
#[Member] 
#ETCD_CORS="" 
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
#ETCD_LISTEN_PEER_URLS="http://localhost:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
#ETCD_MAX_SNAPSHOTS="5" #ETCD_MAX_WALS="5"
ETCD_NAME="default"
#ETCD_SNAPSHOT_COUNT="100000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
#ETCD_QUOTA_BACKEND_BYTES="0"
#ETCD_MAX_REQUEST_BYTES="1572864"
#ETCD_GRPC_KEEPALIVE_MIN_TIME="5s"
#ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s"
#ETCD_GRPC_KEEPALIVE_TIMEOUT="20s"
#
#[Clustering]
#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"
#ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
#ETCD_INITIAL_CLUSTER_STATE="new"
#ETCD_STRICT_RECONFIG_CHECK="true"
#ETCD_ENABLE_V2="true"
# 
#[Proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"
#
#[Security]
#ETCD_CERT_FILE=""
#ETCD_KEY_FILE=""
#ETCD_CLIENT_CERT_AUTH="false"
#ETCD_TRUSTED_CA_FILE=""
#ETCD_AUTO_TLS="false"
#ETCD_PEER_CERT_FILE=""
#ETCD_PEER_KEY_FILE=""
#ETCD_PEER_CLIENT_CERT_AUTH="false"
#ETCD_PEER_TRUSTED_CA_FILE=""
#ETCD_PEER_AUTO_TLS="false"
# #[Logging]
#ETCD_DEBUG="false"
#ETCD_LOG_PACKAGE_LEVELS=""
#ETCD_LOG_OUTPUT="default"
#
#[Unsafe]
#ETCD_FORCE_NEW_CLUSTER="false"
#
#[Version]
#ETCD_VERSION="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"
#
#[Profiling]
#ETCD_ENABLE_PPROF="false"
#ETCD_METRICS="basic"
#
#[Auth]
#ETCD_AUTH_TOKEN="simple"
```

**4.4.3 配置开机启动**
```
#systemctl daemon-reload
#systemctl enable etcd.service
#systemctl start etcd.service
```
**4.4.4 检验etcd是否安装成功**
```
# etcdctl cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://0.0.0.0:2379
cluster is healthy
```

**4.5 kube-apiserver服务 **

**4.5.1 下载并复制二进制文件到/usr/bin目录**
```
wget https://dl.k8s.io/v1.11.1/kubernetes-server-linux-amd64.tar.gz 
tar -xzvf kubernetes-server-linux-amd64.tar.gz 
cd kubernetes 
tar -xzvf kubernetes-src.tar.gz 
cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/bin/
```

**4.5.2 新建并编辑kube-apiserver.service 文件**
```
[root@k8s-master]#cat /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/bin/kube-apiserver $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBE_ETCD_SERVERS $KUBE_API_ADDRESS $KUBE_API_PORT $KUBELET_PORT $KUBE_ALLOW_PRIV $KUBE_SERVICE_ADDRESSES $KUBE_ADMISSION_CONTROL $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**4.5.3 新建参数配置文件/etc/kubernetes/apiserver**
```
[root@k8s-master]#cat /etc/kubernetes/apiserver
### 
# kubernetes system config
# 
# The following values are used to configure the kube-apiserver 
# 
# The address on the local server to listen to. 
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
# The port on the local server to listen on. 
KUBE_API_PORT="--port=8080"
# Port minions listen on 
# KUBELET_PORT="--kubelet-port=10250"

# Comma separated list of nodes in the etcd cluster 
KUBE_ETCD_SERVERS="--etcd-servers=http://192.168.56.100:2379"
# Address range to use for services 
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=192.168.1.0/24"
# default admission control policies 
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
# Add your own! 
KUBE_API_ARGS=""
```

**4.5.4 新建参数配置文件/etc/kubernetes/config**
```
### 
# kubernetes system config 
# 
# The following values are used to configure various aspects of all 
# kubernetes services, including 
# 
#   kube-apiserver.service 
#   kube-controller-manager.service 
#   kube-scheduler.service 
#   kubelet.service 
#   kube-proxy.service 
# logging to stderr means we get it in the systemd journal 
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug 
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers 
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver 
KUBE_MASTER="--master=http://192.168.56.100:8080"
```

**4.6 kube-controller-manger服务 **
**4.6.1 配置kube-controller-manager systemd 文件服务**
```
[root@k8s-master]#cat /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/bin/kube-controller-manager $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBE_MASTER $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**4.6.2 配置文件 /etc/kubernetes/controller-manager 内容如下：**
```
[root@k8s-master]#cat /etc/kubernetes/controller-manager
KUBE_CONTROLLER_MANAGER_ARGS=" "
```

**4.6.3 新建参数配置文件/etc/kubernetes/config**
```
### 
# kubernetes system config 
# 
# The following values are used to configure various aspects of all 
# kubernetes services, including 
# 
#   kube-apiserver.service 
#   kube-controller-manager.service 
#   kube-scheduler.service 
#   kubelet.service 
#   kube-proxy.service 
# logging to stderr means we get it in the systemd journal 
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug 
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers 
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver 
KUBE_MASTER="--master=http://192.168.56.100:8080"
```

**4.7 kube-scheduler服务 **
**4.7.1 配置kube-scheduler systemd服务文件**
```
[root@k8s-master]#cat /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/bin/kube-scheduler $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBE_MASTER $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**4.7.2 配置/etc/kubernetes/scheduler参数文件**
```
[root@k8s-master]#cat /etc/kubernetes/scheduler
#KUBE_SCHEDULER_ARGS="--logtostderr=true --log-dir=/home/k8s-t/log/kubernetes --v=2"
```

**4.7.3 新建参数配置文件/etc/kubernetes/config**
```
### 
# kubernetes system config 
# 
# The following values are used to configure various aspects of all 
# kubernetes services, including 
# 
#   kube-apiserver.service 
#   kube-controller-manager.service 
#   kube-scheduler.service 
#   kubelet.service 
#   kube-proxy.service 
# logging to stderr means we get it in the systemd journal 
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug 
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers 
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver 
KUBE_MASTER="--master=http://192.168.56.100:8080"
```

**4.8 将各组件加入开机自启**
```
systemctl daemon-reload 
systemctl enable kube-apiserver.service 
systemctl start kube-apiserver.service 
systemctl enable kube-controller-manager.service 
systemctl start kube-controller-manager.service 
systemctl enable kube-scheduler.service 
systemctl start kube-scheduler.service
```

**4.9 验证 master 节点功能**
```
[root@k8s-master]#kubectl get componentstatuses 
NAME STATUS MESSAGE ERROR 
controller-manager Healthy ok 
scheduler Healthy ok 
etcd-0 Healthy {"health": "true"}
```

## 第五章 Node 节点部署
**5.1 Docker部署请看第一章内容**
**5.2 kubernetes Node 节点包含如下组件：**

```
flanneld
docker
kubelet
kube-proxy
```

**5.3 安装和配置 kubelet**
```
weget https://dl.k8s.io/v1.11.1/kubernetes-server-linux-amd64.tar.gz 
tar -xzvf kubernetes-server-linux-amd64.tar.gz 
cd kubernetes 
tar -xzvf kubernetes-src.tar.gz 
sudo cp -r ./server/bin/{kube-proxy,kubelet} /root/local/bin/
```

**5.3.1 配置kubelet配置文件**
```
[root@k8s-node-1 ~]# cat /etc/kubernetes/config 

### 
# kubernetes system config 
# 
# The following values are used to configure various aspects of all 
# kubernetes services, including 
# 
#   kube-apiserver.service 
#   kube-controller-manager.service 
#   kube-scheduler.service 
#   kubelet.service 
#   kube-proxy.service 
# logging to stderr means we get it in the systemd journal 
KUBE_LOGTOSTDERR="--logtostderr=true"
# journal message level, 0 is debug 
KUBE_LOG_LEVEL="--v=0" 
# Should this cluster be allowed to run privileged docker containers 
KUBE_ALLOW_PRIV="--allow-privileged=false"
# How the controller-manager, scheduler, and proxy find the apiserver 
KUBE_MASTER="--master=http://192.168.56.100:8080"
```

```
[root@k8s-node-1 ~]# cat /etc/kubernetes/kubelet

# 启用日志标准错误
KUBE_LOGTOSTDERR="--logtostderr=true"

# 日志级别
KUBE_LOG_LEVEL="--v=0"

# Kubelet服务IP地址
NODE_ADDRESS="--address=192.168.56.101"

# Kubelet服务端口 
NODE_PORT="--port=10250"

# 自定义节点名称
NODE_HOSTNAME="--hostname-override=192.168.56.101"

# kubeconfig路径，指定连接API服务器
KUBELET_KUBECONFIG="--kubeconfig=/etc/kubernetes/kubelet.kubeconfig"

# 允许容器请求特权模式，默认false 
KUBE_ALLOW_PRIV="--allow-privileged=false"

# KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"

# DNS信息 
KUBELET_DNS_IP="--cluster-dns=10.254.0.2"
KUBELET_DNS_DOMAIN="--cluster-domain=cluster.local"

# 禁用使用 Swap 
KUBELET_SWAP="--fail-swap-on=false"
```

**5.3.1.1 kubelet.kubeconfig文件内容如下：**
```
[root@k8s-node-1 ~]# cat /etc/kubernetes/kubelet.kubeconfig

apiVersion: v1
kind: Config
clusters: 
  - cluster:
      server: http://192.168.56.100:8080
  - name: local
contexts: 
  - context: 
      cluster: local
  - name: local
current-context: local
```

**5.3.2 创建 kubelet 的 systemd unit(kubelet.service) 文件**
```
[root@k8s-node-1 ~]# cat /usr/lib/systemd/system/kubelet.service

[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet ${KUBE_LOGTOSTDERR} ${KUBE_LOG_LEVEL} ${NODE_ADDRESS} ${NODE_PORT} ${NODE_HOSTNAME} ${KUBELET_KUBECONFIG} ${KUBE_ALLOW_PRIV} ${KUBELET_DNS_IP} ${KUBELET_DNS_DOMAIN} ${KUBELET_SWAP}  --cgroup-driver=systemd
Restart=on-failure
KillMode=process

[Install]
WantedBy=multi-user.target
```

**5.3.3 启动 kubelet**
```
[root@k8s-node-1 ~]# systemctl daemon-reload
[root@k8s-node-1 ~]# systemctl enable kubelet.service
[root@k8s-node-1 ~]# systemctl start kubelet.service
[root@k8s-node-1 ~]# systemctl status kubelet.service
```

**5.4 安装和配置 kube-proxy**
**5.4.1 配置kube-proxy配置文件**
```
[root@k8s-node-1 ~]# cat /etc/kubernetes/proxy 

###
# kubernetes proxy config
# default config should be adequate
# Add your own!
KUBE_PROXY_ARGS=""
```

**5.4.2 新建参数配置文件/etc/kubernetes/config**
```
详见5.3.1
```

**5.4.3 创建 kube-proxy kubeconfig 文件**
```
[root@k8s-node-1 ~]# cat /usr/lib/systemd/system/kube-proxy.service

[Unit]
Description=Kubernetes Kube-Proxy Server 
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/bin/kube-proxy $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBE_MASTER $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**5.4.4 启动 kube-proxy**
```
[root@k8s-node-1 ~]# systemctl daemon-reload
[root@k8s-node-1 ~]# systemctl enable kube-proxy
[root@k8s-node-1 ~]# systemctl start kube-proxy
[root@k8s-node-1 ~]# systemctl status kube-proxy
```

**4.4 检查节点状态**
```
[root@k8s-master ~]# kubectl get nodes
NAME            STATUS    AGE
192.168.1.181   Ready     1m
```
**注：如果没有输出上述信息，需要安装Flannel(网络通信)**

## 第六章 部署 Flannel 网络
```
部署flannel网络有两种方式，可以使用rpm包安装，也可以使用二进制方式安装，本文演示两种安装方式：
```

**6.1 rpm包部署**

```
rpm -ivh flannel-0.7.1-4.el7.x86_64.rpm
or
yum -y install flannel
```

**6.1.1 配置flannel网络**

```
[root@k8s-master ~]# cat /etc/sysconfig/flanneld

# Flanneld configuration options  
# etcd url location.  Point this to the server where etcd runs 
FLANNEL_ETCD_ENDPOINTS="http://192.168.56.100:2379"
# etcd config key.  This is the configuration key that flannel queries 
# For address range assignment 
FLANNEL_ETCD_PREFIX="/k8s/network"
# Any additional options that you want to pass 
#FLANNEL_OPTIONS=""
```

**6.1.2 配置etcd中关于flannel的key**
```
[root@k8s-master ~]# etcdctl set /k8s/network/config '{"Network": "172.20.0.0/16"}'
{"Network": "172.20.0.0/16"}
[root@k8s-master ~]# etcdctl get /k8s/network/config
{"Network": "172.20.0.0/16"}
```

**6.1.3 启动**
**6.1.3.1 启动Flannel之后，需要依次重启docker、kubernete。**
**6.1.3.2 在master执行：**

```
[root@k8s-master ~]# systemctl daemon-reload
[root@k8s-master ~]# systemctl enable flanneld.service
[root@k8s-master ~]# systemctl start flanneld.service
[root@k8s-master ~]# service docker restart
[root@k8s-master ~]# systemctl restart kube-apiserver.service
[root@k8s-master ~]# systemctl restart kube-controller-manager.service
[root@k8s-master ~]# systemctl restart kube-scheduler.service
```

**6.1.3.3 在node上执行：**
```
[root@k8s-master ~]# systemctl daemon-reload
[root@k8s-node-1 ~]# systemctl enable flanneld.service
[root@k8s-node-1 ~]# systemctl start flanneld.service
[root@k8s-node-1 ~]# service docker restart
[root@k8s-node-1 ~]# systemctl restart kubelet.service
[root@k8s-node-1 ~]# systemctl restart kube-proxy.service
```
