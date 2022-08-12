## 1.什么是Pod？

<font color=red>Pod是在Kubernetes系统中创建、调度和管理的最小单位。</font>其他的大多数资源对象都是用于支撑和扩展`Pod`对象功能的,例如:

- 用于管控`Pod`运行的`StatefulSet`和`Deployment`等控制器对象，
- 用于暴露`Pod`应用的`Service`和`Ingress`对象。
- 为`Pod`提供存储的`PersistentVolume`存储资源对象。

## 2. Pod基本操作

### 2.1 创建

```bash
# nginx.pod 为资源配置清单
$ kubectl apply -f nginx-pod.yaml
```

### 2.2 查询

```bash
# 基本信息查询
$ kubectl get pod
NAME                                READY   STATUS              RESTARTS   AGE
test-nginx-pod                      0/1     ContainerCreating   0          2s
# 查询更多信息 -o wide
[root@master k8s]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
test-nginx-pod                      1/1     Running   0          52s   10.244.104.25    node2   <none>           <none>
```

### 2.3 更多详情

```bash
$ kubectl describe pod test-nginx-pod
Name:         test-nginx-pod
Namespace:    default
Priority:     0
Node:         node2/192.168.148.132
Start Time:   Thu, 11 Aug 2022 18:28:19 +0800
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 2840f0d0c113552022c24fd5b350c7f61d874f6fde75a8d6431e6a3317a4f832
              cni.projectcalico.org/podIP: 10.244.104.25/32
              cni.projectcalico.org/podIPs: 10.244.104.25/32
Status:       Running
IP:           10.244.104.25
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  104s  default-scheduler  Successfully assigned default/test-nginx-pod to node2
  Normal  Pulling    103s  kubelet            Pulling image "nginx"
  Normal  Pulled     101s  kubelet            Successfully pulled image "nginx" in 2.268676575s
  Normal  Created    101s  kubelet            Created container test-containers
  Normal  Started    101s  kubelet            Started container test-containers
```

### 2.4 删除

```bash
$ kubectl delete pod test-nginx-pod
pod "test-nginx-pod" deleted
```

## 3. Pod生命周期

### 3.1 周期示意图

`Pod`对象自从其创建开始至其终止退出的时间范围称为其生命周期，典型的Pod生命周期是这样的：

![](http://img.liuqh.icu/image-20220812145535125.png)

1. 用户定义一个`YAML`格式的清单文件，并将其`POST`到`API Server`。一旦发送成功，清单文件中的内容就被作为一条意图记录（`a record of intent`）——即“期望状态”（`desired` ）——持久化记录在集群存储中，
2. 然后`Pod`被调度到一个健康的、有充足资源的节点上。一旦完成调度，就会进入等待状态（`pending `），此时节点会下载镜像并启动容器。
3. 在所有资源就绪前，`Pod`的状态都保持在等待状态。一切就绪后，`Pod`进入运行状态（`running `）。
4. 在完成所有任务后，会停止并进入成功状态（`succeeded` ）。

### 3.2 状态说明

- **`Pending(等待)`：**`API Server`创建了`Pod`资源对象并已存入`etcd`中，但它尚未被调度完成，或者仍处于从仓库下载镜像的过程中。
- **`Running(运行)`：**   `Pod`已经被调度至某节点，并且所有容器都已经被kubelet创建完成。
- **`Succeeded（成功）`：**  `Pod`中的所有容器都已经成功终止并且不会被重启。
- **`Failed(失败)`：** 所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态或已经被系统终止。
- **`Unknown（未知）`：**   `API Server`无法正常获取到`Pod`对象的状态信息，通常是由于其无法与所在工作节点的`kubelet`通信所致。



## 4. Pod创建和终止

### 4.1 `Pod`创建过程

`Pod`是`Kubernetes`的基础单元，理解它的创建过程对于了解系统运作大有裨益。下图描述了一个`Pod`资源对象的典型创建过程。

![Pod创建过程](http://img.liuqh.icu/image-20220812145856169.png)

**创建流程描述:**

1. 用户通过`kubectl`或其他`API`客户端提交`Pod Spec`给`API Server`。
2. `API Server`尝试着将`Pod`对象的相关信息存入`etcd`中，待写入操作执行完成，`API Server`即会返回确认信息至客户端。
3. `API Server`开始反映`etcd`中的状态变化。
4. 所有的`Kubernetes`组件均使用`watch`机制来跟踪检查`API Server`上的相关的变动。
5. `kube-scheduler`（调度器）通过其`watcher`觉察到`API Server`创建了新的`Pod`对象但尚未绑定至任何工作节点。
6. `kube-scheduler`为`Pod`对象挑选一个工作节点并将结果信息更新至`API Server`。
7. 调度结果信息由`API Server`更新至`etcd`存储系统，而且`API Server`也开始反映此`Pod`对象的调度结果。
8. `Pod`所在工作节点上的`kubelet`尝试在调用`Docker`启动容器，并将容器的结果状态回送至`API Server`。
9. `API Server`将`Pod`状态信息存入`etcd`系统中。
10. 在`etcd`确认写入操作成功完成后，`API Server`将确认信息发送至相关的`kubelet`，事件将通过它被接受。

### 4.2 Pod的终止过程

`Pod`对象代表了在`Kubernetes`集群节点上运行的进程，它可能曾用于处理生产数据或向用户提供服务等，当`Pod`本身不再具有存在的价值时，如何将其优雅地终止呢?

![Pod终止过程](http://img.liuqh.icu/image-20220812153940745.png)

**终止流程描述:**

1. 用户发送删除`Pod`对象的命令。

2. `API Server`中的`Pod`对象会随着时间的推移而更新，在宽限期内（默认为30秒）`Pod`被视为`dead`。

3. 将`Pod`标记为`Terminating`状态。

   - `kubelet`在监控到`Pod`对象转为`Terminating`状态的同时,启动`Pod`关闭过程。

   - 端点控制器监控到`Pod`对象的关闭行为时,将其从所有匹配到此端点的`Service`资源的端点列表中移除。

4. 如果当前`Pod`对象定义了`preStop`钩子处理器，则在其标记为`terminating`后即会以同步的方式启动执行；如若宽限期结束后，`preStop`仍未执行结束，则第2步会被重新执行并额外获取一个时长为2秒的小宽限期。

5. `Pod`对象中的容器进程收到`TERM`信号。

6. 宽限期结束后，若存在任何一个仍在运行的进程，那么`Pod`对象即会收到`SIGKILL`信号。

9. `Kubelet`请求`API Server`将此`Pod`资源的宽限期设置为`0`从而完成删除操作，它变得对用户不再可见。

> 默认情况下，所有删除操作的宽限期都是30秒，不过，`kubectl delete`命令可以使用`--grace-period=<seconds>`选项自定义其时长，若使用0值则表示直接强制删除指定的资源，不过，此时需要同时为命令使用`--force`选项。

## 5. Pod主要配置说明

> <font color=red>虽然配置清单(yaml)中的字段有很多，但并不是所有的都要写，循序渐进，慢慢熟悉~</font>

### 5.1 配置清单

```yaml
apiVersion: v1  #必选，版本号，例如v1
kind: Pod       #必选，Pod
metadata:       #必选，元数据
  name: string    #必选，Pod名称
  namespace: string    #必选，Pod所属的命名空间
  labels:      #自定义标签
    - name: string     #自定义标签名字
  annotations:       #自定义注释列表
    - name: string
spec:         #必选，Pod中容器的详细定义
  containers:      #必选，Pod中容器列表
  - name: string     #必选，容器名称
    image: string    #必选，容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent] #获取镜像的策略
    ports:       #需要暴露的端口库号列表
     - name: string     #端口号名称
       containerPort: int   #容器需要监听的端口号
       hostPort: int    #容器所在主机需要监听的端口号，默认与Container相同
       protocol: string     #端口协议，支持TCP和UDP，默认TCP
    restartPolicy: [Always | Never | OnFailure] #Pod的重启策略
    ....
```

### 5.2 imagePullPolicy(镜像获取策略)

容器的`imagePullPolicy`字段用于为其指定镜像获取策略，它的可用值包括如下几个:

- `Always`:  镜像标签为`latest`或镜像不存在时总是从指定的仓库中获取镜像。
- `Never`: 禁止从仓库下载镜像，即仅使用本地镜像。
- `IfNotPresent`: 仅当本地镜像缺失时，才从目标仓库下载镜像

### 5.3 restartPolicy(Pod的重启策略)

- `Always`: 表示容器失效时，由`kubelet`自动重启该容器。
- `OnFailure`: 表示容器终止运行且退出码不为`0`时，`由kubelet`自动重启该容器。
- `Nerver`：表示不论容器运行状态如何，`kubelet`都不会重启该容器。








