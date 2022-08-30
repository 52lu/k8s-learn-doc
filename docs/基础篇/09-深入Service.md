## 1. 介绍

### 1.1 概念

`Service`是`Kubernetes`的核心资源类型之一，通常可看作微服务的一种实现。<u>事实上它是一种抽象：通过规则定义出由多个`Pod`对象组合而成的逻辑集合，以及访问这组`Pod`的策略。</u>`Service`关联`Pod`资源的规则要借助于**标签选择器**来完成。

### 1.2 为什么需要Service

通过前面的学习，我们已经知道服务是部署在`Pod`的容器中，然而`Pod`作为服务提供者存在以下几个问题：

- `Pod`是短暂的,它们随时会启动或者关闭。
- `Kubernetes`在`Pod`启动前会给已经调度到节点上的`Pod`分配`IP`地址;因此客户端不能提前知道 `Pod` 的` IP` 信息。
- 当做水平伸缩时,就意味会动态添加或减少`Pod`,而且每个 `Pod` 都有自己的 `IP`地址，客户端不可能做到实时更新维护这些信息。

> 为了解决上述问题，`Kubernetes`  提供了一种资源类型 一一服 务( `service`)

## 2.快速开始

`Service`从逻辑上代表了一组`Pod`，具体是哪些`Pod`则是由`label`来挑选的。`Service`有自己的`IP`，而且这个`IP`是不变的。客户端只需要访问`Service`的`IP`，`Kubernetes`则负责建立和维护`Service`与`Pod`的映射关系。无论后端`Pod`如何变化，对客户端不会有任何影响，因为`Service`没有变。

> 前面的几篇文章中，已经实践过`Service`的使用，这里在复习下。

### 2.1 创建Pod

在实际使用中，很少单独或者直接创建`Pod`，都是通过各种控制器来管理`Pod`的生命周期，常用的控制器是`Deployment`。

下面通过`Deployment`创建标签`labels: hello=v1 和labels: hello=v2`的`Pod`,具体资源配置清单（此处省略）。

```bash
# 查看创建的deployment
$ kubectl get deploy -o wide
NAME              READY  ...   CONTAINERS     IMAGES                             SELECTOR
hello-v1-deploy   1/1    ...  hello-v1-pod   docker.io/liuqinghui/gin-hello:v1   hello=v1
hello-v2-deploy   1/1    ...  hello-v2-pod   docker.io/liuqinghui/gin-hello:v2   hello=v2
# 查看pod的标签为hello的值
$ kubectl get pod -L hello
NAME                               READY   STATUS    RESTARTS   AGE     HELLO
hello-v1-deploy-7cc98c464f-4xghg   1/1     Running   0          5h43m   v1
hello-v2-deploy-9d9f777d4-zp8rc    1/1     Running   0          5h43m   v2
```

### 2.2 关联Pod&暴露服务

下面创建两个`Service`，每个`Service`对应不同的标签。具体的配置清单如下:

#### a. 配置Yaml

```yaml
apiVersion: v1 # 第一个Service
kind: Service
metadata:
  name: hello-svc-v1
spec:
  selector:
    hello: v1 # 选中标签为 hello=v1的pod 
  ports:
  - name: http
    port: 8080 # ClusterIP监听的端口
    targetPort: 80 # Pod监听的端口
---
apiVersion: v1   # 第二个Service
kind: Service
metadata:
  name: hello-svc-v2
spec:
  selector:
    hello: v2 # 选中标签为 hello=v2的pod 
  ports:
  - name: http
    port: 8081 # ClusterIP监听的端口
    targetPort: 80 # Pod监听的端口
```

#### b. 查看 & 访问 

```bash
# 查询服务
$ kubectl get svc
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
hello-svc-v1   ClusterIP   10.104.54.62     <none>        8080/TCP   6s
hello-svc-v2   ClusterIP   10.102.236.237   <none>        8081/TCP   6s
# 访问
$ curl 10.104.54.62:8080
{"hostName":"hello-v1-deploy-7cc98c464f-4xghg","time":"2022-08-17 10:56:06","version":"v1"}
# 访问
$ curl 10.102.236.237:8081
{"hostName":"hello-v2-deploy-9d9f777d4-zp8rc","time":"2022-08-17 10:56:14","version":"v2"}
```

## 3. Service操作Pod

### 3.1 如何匹配Pod?

`Service`与`Pod`之间是通过`Label`和`Label`筛选器（`selector`）松耦合在一起的。`Deployment`和`Pod`之间也是通过这种方式进行关联的。

<font color=red>比如下图中的`Service`筛选标签时，要求`Pod`同时包含标签`version=v1`和`name=go`</font>

![](http://img.liuqh.icu/image-20220819145011981.png)



### 3.2 如何连接Pod?

`Service`并不是和 `pod` 直接相连的,他们之间有一种资源一一`Endpoint`资源；每一个`Service`在被创建的时候，都会得到一个关联的`Endpoint`对象。整个`Endpoint`对象其实就是一个动态的列表，其中包含集群中所有的匹配`Service Label`筛选器的健康`Pod`。

> `Kubernetes`会不断地检查`Label`筛选器对应的`Pod`的列表,如果有新的能够匹配`Label`筛选器的`Pod`出现，它就会被加入`Endpoint`对象，而消失的`Pod`则会被剔除。**Endpoint对象始终是保持更新的**

```bash
# 查看service的Endpoint信息
$ kubectl describe service hello-svc-v1
Name:              hello-svc-v1
...
Selector:          hello=v1
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.104.54.62
IPs:               10.104.54.62
Port:              http  8080/TCP
TargetPort:        80/TCP
Endpoints:         10.244.166.173:80 #这里就是对应的Endpoint信息
...
```

## 4. 集群内部访问

### 4.1 通过`ClusterIP + Port`

```bash
# 查看集群信息
$ kubectl get svc
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
hello-svc-v1   ClusterIP   10.104.54.62     <none>        8080/TCP   46h
# 在集群内创建测试Pod，并访问
$ kubectl run busybox-default -it --rm --image=busybox sh
/ $ wget 10.104.54.62:8080
Connecting to 10.104.54.62:8080 (10.104.54.62:8080)
saving to 'index.html'
index.html           100% |**********************************************************|    91  0:00:00 ETA
'index.html' saved
/ $ cat index.html
{"hostName":"hello-v1-deploy-7cc98c464f-46qx2","time":"2022-08-19 09:12:09","version":"v1"}
```

### 4.2 通过DNS访问

`kubeadm`部署时会默认安装`kube-dns`组件,`kube-dns`是一个`DNS`服务器。每当有新的`Service`被创建，`kube-dns`会添加该`Service`的`DNS`记录。<font color=red>集群中的`Pod`可以通过`ServiceName.NamespaceName:Port`访问`Service`。</font>

```bash
# ---- 访问,携带命名空间
/ $ wget hello-svc-v1.default:8080
Connecting to hello-svc-v1.default:8080 (10.104.54.62:8080)
saving to 'index.html'
index.html           100% |**********************************************|    91  0:00:00 ETA
'index.html' saved
/ # cat index.html
{"hostName":"hello-v1-deploy-7cc98c464f-46qx2","time":"2022-08-19 09:24:17","version":"v1"}

# ----- 省略命名空间 default
/ $ wget hello-svc-v1:8080
Connecting to hello-svc-v1:8080 (10.104.54.62:8080)
saving to 'index.html'
index.html           100% |**********************************************|    91  0:00:00 ETA
'index.html' saved
/ $ cat index.html
{"hostName":"hello-v1-deploy-7cc98c464f-46qx2","time":"2022-08-19 09:26:08","version":"v1"}
```

> <font color=red>当Pod和Service同处于一个命名空间下时,可以省略命名空间</font>


## 5. 集群外部访问

### 5.1 服务分类

除了`Cluster`内部可以访问`Service`，很多情况下我们也希望应用的`Service`能够暴露给`Cluster`外部。`Kubernetes`提供了多种类型的`Service`，默认是`ClusterIP`。

- `ClusterIP`: 默认的类型，这种`Service`面向集群内部有固定的`IP`地址。但是在集群外是不可访问的。

- `NodePort`: `Service`通过`Cluster`节点的静态端口对外提供服务。`Cluster`外部可以通过`<NodeIP>:<NodePort>`访问`Service`。
- `LoadBalance`: 能够与诸如`AWS、Azure、DO、IBM`云和`GCP`等云服务商提供的负载均衡服务集成。它基于`NodePort Service`（它又基于`ClusterIP Service`）实现，并在此基础上允许互联网上的客户端能够通过云平台的负载均衡服务到达`Pod`。

### 5.2 发布NodePort类型服务

#### a. 编辑配置文件

文件:	`hello-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc-nodeport
spec:
  type: NodePort
  selector:
    hello: v1 # 选中标签为 hello=v1的pod
  ports:
  - name: http
    port: 8080 # ClusterIP监听的端口
    targetPort: 80 # Pod监听的端口
    nodePort: 30001 # 端口范围在 30000～3276
```

- `type`: 需要设置成`NodePort`
- `nodePort`: 指定当前节点暴露的端口号，端口范围: `30000～3276`,不设置时,`Kubernetes`会在端口范围内随机生成。

#### b. 发布&访问

```bash
# 发布
$ kubectl apply -f hello-svc.yaml
service/hello-svc-v1 created
# 查看发布结果
$ kubectl get svc
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-svc-nodeport   NodePort    10.108.65.192   <none>        8080:30001/TCP   4s
# 访问,192.168.148.130为master IP
$ curl 192.168.148.130:30001
{"hostName":"hello-v1-deploy-7cc98c464f-46qx2","time":"2022-08-24 04:14:34","version":"v1"}
# 访问,192.168.148.130为node IP
$ curl 192.168.148.131:30001
{"hostName":"hello-v1-deploy-7cc98c464f-46qx2","time":"2022-08-24 04:16:40","version":"v1"}
```



## 6. Ingress资源

> `Kubernetes`提供了两种内建的云端负载均衡机制用于发布公共应用：
>
> - 一种是工作于传输层的`Service`资源，它实现的是`TCP负载均衡器`，
> - 一种是`Ingress`资源，它实现的是`HTTP(S)负载均衡器`。

### 6.1 为什么需要Ingress?

一个重要的原因是每个 `LoadBalancer `服务都需要自己的负载均衡器，以及独有的公有`IP` 地址，<font color=red>而`Ingress`只需要一个公网`IP` 就能为许 多 服务提供访问 。</font>

当客户端向`Ingress`发送`HTTP`请求 时，`Ingress` 会根据请求的主机名和路径决定请求转发到对应的服务，如下图所示:

![](http://img.liuqh.icu/image-20220825142205999.png)



### 6.2  Ingress和Ingress Controller

`Ingress`是`Kubernetes API`的标准资源类型之一，它其实就是一组基于`DNS`名称（`host`）或`URL`路径把请求转发至指定的`Service`资源的规则，用于将集群外部的请求流量转发至集群内部完成服务发布。

然而，`Ingress`资源自身并不能进行“流量穿透”，它仅是一组路由规则的集合，这些规则要想真正发挥作用还需要其他功能的辅助，如监听某套接字，然后根据这些规则的匹配机制路由请求流量。这种能够为`Ingress`资源监听套接字并转发流量的组件称为**Ingress控制器（Ingress Controller）**。



> <font color=blue>@注意:</font> 只有`Ingress Controller`在 集群中运行的前提下，`Ingress`资源才能正常工作,`Ingress Controller`不同于`Deployment`控制器等，`Ingress控制器`并不直接运行为`kube-controller-manager`的一部分，它是`Kubernetes`集群的一个重要附件，类似于`CoreDNS`，**需要在集群上单独部署。**

### 6.3 安装Nginx Ingress

> 下面通过`helm`来安装`Nginx Ingress`,至于如何安装`helm`,可参考[如何安装helm https://helm.sh/zh/docs/intro/install/](https://helm.sh/zh/docs/intro/install/)

#### a. 官方helm仓库

```bash
# 添加ingress-nginx官方helm仓库
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
"ingress-nginx" has been added to your repositories
# 更新
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "kubernetes-dashboard" chart repository
...Successfully got an update from the "ingress-nginx" chart repository
Update Complete. ⎈Happy Helming!⎈
```

#### b.下载 helm chart

```bash
# 第一种: 尝试直接下载，报错,估计是被墙的原因
$ helm pull ingress-nginx/ingress-nginx
Error: Get "https://github.com/kubernetes/ingress-nginx/releases/download/helm-chart-4.2.3/ingress-nginx-4.2.3.tgz": unexpected EOF
# 第二种: 直接下载包
$ wget https://github.com/kubernetes/ingress-nginx/releases/download/helm-chart-4.2.3/ingress-nginx-4.2.3.tgz
```

#### c. 编写配置文件

文件`ingress-nginx-chart.yaml`

```yaml
controller:
  ingressClassResource:
    name: nginx
    enabled: true
    default: true
    controllerValue: "k8s.io/ingress-nginx"
  admissionWebhooks:
    enabled: false
  replicaCount: 1 #副本数量
  image:
    registry: docker.io
    image: unreachableg/k8s.gcr.io_ingress-nginx_controller
    tag: "v1.2.0"
    digest: sha256:314435f9465a7b2973e3aa4f2edad7465cc7bcdc8304be5d146d70e4da136e51
  hostNetwork: true #使用宿主机网络
  nodeSelector:
    node-role.kubernetes.io/edge: ''
  affinity:
    podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - nginx-ingress
            - key: component
              operator: In
              values:
              - controller
          topologyKey: kubernetes.io/hostname
  tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: PreferNoSchedule
```

#### d. 安装

```bash
# 安装在命令空间ingress-nginx下，不存在该空间，则直接创建
$ helm install ingress-nginx ingress-nginx-4.2.3.tgz --create-namespace -n ingress-nginx -f ingress-nginx-chart.yaml

[root@master kubernetes]$ helm install ingress-nginx ingress-nginx-4.2.3.tgz --create-namespace -n ingress-nginx -f install-ingress.yaml
NAME: ingress-nginx
LAST DEPLOYED: Sun Jul 24 00:01:37 2022
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace ingress-nginx get services -o wide -w ingress-nginx-controller'

...

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

### 6.4 创建Ingress资源

#### a. 查看现有服务

```bash
# 可以看到下面有两个service
$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
hello-svc1   ClusterIP   10.110.245.196   <none>        8080/TCP   99m
hello-svc2   ClusterIP   10.108.2.174     <none>        8080/TCP   99m
```

#### b. 编辑配置文件

```yaml
apiVersion: networking.k8s.io/v1      
kind: Ingress        
metadata:          
  name: hello-ingress
  annotations:
    kubernetes.io/ingress.class: nginx # 不设置会404
    nginx.ingress.kubernetes.io/rewrite-target: /  # 不设置会404
spec: 
  rules: 
  - host: hui.k8s.com # 不指定host时，访问ingress-nginx所在服务器IP
    http:
      paths:
      - path: /v1 
        pathType: Prefix
        backend:
          service:
            name: hello-svc1 #匹配v1服务
            port:
              number: 8080
      - path: /v2 
        pathType: Prefix
        backend:
          service:
            name: hello-svc2 #匹配v2服务
            port:
              number: 8080
```

<font color=red><b>@注意: 一定要设置上面的annotations信息，否则请求时报404</b></font>

#### c. 发布资源

```bash
$ kubectl apply -f ingress-hello.yaml
ingress.networking.k8s.io/hello-ingress created
```

#### d. 设置host

```bash
# 查看ingress-nginx的Pod
$ kubectl get pod -A
NAMESPACE     NAME                                      READY   STATUS    ...
...
ingress-nginx ingress-nginx-controller-9b7f9c4b5-78pkt   1/1     Running   ...
...
# 查看详情找到IP为( 192.168.148.131)
$ kubectl describe pod ingress-nginx-controller-9b7f9c4b5-78pkt --namespace=ingress-nginx
Name:         ingress-nginx-controller-9b7f9c4b5-78pkt
Namespace:    ingress-nginx
Priority:     0
Node:         node1/192.168.148.131
...
Status:       Running
IP:           192.168.148.131
...
# 设置host,新增下面记录
$ sudo vim /etc/hosts
192.168.148.131 hui.k8s.com
```

#### e. 浏览器访问

![](http://img.liuqh.icu/image-20220830182558290.png)

