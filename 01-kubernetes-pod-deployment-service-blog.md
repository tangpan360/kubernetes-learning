# Kubernetes 入门学习笔记 01：用 kind 跑通 Pod、Deployment、Service 与 NodePort

@[toc]

## 问题背景

学完 Docker 和 Docker Compose 之后，很自然会遇到一个新问题：

`如果服务不再只跑在一台机器上，而是要跑在一个集群里，应该怎么部署和管理？`

这正是 Kubernetes 要解决的问题。

但是在真正进入：

- `ConfigMap`
- `Ingress`
- `滚动更新`
- `健康检查`

这些更完整的主题之前，最先要跑通的一条最小主线其实只有 4 个东西：

- `kind`
- `Pod`
- `Deployment`
- `Service`

再往前走一步，就是：

- `ClusterIP`
- `NodePort`

这一篇的目标就是把这条最小主线真正跑通，并且让整个链路清晰到可以自己复现。

## 一、先明确这篇要解决什么

这一篇不追求把 Kubernetes 所有对象一次讲完，而是只解决下面几个最核心的问题：

1. `Pod` 到底是什么
2. 为什么真正部署应用时更常用 `Deployment`
3. 为什么需要 `Service`
4. 为什么 `Service` 创建成功了，宿主机却不一定能直接访问
5. 在 `kind` 本地集群里，怎么真正把服务访问打通

如果这几个问题真正搞清楚了，Kubernetes 的第一层理解就已经建立起来了。

## 二、先分清 3 个最核心对象

### 1. Pod

`Pod` 是 Kubernetes 中最小的运行和调度单位。

容器真正运行在 `Pod` 里。很多时候一个 `Pod` 里只有一个主容器，但 Kubernetes 不是直接把“单个容器”当成最小调度对象，而是通过 `Pod` 来统一承载和管理容器。

可以先把它理解成：

- 容器是执行单元
- `Pod` 是 Kubernetes 的最小运行单元

### 2. Deployment

单独创建一个 `Pod` 能跑起来应用，但不适合作为长期部署方式。

原因很简单：

- `Pod` 会消失
- `Pod` 可能被重建
- `Pod` 可能需要多个副本
- 更新镜像版本时，也不能只靠手工删建 `Pod`

所以 Kubernetes 更常用：

`Deployment`

来管理一组同类 `Pod`。

它做的事情包括：

- 声明副本数
- 维持目标状态
- `Pod` 挂掉后补回来
- 后续支持滚动更新

所以更准确地说：

- `Pod` 负责“一个实例怎么跑”
- `Deployment` 负责“这一类实例应该维持什么状态”

### 3. Service

有了 `Deployment` 以后，后面可能会有多个 `Pod` 副本。

这时又会遇到一个问题：

`外部到底访问哪一个 Pod？`

直接访问 `Pod` 不稳定，因为：

- `Pod` 可能重建
- `Pod IP` 可能变化
- 副本数量可能变化

所以 Kubernetes 引入了：

`Service`

`Service` 的作用是：

`给一组符合条件的 Pod 提供稳定访问入口。`

也就是说：

- `Deployment` 管一组 `Pod`
- `Service` 给这组 `Pod` 提供稳定入口

## 三、这 3 个对象之间是什么关系

最常见的最小应用模型可以理解成：

```text
Deployment
  -> 管理一组 Pod
  -> Pod 里运行容器
Service
  -> 通过 label selector 找到这些 Pod
  -> 对外提供稳定访问入口
```

这里最关键的是 `label`。

通常会这样配：

- `Deployment` 生成的 `Pod` 带上 `app: nginx`
- `Service` 用 `selector: app: nginx` 去找到这些 `Pod`

所以可以先记一句话：

`Deployment 负责把 Pod 跑出来，Service 负责把这组 Pod 找出来并暴露出来。`

## 四、本文使用的本地环境

为了在本地最小成本学习 Kubernetes，这里使用：

`kind`

也就是：

`Kubernetes IN Docker`

它的特点是：

- 本地学习成本低
- 不需要真的准备多台机器
- 很适合先理解对象关系和基本操作

本文默认目录结构如下：

```text
k8s-demo/
  kind-config.yaml
  nginx-demo.yaml
```

## 五、先准备 kind 配置

为了让宿主机后面能直接访问 `NodePort`，先创建 `kind-config.yaml`：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://docker.m.daocloud.io", "https://docker.1ms.run"]
```

这份配置做了两件事：

1. 把 kind 节点容器的 `30080` 映射到宿主机的 `30080`
2. 给 `docker.io` 配置镜像加速地址

创建集群：

```bash
cd k8s-demo
kind create cluster --name demo --config kind-config.yaml
kubectl config current-context
kubectl get nodes
kubectl get pods -A
```

如果一切正常，当前上下文应该是：

```bash
kind-demo
```

并且可以看到 `control-plane` 节点和一批 `kube-system` 下的系统 Pod。

## 六、准备最小示例 YAML

接着创建 `nginx-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

这份 YAML 一共包含两个对象：

1. 一个 `Deployment`
2. 一个 `Service`

## 七、先看懂 Deployment 这一部分

`Deployment` 里的核心字段有：

- `replicas: 2`
- `selector.matchLabels`
- `template.metadata.labels`
- `template.spec.containers`

其中最关键的关系是：

### 1. `replicas`

表示期望维持几个 `Pod` 副本。

这里写 `2`，意味着 Kubernetes 会尽量维持 2 个 nginx Pod 存在。

### 2. `selector.matchLabels`

表示这个 `Deployment` 要管理哪些标签的 `Pod`。

### 3. `template.metadata.labels`

表示这个 `Deployment` 新创建出来的 `Pod` 会打上什么标签。

实际使用中，这两者必须对应起来。  
最常见写法就是都写成：

```yaml
app: nginx
```

这样 `Deployment` 才知道自己创建出来的 `Pod` 也是自己要管理的那一批。

### 4. `containers`

表示这个 `Pod` 里运行什么容器。

这里就是：

- 容器名：`nginx`
- 镜像：`nginx:latest`
- 容器对外监听端口：`80`

## 八、再看懂 Service 这一部分

这一部分的核心字段有：

- `type: NodePort`
- `selector`
- `ports`

### 1. `selector`

它的作用是根据标签找到后端 `Pod`。

这里写的是：

```yaml
selector:
  app: nginx
```

也就是说，`Service` 会把所有带有 `app: nginx` 标签的 `Pod` 当成自己的后端。

### 2. `port` 和 `targetPort`

- `port`：Service 自己提供服务的端口
- `targetPort`：转发到后端 Pod 的端口

这里都写 `80`，表示访问 `Service:80` 时，最终会被转发到 `Pod:80`。

### 3. `NodePort`

`NodePort` 的含义是：

`除了集群内部访问，还额外在节点上开放一个固定端口供外部访问。`

这里指定的是：

```yaml
nodePort: 30080
```

也就是希望通过节点的 `30080` 访问这个服务。

## 九、为什么默认创建成功后宿主机还不一定能访问

这是本地学习时最容易困惑的点之一。

即使：

- `Service` 已经是 `NodePort`
- `nodePort` 已经写成了 `30080`

也不等于你一定能直接在宿主机访问：

```bash
curl http://localhost:30080
```

原因是：

`kind` 里的 Kubernetes 节点，本质上是 Docker 容器，不是你的宿主机本身。

所以 `NodePort` 开在的是：

- kind 节点容器里的 `30080`

而不是自动开在宿主机的 `30080`。

这就是为什么前面要在 `kind-config.yaml` 里增加：

```yaml
extraPortMappings:
  - containerPort: 30080
    hostPort: 30080
```

它打通的是：

- 宿主机 `30080`
- 到 kind 节点容器 `30080`

## 十、开始部署应用

执行：

```bash
kubectl apply -f nginx-demo.yaml
kubectl get deployments
kubectl get pods
kubectl get services
```

正常情况下会看到：

- `nginx-deployment` 已创建
- 有 2 个 nginx Pod
- `nginx-service` 已创建

这一步里最值得观察的几个字段是：

### 1. `kubectl get deployments`

重点看：

- `READY`
- `UP-TO-DATE`
- `AVAILABLE`

如果看到类似 `2/2`，通常表示两个副本已经都可用了。

### 2. `kubectl get pods`

重点看：

- 是否有 2 个 Pod
- 状态是否是 `Running`

### 3. `kubectl get services`

重点看：

- `TYPE`
- `CLUSTER-IP`
- `PORT(S)`

如果看到：

- `TYPE=NodePort`
- `PORT(S)=80:30080/TCP`

就说明：

- 集群内部访问入口是 `80`
- 节点对外暴露端口是 `30080`

## 十一、理解 ClusterIP 和 NodePort

### 1. ClusterIP

`ClusterIP` 是集群内部访问服务的稳定入口。

它主要用于：

- Pod 访问 Pod
- 服务访问服务
- 集群内部调用

如果只创建普通 `Service`，默认类型就是 `ClusterIP`。

它的特点是：

- 集群内可访问
- 宿主机通常不能直接访问

### 2. NodePort

`NodePort` 是在 `ClusterIP` 的基础上，再额外开放一个节点端口。

它的特点是：

- 保留集群内部访问能力
- 再增加一个对外入口

所以可以这样理解：

`ClusterIP` 面向集群内部，`NodePort` 面向集群外部测试访问。`

## 十二、再用 describe 看清楚转发关系

执行：

```bash
kubectl describe service nginx-service
```

这里最值得看的是两个字段：

### 1. `Selector`

例如：

```text
Selector: app=nginx
```

说明这个 `Service` 是通过 `app=nginx` 找后端 Pod 的。

### 2. `Endpoints`

例如：

```text
Endpoints: 10.244.0.5:80,10.244.0.6:80
```

这说明当前有两个后端 Pod 正在为这个 `Service` 提供服务。

也就是说：

- `Service` 本身并不直接运行容器
- 它只是一个稳定入口
- 背后真正处理请求的是这些 Pod

## 十三、真正访问服务

当前配置正确后，宿主机执行：

```bash
curl http://localhost:30080
```

如果链路打通，通常会返回 nginx 的欢迎页 HTML。

这就说明整个访问链已经跑通了。

## 十四、最终访问链是怎么走通的

当 `kind-config.yaml` 和 `nginx-demo.yaml` 都配置正确后，宿主机访问：

```bash
curl http://localhost:30080
```

其实际链路是：

```text
localhost:30080
  ->
宿主机 30080
  ->
kind 节点容器 30080
  ->
NodePort Service
  ->
Endpoints
  ->
nginx Pod:80
```

如果一切正常，就会返回 nginx 的首页 HTML。

## 十五、这一篇最值得记住的 5 句话

1. `Pod` 是 Kubernetes 最小的运行和调度单位。
2. `Deployment` 用来维持一组 `Pod` 的目标状态。
3. `Service` 用来给一组 `Pod` 提供稳定访问入口。
4. `ClusterIP` 主要用于集群内部访问。
5. 在 `kind` 中使用 `NodePort` 时，还需要把宿主机端口映射到 kind 节点容器。

## 十六、这一篇最值得记住的一句话

如果只记一句话，最值得记住的是：

`Kubernetes 的最小应用模型不是单独的 Pod，而是 Deployment 负责维持 Pod，Service 负责稳定暴露 Pod，而在 kind 本地环境中，NodePort 还需要配合宿主机端口映射才能真正访问到服务。`

## 十七、下一步要学什么

跑通这条最小主线之后，下一步最自然的延伸就是：

- `kubectl logs`
- `kubectl describe`
- `kubectl delete`
- `kubectl scale`
- `kubectl rollout`

也就是：

`开始进入 Kubernetes 的基本运维与排障动作。`
