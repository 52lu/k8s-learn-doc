## 1.Secret介绍

应用启动过程中可能需要一些敏感信息，比如访问数据库的用户名、密码或者密钥。将这些信息直接保存在容器镜像中显然不妥，`Kubernetes`提供的解决方案是`Secret`。

> `Secret`会以密文的方式存储数据，避免了直接在配置文件中保存敏感信息。

## 2. 创建Secret

### 2.1 通过`env`文件

> <font color=red>语法: `kubectl create secret generic name --from-env-file=env文件 `</font>

#### a. 编辑`env`文件

文件:`.env`

```properties
username=admin
password=123456
```

#### b. 创建secret

```bash
# secret 名称为: mysql-secret
$ kubectl create secret generic mysql-secret --from-env-file=./.env
```

### 2.2 通过`yaml`文件

#### a. 编辑文件

文件: `redis_secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
data:
  ip: MTI3LjAuMC4x
  port: NjM3OQ==
  user: dGVzdA==
  pass: MTEyMjMz
```

> <font color=red>通过配置文件生成Secret时，val(值)必须通过base64编码，否则会报错。</font>

#### b. 创建secret

```bash
$ kubectl apply -f redis-secret.yaml
secret/redis-secret created
```

## 3. 查看Secret

### 3.1 获取Secret列表

使用`kubectl get secret`可以查看所有的`Secret`

```bash
# Data数量代表有几个键值对
$ kubectl get secret
NAME                               TYPE                 DATA   AGE
mysql-secret                       Opaque               2      86m
redis-secret                       Opaque               4      3m48s
```

### 3.2 查看Secret中的Key

使用`kubectl describe secret 名称`可以查看`Secret`的详情信息，其中就包括`Key`

```bash
$ kubectl describe secret redis-secret
Name:         redis-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data  # 下面就是所有的key 
====
user:  4 bytes
ip:    9 bytes
pass:  6 bytes
port:  4 bytes
```

### 3.3  查看Secret中的Val

通过使用`kubectl edit secret 名称`可以查看`Secret`中`Key`对应的`Val`信息。

```bash
apiVersion: v1
data:
  ip: MTI3LjAuMC4x
  pass: MTEyMjMz
  port: NjM3OQ==
  user: dGVzdA==
kind: Secret
metadata:
...
```

> <font color=red>需要注意的是，查看的Val是通过base64编码后的,想看到具体值，需要反编码。</font>

```bash
# 反编码查询具体值信息
$  echo -n MTEyMjMz | base64 --decode
112233
```

## 4. 使用Secret

`Pod`可以通过`Volume`或者环境变量的方式使用`Secret`.

### 4.1 Volume方式

#### a. Pod配置文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - image: busybox
      name: write-box
      volumeMounts:
        - mountPath: /home/redisconf
          name: secret-volume
      args:
        - /bin/sh
        - -c
        - sleep 10; touch /tmp/health;sleep 30000

  volumes:
    - name: secret-volume
      secret:
        secretName: redis-secret
```

#### b. 发布验证

```bash
# 发布资源
$ kubectl apply -f test-po.yaml
pod/test-pod created
# 进入pod容器中
$  kubectl exec -it  test-pod sh
# 查看volume映射信息
/ $ cd /home/redisconf/
/home/redisconf $ ls
ip    pass  port  user
# 查看具体文件信息
/home/redisconf $ cat ip
127.0.0.1
```

> 可以看到，`Kubernetes`会在指定的路径`/home/redisconf`下为每条敏感数据创建一个文件，文件名就是数据条目的`Key`，`Value`则以明文存放在文件中。

#### c. 验证变更同步

将`redis-secret`中的`IP`改为`100.100.100.100`，发布后，查看`Pod`中是否更新

```bash
# 1.修改 redis-secret配置文件
$ vim redis-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
data:
  ip: MTAwLjEwMC4xMDAuMTAw # 这里修改了IP为 100.100.100.100
  port: NjM3OQ==
  user: dGVzdA==
  pass: MTEyMjMz
# 2.重新发布
$ kubectl apply -f redis-secret.yaml
secret/redis-secret configured
# 去pod中查看是否更新
/ # cat /home/redisconf/ip
100.100.100.100 
```

### 4.2 环境变量方式

通过`Volume`使用`Secret`，容器必须从文件读取数据，稍显麻烦，`Kubernetes`还支持通过环境变量使用`Secret`。

#### a. Pod配置文件

文件: `test-po2.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod2
spec:
  containers:
    - image: busybox
      name: write-box
      args:
        - /bin/sh
        - -c
        - sleep; touch /tmp/health;sleep 30000
      env: # 通过环境变量
        - name: REDIS_IP # 环境变量名
          valueFrom:
            secretKeyRef:
              name: redis-secret # secret 名称
              key: ip  # secret中的key
        - name: REDIS_PORT
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: port
```

#### b. 发布验证

```bash
# 发布资源
$ kubectl apply -f test-po2.yaml
pod/test-pod2 created
$ kubectl exec -it test-pod2 bash
/ # echo $REDIS_IP
100.100.100.100
/ # echo $REDIS_PORT
6379
```

> <font color=red>需要注意的是，环境变量读取Secret很方便，但无法支撑Secret动态更新。</font>

## 5. ConfigMap介绍

`Secret`可以为`Pod`提供密码、`Token`、私钥等敏感数据；对于一些非敏感数据，比如应用的配置信息，则可以用`ConfigMap`。

> `ConfigMap`的创建和使用方式与`Secret`非常类似，主要的不同是数据以明文的形式存放。

### 5.1 创建ConfigMap

#### a. 通过`yaml`创建

文件: `app_conf.yaml`

```yaml
apiVersion: v1
kind: ConfigMap # 注意这里类型是ConfigMap
metadata:
  name: appconfig
data:
  appName: "k8s学习" # 这里只写明文，不需要base64
  appVersion: "v1.0.0"
```

**发布:**

```bash
$ kubectl apply -f app-conf.yaml
configmap/appconfig created
```

### 5.2 使用ConfigMap

与`Secret`一样，`Pod`也可以通过`Volume`或者**环境变量**的方式使用`Secret`,这里只演示`Volume`方式：

#### a. Pod配置文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-demo
spec:
  containers:
    - image: busybox
      name: write-box
      args:
        - /bin/sh
        - -c
        - sleep; touch /tmp/health;sleep 30000
      volumeMounts:
        - mountPath: "/etc/k8sconf" # 映射到容器的目录位置
          name: "confmapdemo"
  volumes:
    - name: confmapdemo
      configMap: # 这里有别于secret
        name: appconfig # 上面创建的configMap名称
```

#### b. 发布验证

```bash
# 发布
$ kubectl apply -f config-map-pod.yaml
pod/config-map-demo created
# 进入容器查看
[root@master secret]# kubectl exec -it config-map-demo sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
➜ cd /etc/k8sconf/
➜ ls
appName     appVersion
➜ cat appName
k8s学习
➜ cat appVersion
v1.0.0
```

### 5.3 通过配置文件创建Configmap

大多数情况下，配置信息都以文件形式提供，所以在创建`ConfigMap`时通常采用`--from-file`或`YAML`方式，读取`ConfigMap`时通常采用`Volume`方式。

比如我们的应用配置文件如下,现需要把它传递到容器中:

文件名: `app.toml`

```toml
app_name = "k8s-learn"
app_version = "v1.0.0"
[database.default]
driver = "mysql"
host = "127.0.0.1"
port = "3306"
username = "test"
password = "test"
[redis.default]
addr = "127.0.0.1:6379"
db = 2
```

#### a. 创建

```bash
# 发布
$ kubectl create configmap appconf --from-file=./app.toml
# 查看
$ kubectl describe cm appconf
Name:         appconf
Namespace:    default
...
Data
====
app.toml:
----
app_name = "k8s-learn"
app_version = "v1.0.0"
[database.default]
driver = "mysql"
host = "127.0.0.1"
port = "3306"
username = "test"
password = "test"
[redis.default]
addr = "127.0.0.1:6379"
db = 2
...
```

#### b. 使用

**编辑`Pod`配置文件:`conf-map-file.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-file
spec:
  containers:
    - image: busybox
      name: write-box
      args:
        - /bin/sh
        - -c
        - sleep; touch /tmp/health;sleep 30000
      volumeMounts:
        - mountPath: "/etc/appconf" # 容器映射目录
          name: "confmapdemo"
  volumes:
    - name: confmapdemo
      configMap:
        name: appconf
        items:
          - key: app.toml # configMap中的key
            path: www/app.toml # 存到容器相对目录： 容器映射目录+www/app.toml
```

**发布&验证:**

```bash
# 发布
$ kubectl apply -f conf-map-file.yaml
pod/config-map-file created
# 进入容器查看
➜  pwd
/etc/appconf/www
➜ ls
app.toml
➜ cat app.toml
app_name = "k8s-learn"
app_version = "v1.0.0"
[database.default]
driver = "mysql"
host = "127.0.0.1"
port = "3306"
username = "test"
password = "test"
[redis.default]
addr = "127.0.0.1:6379"
db = 2
```




