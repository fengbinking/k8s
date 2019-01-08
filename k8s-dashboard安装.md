# Centos7 节点上安装kubernetes-dashboard过程

**下载kubernetes-dashboard.yaml文件，修改一下即可；因为我使用的是公网的docker镜像，所以先下载dashborad的镜像到本地仓库.**
```
[root@localhost ~]#docker pull docker.io/siriuszg/kubernetes-dashboard-amd64:v1.5.1
```

**创建kubernetes-dashboard.yaml，文件如下：**
```
[root@localhost ~]# cat kubernetes-dashboard.yaml
```
```
kind: Deployment 
apiVersion: extensions/v1beta1 
metadata: 
  labels: 
    app: kubernetes-dashboard 
  name: kubernetes-dashboard 
  namespace: kube-system 
spec: 
  replicas: 1 
  selector: 
    matchLabels: 
      app: kubernetes-dashboard 
  template: 
    metadata: 
      labels: 
        app: kubernetes-dashboard 
      # Comment the following annotation if Dashboard must not be deployed on master 
      annotations: 
        scheduler.alpha.kubernetes.io/tolerations: | 
          [ 
            { 
              "key": "dedicated", 
              "operator": "Equal", 
              "value": "master", 
              "effect": "NoSchedule" 
            } 
          ] 
    spec: 
      containers: 
      - name: kubernetes-dashboard 
        image: docker.io/siriuszg/kubernetes-dashboard-amd64:v1.5.1 
        imagePullPolicy: Always 
        ports: 
        - containerPort: 9090 
          protocol: TCP 
        args: 
          # Uncomment the following line to manually specify Kubernetes API server Host 
          # If not specified, Dashboard will attempt to auto discover the API server and connect 
          # to it. Uncomment only if the default does not work. 
          - --apiserver-host=http://192.168.56.100:8080
        livenessProbe: 
          httpGet: 
            path: / 
            port: 9090 
          initialDelaySeconds: 30 
          timeoutSeconds: 30 
--- 
kind: Service 
apiVersion: v1 
metadata: 
  labels: 
    app: kubernetes-dashboard 
  name: kubernetes-dashboard 
  namespace: kube-system 
spec: 
  type: NodePort 
  ports: 
  - port: 80 
    targetPort: 9090 
  selector: 
    app: kubernetes-dashboard
```

**创建实例：**
```
# kubectl create -f kubernetes-dashboard.yaml 
```

**查看是否成功运行：**
```
[root@localhost ~]# kubectl get pods --all-namespaces  
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE
default       my-nginx-379829228-mpm5f                1/1       Running   0          59m
kube-system   kubernetes-dashboard-3162153223-2s0qb   1/1       Running   0          40m
```
**注：如果kubernetes-dashboard状态不是Running**
**使用如下命令排查问题**
```
[root@localhost k8s]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                    READY     STATUS             RESTARTS   AGE
kube-system   kubernetes-dashboard-846c9cc59c-6grt2   0/1       CrashLoopBackOff   9          28m

[root@localhost k8s]# kubectl describe pods --namespace=kube-system kubernetes-dashboard-846c9cc59c-6grt2
RunPodSandbox from runtime service failed: rpc error: code = Unknown desc = failed pulling image "gcr.io/google_containers/pause-amd64:3.0": Get https://gcr.io/v1/_ping: dial tcp 64.233.189.82:443: i/o timeout

[root@localhost k8s]# docker pull googlecontainer/pause-amd64:3.0
[root@localhost k8s]# docker tag googlecontainer/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0

[root@localhost k8s]# kubectl delete -f kubernetes-dashboard.yaml 

[root@localhost k8s]# kubectl create -f kubernetes-dashboard.yaml 

[root@localhost k8s]# kubectl logs -f kubernetes-dashboard-846c9cc59c-6grt2 -n kube-system
Error while initializing connection to Kubernetes apiserver. This most likely means that the cluster is misconfigured (e.g., it has invalid apiserver certificates or service accounts configuration) or the --apiserver-host param points to a server that does not exist. Reason: Get http://10.200.3.81:8080/version: dial tcp 10.200.3.81:8080: i/o timeout
Refer to the troubleshooting guide for more information: https://github.com/kubernetes/dashboard/blob/master/docs/user-guide/troubleshooting.md

[root@localhost ~]# kubectl get pods --all-namespaces  
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE
default       my-nginx-379829228-mpm5f                1/1       Running   0          59m
kube-system   kubernetes-dashboard-3162153223-2s0qb   1/1       Running   0          40m
```


**查看dashboard访问地址**
```
[root@localhost k8s]# kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                   READY     STATUS    RESTARTS   AGE       IP            NODE             NOMINATED NODE
kube-system   kubernetes-dashboard-8ff88dd47-n644g   1/1       Running   0          5m        172.20.81.2   192.168.56.101   <none>
# 发现dashboard部署到了192.168.56.101这个节点上。
# 接着，查看dashboard的集群内部IP及端口

[root@localhost k8s]# kubectl get services --all-namespaces
NAMESPACE     NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
default       kubernetes             ClusterIP   192.168.1.1     <none>        443/TCP        12d
kube-system   kubernetes-dashboard   NodePort    192.168.1.119   <none>        80:30904/TCP   5m
```
```
  由上面命令可以看出kubernetes-dashboard的访问地址为192.168.56.101:30904
```
