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
