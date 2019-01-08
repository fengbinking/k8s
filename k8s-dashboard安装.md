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
          - --apiserver-host=http://10.200.3.81:8080 
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

[root@localhost ~]# kubectl get pods --all-namespaces  
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE
default       my-nginx-379829228-mpm5f                1/1       Running   0          59m
kube-system   kubernetes-dashboard-3162153223-2s0qb   1/1       Running   0          40m
```
