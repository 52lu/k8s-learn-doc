## 1.前文

### 1.1  什么是Containerd

`containerd` 是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性，`containerd` 可以负责干下面这些事情：

- 管理容器的生命周期(从创建容器到销毁容器)
- 拉取/推送容器镜像
- 存储管理(管理镜像及容器数据的存储)
- 调用 `runc `运行容器(与` runc` 等容器运行时交互)
- 管理容器网络接口及网络



### 1.2 Containerd和docker区别

![docker架构](https://raw.githubusercontent.com/52lu/pic-go-img/master/img/2022/image-20220801142954335.png)

> `containerd` 是` Docker` 在` 2016 `年 `12` 月从 `Docker Engine` 中分离并单独集成且开源的项目，目标是提供一个更加开放、稳定的容器运行基础设施,后来`Docker` 宣布将 `containerd` 项目捐赠给云原生计算基金会(`CNCF`)



### 1.3  K8s为什么弃用Docker

从网络上搜索得出的原因如下：

- 第一个原因: `Kubernetes` 只能与` CRI `通信，因此要与` Docker` 通信，就必须使用桥接服务。

- 第二个原因: `Kubernetes`只是用到`docker`中的一部分功能，甚至连`docker`网络与存储卷都被排查在外。而那些用不到的功能本身就可能带来安全隐患。

![红色部分是用到的功能](https://raw.githubusercontent.com/52lu/pic-go-img/master/img/2022/60521b369a836cab3081ba08043544dc.png)

- 第三个原因: 调用链变得更短,依赖的功能变少。

![调用链](https://raw.githubusercontent.com/52lu/pic-go-img/master/img/2022/image-20220801165352965.png)

## 2. 安装

### 2.1 下载

```bash
# 下载二进制包 (最新版本: https://github.com/containerd/containerd/releases/)
$ wget https://github.com/containerd/containerd/releases/download/v1.6.4/cri-containerd-cni-1.6.4-linux-amd64.tar.gz
```

### 2.2 解压到根目录

```bash
# 解压到系统的根目录/中
$ tar -zxvf cri-containerd-cni-1.6.4-linux-amd64.tar.gz -C /
```

### 2.3 验证

```bash
# containerd 的版本
$ ctr -v
ctr github.com/containerd/containerd v1.6.4

$ crictl -v
crictl version 1.20.0-24-g53ad8bb7
```

### 2.4 开机启动
```bash
$ systemctl enable --now containerd
```

## 3. 配置文件

### 3.1 生成默认文件

```bash
# 创建目录，有则不创建
$ mkdir -p /etc/containerd 
# 生成默认配置文件
$ containerd config default > /etc/containerd/config.toml
```

### 3.2 修改镜像源地址

```toml
[plugins."io.containerd.grpc.v1.cri"]
  ...
  # sandbox_image = "k8s.gcr.io/pause:3.6"
  sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.7"
```

## 4. `ctr`和`crictl`

### 4.1 两者区别

- `ctr`: 是 `containerd` 的一个客户端工具 ;类似于`docker`的管理工具`docker cli`。
- `crictl`: 是 `CRI` 兼容的容器运行时命令行接口，可以使用它来检查和调试` k8s `节点上的容器运行时和应用程序

> 通俗点理解: `ctr`是`containerd`自带的工具,`crictl`是`CRI`通用的系统工具。

### 4.2 常用命令整理

> 大部分命令只要把`docker`关键字改为`crictl`命令即可操作`containerd`，比如 **:<font color=red>docker ps 改成 crictl ps</font>**



![image-20220809153520718](https://raw.githubusercontent.com/52lu/pic-go-img/master/img/2022/image-20220809153520718.png)



> <font color=red font-size=18>虽然kubernetes弃用了docker,但是通过docker构建的镜像，在containerd中依然可以正常使用,后面会讲到....</font>

