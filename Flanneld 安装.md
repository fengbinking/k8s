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
