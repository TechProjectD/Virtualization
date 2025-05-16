## Horizontal Pod Autoscaler(HPA)

- 在前面的课程中，我们已经可以实现通过手工执行`kubectl scale`命令实现Pod扩容或缩容，但是这显然不符合Kubernetes的定位目标----自动化、智能化。 Kubernetes期望可以实现通过监测Pod的使用情况，实现pod数量的自动调整，于是就产生了Horizontal Pod Autoscaler（HPA）这种控制器。

- HPA可以获取每个Pod利用率，然后和HPA中定义的指标进行对比，同时计算出需要伸缩的具体值，最后实现Pod的数量的调整。其实HPA与之前的Deployment一样，也属于一种Kubernetes资源对象，它通过追踪分析RC控制的所有目标Pod的负载变化情况，来确定是否需要针对性地调整目标Pod的副本数，这是HPA的实现原理。

![img](../pics/image-20200608155858271.png)

- 接下来，我们来做一个实验

#####  安装metrics-server

- metrics-server可以用来收集集群中的资源使用情况

```shell
# 安装git
[root@k8s-master01 ~]# yum install git -y
# 获取metrics-server, 注意使用的版本
[root@k8s-master01 ~]# git clone -b metrics-server-helm-chart-3.8.2 https://github.com/kubernetes-incubator/metrics-server
# 下载metrics的配置文件
[root@k8s-master01 ~]# wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

```shell
修改metrics配置文件添加如下内容:
[root@k8s-master01 ~]#vim componets.yml
#添加 - --kubelet-insecure-tls 到下面位置:
spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s

#如果不添加以上内容会在描述中出现如下两种报错报错:
报错1:Readiness probe failed: HTTP probe failed with statuscode: 501
报错2:Readiness probe failed: HTTP probe failed with statuscode: 500

#修改metrics配置文件，添加 hostNetwork: true 到如下位置:
 nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      hostNetwork: true
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir

#如不添加会出现如下错误导致metrics的pod无法启动:
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  62s                default-scheduler  Successfully assigned kube-system/metrics-server-8698b98df9-5lt96 to linux6.skills.com
  Normal   Pulled     29s (x2 over 61s)  kubelet            Container image "k8s.gcr.io/metrics-server/metrics-server:v0.6.1" already present on machine
  Normal   Created    29s (x2 over 61s)  kubelet            Created container metrics-server
  Normal   Started    29s (x2 over 61s)  kubelet            Started container metrics-server
  Warning  Unhealthy  1s                 kubelet            Liveness probe failed: Get "https://10.10.1.26:4443/livez": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)


#查看日志会会出现如下错误:
[root@k8s-master01 ~]#kubectl logs -n kube-system metrics-server-6dfddc5fb8-f54vr
……
I0816 01:07:06.734107       1 server.go:188] "Failed probe" probe="metric-storage-ready" err="not metrics to serve"
I0816 01:07:16.736864       1 server.go:188] "Failed probe" probe="metric-storage-ready" err="not metrics to serve"
E0816 01:07:18.834625       1 scraper.go:139] "Failed to scrape node" err="Get \"https://192.168.130.100:10250/stats/summary?only_cpu_and_memory=true\": x509: cannot validate certificate for 192.168.130.100 because it doesn't contain any IP SANs" node="loki"
#具体原因是“cannot validate certificate for 192.168.130.100 because it doesn't contain any IP SANs”。
```

```shell
# 安装metrics-server
[root@k8s-master01 ~]# kubectl apply -f components.yml

# 查看pod运行情况
[root@k8s-master01 ~]# kubectl get pod -n kube-system
metrics-server-6b976979db-2xwbj   1/1     Running   0          90s

# 使用kubectl top node 查看资源使用情况
[root@k8s-master01 ~]# kubectl top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master01   289m         14%    1582Mi          54%
k8s-node01     81m          4%     1195Mi          40%
k8s-node02     72m          3%     1211Mi          41%
[root@kube-master01 ~]# kubectl top pod -n kube-system
NAME                                             CPU(cores)   MEMORY(bytes)
coredns-64897985d-5ndsv                          1m           20Mi
coredns-64897985d-n6tzb                          2m           21Mi
etcd-kube-master.skills.com                      14m          67Mi
kube-apiserver-kube-master.skills.com            49m          342Mi
kube-controller-manager-kube-master.skills.com   13m          62Mi
kube-flannel-ds-67vrw                            4m           35Mi
kube-flannel-ds-t9fqr                            4m           35Mi
kube-proxy-78g2t                                 1m           24Mi
kube-proxy-dhxk4                                 1m           24Mi
kube-proxy-hrdr7                                 1m           24Mi
kube-scheduler-kube-master.skills.com            3m           26Mi
metrics-server-7cf8b65d65-lbqx5                  4m           27Mi
...
```

> **至此metrics-server安装完成**


##### **准备deployment和servie**
- 创建pc-hpa-pod.yaml文件，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx 
spec:
  selector:
    matchLabels:
      run: nginx-pod 
  replicas: 1
  template:
    metadata:
      labels:
        run: nginx-pod 
    spec:
      containers:
      - name: nginx 
        image: docker.io/library/nginx:1.23.5
        ports:
        - containerPort: 80
        resources:  # 资源配额
          limits:  # 限制资源（上限）
            cpu: 500m #这里的500m表示0.5个核心，也可以叫做"five hundred millicores",换算方式是500除以1000，比如64m就是64/1000=0.064也就是6.4%,如果直接写整数比如"1"则表示1个核心 
          requests: # 请求资源（下限） 
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: nginx 
  labels:
    run: nginx-pod 
spec:
  ports:
  - port: 80
  selector:
    run: nginx-pod 
```

```shell
# 创建deployment
[root@k8s-master01 ~]# kubectl apply -f pc-hpa-pod.yaml

# 创建service
[root@k8s-master01 ~]# kubectl expose deployment nginx --type=NodePort --port=80 -n dev

[root@k8s-master01 ~]# kubectl get deployment,pod,svc
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           49m

NAME                              READY   STATUS    RESTARTS   AGE
pod/nginx-5cd8d6f579-zrvp2   1/1     Running   0          49m

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   21d
service/nginx        ClusterIP   10.102.81.95   <none>        80/TCP    49m
```

##### **部署HPA**
- 使用命令行方式创建HPA:

```shell
[root@k8s-master01 ~]# kubectl autoscale deployment pc-hpa --cpu-percent=50 --min=1 --max=10
#创建一个名为pc-hpa的HPA，设置CPU占用最多为50%，开启pod数量最少1个，最大为10个
```

- 创建pc-hpa.yaml文件，内容如下：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: pc-hpa
spec:
  scaleTargetRef: #指定要控制的nginx信息
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 1 #最小pod数量
  maxReplicas: 10 #最大pod数量
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50 #CPU使用率指标单位是百分比
```

```shell
# 创建hpa
[root@k8s-master01 ~]# kubectl create -f pc-hpa.yaml
horizontalpodautoscaler.autoscaling/pc-hpa created

# 查看hpa
[root@kube-master01 ~]# kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
pc-hpa     Deployment/nginx           0%/50%      1         10        1       45m
```

##### **测试**
- 使用下面的压力测试命令对Deployment进行压测,然后通过控制台查看hpa和pod的变化:
`ab -n 500000 -c 1000 http://10.102.81.95/`

- 观察hpa变化

```shell
[root@k8s-master01 ~]# kubectl get hpa -n dev -w
NAME   REFERENCE              TARGETS   MINPODS  MAXPODS  REPLICAS  AGE
pc-hpa  Deployment/nginx       0%/50%      1       10       1      4m11s
pc-hpa  Deployment/nginx       0%/50%      1       10       1      5m19s
pc-hpa  Deployment/php-apache  163%/50%    1       10       2      56m
```

- deployment变化

```shell
[root@k8s-master01 ~]# kubectl get deployment -n dev -w
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           11m
nginx   1/4     1            1           13m
nginx   1/4     1            1           13m
nginx   1/4     1            1           13m
nginx   1/4     4            1           13m
nginx   1/8     4            1           14m
nginx   1/8     4            1           14m
nginx   1/8     4            1           14m
nginx   1/8     8            1           14m
nginx   2/8     8            2           14m
nginx   3/8     8            3           14m
nginx   4/8     8            4           14m
nginx   5/8     8            5           14m
nginx   6/8     8            6           14m
nginx   7/8     8            7           14m
nginx   8/8     8            8           15m
nginx   8/1     8            8           20m
nginx   8/1     8            8           20m
nginx   1/1     1            1           20m
```

- pod变化

```shell
[root@k8s-master01 ~]# kubectl get pods -n dev -w
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7df9756ccc-bh8dr   1/1     Running   0          11m
nginx-7df9756ccc-cpgrv   0/1     Pending   0          0s
nginx-7df9756ccc-8zhwk   0/1     Pending   0          0s
nginx-7df9756ccc-rr9bn   0/1     Pending   0          0s
nginx-7df9756ccc-cpgrv   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-8zhwk   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-rr9bn   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-m9gsj   0/1     Pending             0          0s
nginx-7df9756ccc-g56qb   0/1     Pending             0          0s
nginx-7df9756ccc-sl9c6   0/1     Pending             0          0s
nginx-7df9756ccc-fgst7   0/1     Pending             0          0s
nginx-7df9756ccc-g56qb   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-m9gsj   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-sl9c6   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-fgst7   0/1     ContainerCreating   0          0s
nginx-7df9756ccc-8zhwk   1/1     Running             0          19s
nginx-7df9756ccc-rr9bn   1/1     Running             0          30s
nginx-7df9756ccc-m9gsj   1/1     Running             0          21s
nginx-7df9756ccc-cpgrv   1/1     Running             0          47s
nginx-7df9756ccc-sl9c6   1/1     Running             0          33s
nginx-7df9756ccc-g56qb   1/1     Running             0          48s
nginx-7df9756ccc-fgst7   1/1     Running             0          66s
nginx-7df9756ccc-fgst7   1/1     Terminating         0          6m50s
nginx-7df9756ccc-8zhwk   1/1     Terminating         0          7m5s
nginx-7df9756ccc-cpgrv   1/1     Terminating         0          7m5s
nginx-7df9756ccc-g56qb   1/1     Terminating         0          6m50s
nginx-7df9756ccc-rr9bn   1/1     Terminating         0          7m5s
nginx-7df9756ccc-m9gsj   1/1     Terminating         0          6m50s
nginx-7df9756ccc-sl9c6   1/1     Terminating         0          6m50s
```

- **总结**
-  伴随cpu占用的提升，HPA会根据设置的值在deployment中开启容器，当CPU占用下降时，HPA会关闭这些刚刚开启的容器