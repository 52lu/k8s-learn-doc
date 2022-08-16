## 1. 介绍

强大的自愈能力是`Kubernetes`这类容器编排引擎的一个重要特性。自愈的默认实现方式是自动重启发生故障的容器。除此之外，用户还可以利用`Liveness`和`Readiness`探测机制设置更精细的健康检查，进而实现如下需求:

- 零停机部署。
- 避免部署无效的镜像。
- 更加安全的滚动升级。

## 2. 默认健康检查

在[K8s学习(七)：深入了解Pod](https://mp.weixin.qq.com/s/-Rmv8CnU-3fIgA-RTbvHXw)文章中，简单介绍过的`Pod`的重启策略，便是`Kubernetes`默认的健康检查机制，如果进程退出时返回码非零，则认为容器发生故障，`Kubernetes`就会根据`restartPolicy`重启容器,具体重启策略如下:

- `Always`: 表示容器失效时，由`kubelet`自动重启该容器。
- `OnFailure`: 表示容器终止运行且退出码不为`0`时，`由kubelet`自动重启该容器。
- `Nerver`：表示不论容器运行状态如何，`kubelet`都不会重启该容器。



**<font color=blue>下面示例是`Pod`运行10秒后，以退出码非0的方式退出， 来模拟`Pod`发生故障，看看默认机制怎么处理</font>**

### 2.1  编写`Yaml`

下面以`nginx pod`为示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: test
spec:
  restartPolicy: OnFailure # 设定重启策略
  containers:
    - name: health-check
      image: nginx
      args:
       - /bin/sh
       - -c
       - sleep 10; exit 1 # 模拟容器启动10秒后发生故障（退出码不为0时）
```

### 2.2 创建 & 检查

```shell
# 为了测试创建命名空间
$ kubectl create namespace test
namespace/test created
# 创建Pod
$ kubectl apply -f nginx-pod.yaml
pod/nginx-pod created
# 查看Pod 状态,发现Pod已经重启(RESTARTS)了2次
$ kubectl get pod -n test -o wide
NAME        READY   STATUS   RESTARTS      AGE   ...
nginx-pod   0/1     Error    2 (36s ago)   64s   ...
```

### 2.3 总结

在上面的例子中，<u>容器进程返回值非零</u>，`Kubernetes`则认为容器发生故障，需要重启。<font color=red>有不少情况是发生了故障，但进程并不会退出。</font>

> 比如访问`Web`服务器时显示`500`内部错误，可能是系统超载，也可能是资源死锁，此时`httpd`进程并没有异常退出，在这种情况下重启容器可能是最直接、最有效的解决方案，那我们如何利用`Health Check`机制来处理这类场景呢? **<font color=blue>答案：就是存活探针。</font>**

## 3. 存活探针

`Kubernetes`对`Pod`的健康状态除了通过默认机制检查，还可以通过两类探针来检查： `LivenessProbe`和`ReadinessProbe`，`kubelet`定期执行这两类探针来诊断容器的健康状况。

### 3.1 LivenessProbe

> **用于判断容器是否存活（`Running`状态）**

- 如果`LivenessProbe`探针探测到容器不健康，则`kubelet`将杀掉该容器，并根据容器的重启策略做相应的处理。
- 如果一个容器不包含`LivenessProbe`探针，那么`kubelet`认为该容器的`LivenessProbe`探针返回的值永远是`Success`。

### 3.2 ReadinessProbe

> **用于判断容器服务是否可用（`Ready`状态），达到`Ready`状态的`Pod`才可以接收请求。**

对于被`Service`管理的`Pod`，`Service`与`Pod Endpoint`的关联关系也将基于`Pod`是否`Ready`进行设置。

- 如果在运行过程中`Ready`状态变为`False`，则系统自动将其从`Service`的后端`Endpoint`列表中隔离出去>

- 如果在运行过程中`Ready`状态变为`True`，就会把`Pod`加回后端`Endpoint`列表。

**<font color=red>这样就能保证客户端在访问Service时不会被转发到服务不可用的Pod实例上。</font>**

### 3.3 实现方式

`LivenessProbe`和`ReadinessProbe`均可配置以下三种实现方式。

- `Exec：在容器内部执行一个命令，如果该命令的返回码为`0`，则表明容器健康。
- `TCPSocket`：通过容器的`IP`地址和端口号执行`TCP`检查，如果能够建立`TCP`连接，则表明容器健康。
- `HTTPGet`：通过容器的`IP`地址、端口号及路径调用`HTTP Get`方法，如果响应的状态码大于等于`200`且小于`400`，则认为容器健康。

## 4. LivenessProbe实践

> `Liveness`探测让用户可以自定义判断容器是否健康的条件。如果探测失败，`Kubernetes`就会重启容器。

### 4.1 Exec 方式

#### a. 编辑`yaml`

**文件:`gin-hello-deploy.yaml`**

```yaml
apiVersion: apps/v1 # 配置格式版本
kind: Deployment   #资源类型，Deployment
metadata:
  name: gin-hello-deploy #Deployment 的名称
spec:
  replicas: 3   # 副本数量
  selector:   #标签选择器，
    matchLabels: 
      app: gin_hello_pod
  template:    #Pod信息
    metadata:
      labels:  #Pod的标签
        app: gin_hello_pod
    spec:
      containers:
      - name: gin-hello #pod的名称
        image: docker.io/liuqinghui/gin-hello:v3
        args:
         - /bin/sh
         - -c
         - touch /tmp/health; sleep 30; rm -rf /tmp/health;sleep 600 # 指定命令
        livenessProbe:
         exec:
          command:
           - cat 
           - /tmp/health
         initialDelaySeconds: 10 #容器启动10s之后,开始执行Liveness探测
         periodSeconds: 5 #每5秒执行一次Liveness探测
```

> 启动进程首先创建文件`/tmp/health`，30秒后删除，在我们的设定中，如果`/tmp/health`文件存在，则认为容器处于正常状态，反之则发生故障。

**相关字段说明:**

- `initialDelaySeconds`：10;  指定容器启动10秒之后开始执行`Liveness`探测，我们一般会根据应用启动的准备时间来设置。比如某个应用正常启动要花30秒，那么`initialDelaySeconds`的值就应该大于30。
- `periodSeconds`：5;   指定每5秒执行一次`Liveness`探测。`Kubernetes`如果连续执行3次`Liveness`探测均失败，则会杀掉并重启容器。

#### b.创建 & 观察

```bash
# 创建资源
$ kubectl apply -f gin-hello-deploy.yaml
# 查看是否重启
$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS     AGE   ...
gin-hello-deploy-7ffb6bbdf9-fm7h7   1/1     Running   1 (5s ago)   81s   ...
gin-hello-deploy-7ffb6bbdf9-wz4t2   1/1     Running   1 (5s ago)   81s   ...
gin-hello-deploy-7ffb6bbdf9-xrjjk   1/1     Running   1 (5s ago)   81s   ...
```

### 4.2  HTTPGet 方式

基于`HTTP`的探测（`HTTPGetAction`）向目标容器发起一个`HTTP`请求，根据其响应码进行结果判定，响应码形如`2xx`或`3xx`时表示检测通过.

<font color=blue> 下面通过`go`项目，实现随机出现系统错误500，来验证</font>

#### a. 修改`go`代码

```go
package main

import (
	"github.com/gin-gonic/gin"
	"math/rand"
	"os"
	"time"
)

func main() {
	engine := gin.Default()
	engine.GET("/", func(context *gin.Context) {
		// 显示主机名字
		hostName, _ := os.Hostname()
		// 正常响应
		context.JSON(200, gin.H{
			"version":  "v4",
			"hostName": hostName,
			"time":     time.Now().Format("2006-01-02 15:04:05"),
		})
	})
	// 健康检查
	engine.GET("/health", func(context *gin.Context) {
		// 模拟服务故障
		rand.Seed(time.Now().Unix())
		if rand.Intn(10) > 3 {
			panic("模拟服务故障~")
		}
		// 响应
		context.JSON(200, gin.H{})
	})
	_ = engine.Run(":8081")
}

```

打包成镜像`docker.io/liuqinghui/gin-hello:v4`

#### b. 编辑yaml

文件:`http-livenessprobe-deploy.yaml`

```yaml
apiVersion: apps/v1 # 配置格式版本
kind: Deployment   #资源类型，Deployment
metadata:
  name: http-liveness-deploy #Deployment 的名称
spec:
  replicas: 3   # 副本数量
  selector:   #标签选择器，
    matchLabels: 
      app: http-liveness-pod
  template:    #Pod信息
    metadata:
      labels:  #Pod的标签
        app: http-liveness-pod
    spec:
      containers:
      - name: http-liveness #pod的名称
        imagePullPolicy: Always
        image: docker.io/liuqinghui/gin-hello:v4
        livenessProbe: # 设置http探针
          httpGet:
            path: /health
            port: 8081
```

#### c. 创建 & 观察

```bash
# 创建资源
$ kubectl apply -f http-livenessprobe-deploy.yaml
# 查看某一个pod日志
$ kubectl logs -f http-liveness-deploy-558c8b48cf-28d95
[GIN] 2022/08/16 - 03:11:19 | 200 |       82.66µs | 192.168.148.131 | GET      "/health"
[GIN] 2022/08/16 - 03:11:29 | 500 |     178.466µs | 192.168.148.131 | GET      "/health"


2022/08/16 03:11:29 [Recovery] 2022/08/16 - 03:11:29 panic recovered:
GET /health HTTP/1.1
Host: 10.244.166.155:8081
Connection: close
Accept: */*
Connection: close
User-Agent: kube-probe/1.24


模拟服务故障~
/workspace/main.go:27 (0x73688d)
/go/pkg/mod/github.com/gin-gonic/gin@v1.8.1/context.go:173 (0x730a61)
/go/pkg/mod/github.com/gin-gonic/gin@v1.8.1/recovery.go:101 (0x730a4c)
/go/pkg/mod/github.com/gin-gonic/gin@v1.8.1/context.go:173 (0x72fb46)
/go/pkg/mod/github.com/gin-gonic/gin@v1.8.1/logger.go:240 (0x72fb29)
/go/pkg/mod/github.com/gin-gonic/gin@v1.8.1/context.go:173 (0x72ec10)
/go/pkg/mod/github.com/gin-gonic/gin@v1.8.1/gin.go:616 (0x72e878)
/go/pkg/mod/github.com/gin-gonic/gin@v1.8.1/gin.go:572 (0x72e53c)
/usr/local/go/src/net/http/server.go:2916 (0x61801a)
/usr/local/go/src/net/http/server.go:1966 (0x6145b6)
/usr/local/go/src/runtime/asm_amd64.s:1571 (0x465080)
# 查看重启次数
$ kubectl get pod -o wide
NAME                                    READY   STATUS    RESTARTS      ...
http-liveness-deploy-558c8b48cf-28d95   1/1     Running   1 (30s ago)   ...
http-liveness-deploy-558c8b48cf-fqzcd   1/1     Running   1 (30s ago)   ...
http-liveness-deploy-558c8b48cf-wwvbp   1/1     Running   1 (30s ago)   ...
```

## 5. ReadinessProbe实践

除了`Liveness`探测，`Kubernetes Health Check`机制还包括`Readiness`探测。用户通过`Liveness`探测可以告诉`Kubernetes`什么时候通过重启容器实现自愈；**<font color=red>`Readiness`探测则是告诉`Kubernetes`什么时候可以将容器加入到`Service`负载均衡池中，对外提供服务。</font>**

> **<font color=blue>Readiness探测的配置语法与Liveness探测完全一样,只需要把`livenessProbe`换成`readinessProbe`</font>**

因为使用方法和`Liveness`一样，这里只实践命令行(`Exec`)方式。

### 5.1 编写`yaml`

文件:`readiness-deploy.yaml`

```yaml
apiVersion: apps/v1 # 配置格式版本
kind: Deployment   #资源类型，Deployment
metadata:
  name: readiness-deploy #Deployment 的名称
spec:
  replicas: 1   # 副本数量
  selector:   #标签选择器，
    matchLabels:
      app: gin_readiness_pod
  template:    #Pod信息
    metadata:
      labels:  #Pod的标签
        app: gin_readiness_pod
    spec:
      containers:
      - name: readiness-pod #pod的名称
        image: docker.io/liuqinghui/gin-hello:v3
        args:
         - /bin/sh
         - -c
         - touch /tmp/health; sleep 30; rm -rf /tmp/health;sleep 600 # 指定命令
        readinessProbe:  # 把livenessProbe换成readinessProbe
         exec:
          command:
           - cat
           - /tmp/health
         initialDelaySeconds: 10 #容器启动10s之后,开始执行Readiness探测
         periodSeconds: 5 #每5秒执行一次Rreadiness探测
```

### 5.2 创建 & 观察

```bash
# 创建 & 观察 -w:代表实时观察pod状态变更
$ kubectl apply -f readiness-deploy.yaml & kubectl get pod -o wide -w
deployment.apps/readiness-deploy created
NAME                               READY   STATUS    RESTARTS    AGE   
...
# READY：不可用
readiness-deploy-c6cb95448-7t9lz   0/1     Running     0          2s   ...
# READY：可用
readiness-deploy-c6cb95448-7t9lz   1/1     Running     0          15s  ...
# READY：不可用
readiness-deploy-c6cb95448-7t9lz   0/1     Running     0          45s  ...

# 查看deploy状态
$ kubectl get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
readiness-deploy   0/1     1            0           7m15s

# 查看Pod日志
$ kubectl describe pod readiness-deploy-c6cb95448-ltkx9
Name:         readiness-deploy-c6cb95448-ltkx9
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  ...
  # 这里看到执行readiness探针时报错
  Warning  Unhealthy  1s (x15 over 66s)  kubelet            Readiness probe failed: cat: can't open '/tmp/health': No such file or directory 
```

### 5.3 Ready状态经历

`Pod readiness`的`READY`状态经历了如下变化：

- 刚被创建时，`READY`状态为不可用。
- 15秒后（`initialDelaySeconds + periodSeconds`），第一次进行`Readiness`探测并成功返回，设置`READY`为可用。
- 30秒后，`/tmp/healthy`被删除，连续3次`Readiness`探测均失败后，`READY`被设置为不可用。

## 6. Liveness和Readiness 对比

- `Liveness`探测和`Readiness`探测是两种`Health Check`机制，如果不特意配置，`Kubernetes`将对两种探测采取相同的默认行为，即通过判断容器启动进程的返回值是否为零来判断探测是否成功.

- 两种探测的配置方法完全一样，支持的配置参数也一样。**<font color=red>不同之处在于探测失败后的行为</font>**

  - `Liveness`探测: 重启容器；

  - `Readiness`探测: 将容器设置为不可用，不接收`Service`转发的请求。

- `Liveness`探测和`Readiness`探测是独立执行的，二者之间没有依赖，所以可以单独使用，也可以同时使用。
  - 用`Liveness`探测判断容器是否需要重启以实现自愈；
  - 用`Readiness`探测判断容器是否已经准备好对外提供服务。

