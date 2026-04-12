# Kubernetes 入门学习笔记 06：理解 requests、limits、Pending 与 OOMKilled

@[toc]

## 问题背景

前面已经学会了：

- `Deployment`
- `Service`
- `ConfigMap`
- `Secret`
- `Namespace`
- `Probe`

这时会自然遇到另外一类非常关键的问题：

- 一个 `Pod` 到底能用多少 CPU 和内存
- 为什么有的 `Pod` 明明 YAML 没报错，却一直 `Pending`
- 为什么有的容器运行一会儿就重启
- 为什么会看到 `Exit Code 137`
- `Pending` 和 `OOMKilled` 到底有什么区别

这些问题，本质上都和：

`resources`

有关。

这一篇的目标就是把资源管理这条主线讲清楚，并建立下面这层完整理解：

`requests` 主要影响调度阶段，`limits` 主要影响运行阶段；`Pending` 通常是调度前资源不够，运行中内存超限则可能导致容器被杀并进入反复重启。

## 一、先记住 requests 和 limits 的区别

### 1. requests

`requests` 更像是在对 Kubernetes 说：

`我至少需要这么多资源，你在调度 Pod 时要把这部分考虑进去。`

也就是说，它更偏向：

- 调度
- 资源预留
- 最低需求

### 2. limits

`limits` 更像是在说：

`我最多只能用这么多资源，超过就不允许。`

也就是说，它更偏向：

- 运行时约束
- 上限控制

所以最值得记住的一句话是：

`requests 决定调度时至少按多少资源给你留位置，limits 决定运行时最多能用多少资源。`

## 二、CPU 和内存超限的后果不完全一样

这是资源管理里最容易混淆的一点，必须单独讲清楚。

### 1. CPU 超过 limit

通常会表现为：

- 被限速
- 变慢
- 不一定立刻被杀掉

### 2. 内存超过 limit

通常会表现为：

- 容器被系统 kill
- 可能看到 `Exit Code 137`
- 可能进入反复重启
- 常见现象是 `CrashLoopBackOff`

所以一定要分清：

`CPU 超限常见后果是变慢，内存超限常见后果是被杀。`

## 三、本文使用的示例文件

为了方便直接复现，本文统一使用下面这些示例文件：

```text
k8s-demo/
  resources-demo.yaml
  resources-oom-demo.yaml
  resources-pending-demo.yaml
```

本文中的实验都在 `dev` 命名空间中进行。  
如果当前集群里还没有 `dev` 命名空间，可以先执行：

```bash
kubectl create namespace dev
```

如果提示 `AlreadyExists`，说明已经存在，可以直接继续。

## 四、最小 resources 示例

下面内容保存为 `resources-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resources-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resources-demo
  template:
    metadata:
      labels:
        app: resources-demo
    spec:
      containers:
        - name: app
          image: python:3.11-slim
          command:
            - sh
            - -c
            - "python -m http.server 8080"
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: resources-demo-service
  namespace: dev
spec:
  selector:
    app: resources-demo
  ports:
    - port: 80
      targetPort: 8080
```

这里最关键的是：

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

这表示：

- 调度时按至少 `100m CPU + 128Mi 内存` 计算
- 运行时最多允许使用：
  - `500m CPU`
  - `256Mi 内存`

这里的单位先这样理解就够了：

### CPU

- `100m` = `0.1` 个 CPU 核
- `500m` = `0.5` 个 CPU 核

### 内存

- `128Mi`
- `256Mi`

可以先理解成常见的内存容量单位。

保存后执行：

```bash
cd k8s-demo
kubectl apply -f resources-demo.yaml
kubectl get pods -n dev
kubectl describe pod -n dev -l app=resources-demo
```

重点看 `describe` 里的：

- `Requests`
- `Limits`

如果能看到：

- `cpu: 100m`
- `memory: 128Mi`
- `cpu: 500m`
- `memory: 256Mi`

就说明资源配置已经真实进入了 Pod 规格。

## 五、为什么 requests 会影响调度

Kubernetes 在调度 Pod 到某个节点时，不是只看：

- 节点活着没

而是还会看：

- 节点当前还有多少可分配资源
- 这个 Pod 的 `requests` 是多少

如果一个节点剩余资源不够满足 Pod 的 `requests`，那么调度器就不会把它安排上去。

所以更准确地说：

`requests 不是建议值，而是调度时非常真实的最低资源门槛。`

## 六、为什么 limits 会影响运行

`limits` 更偏向运行阶段的资源上限。

也就是说，Pod 已经调度成功、容器已经运行起来之后，才会开始真正体现出：

- 最多能吃多少 CPU
- 最多能吃多少内存

所以一定要分清：

- `requests`：更像“能不能上车”
- `limits`：更像“上车后最多能用多少”

## 七、运行期资源超限：最小 OOM 场景

下面内容保存为 `resources-oom-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resources-oom-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resources-oom-demo
  template:
    metadata:
      labels:
        app: resources-oom-demo
    spec:
      containers:
        - name: app
          image: python:3.11-slim
          command:
            - python
            - -c
            - |
              data = []
              while True:
                  data.append("x" * 1024 * 1024)
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "128Mi"
```

这段 Python 代码的作用是：

- 持续往内存里塞大字符串
- 内存会不断上涨

而这份配置又明确限制：

```yaml
limits:
  memory: "128Mi"
```

所以这个实验的目标就是故意制造：

`运行中内存超限`

保存后执行：

```bash
kubectl apply -f resources-oom-demo.yaml
kubectl get pods -n dev -w
```

你通常会看到类似现象：

- 先短暂创建成功
- 很快变成 `Error`
- 然后进入反复重启
- 后续可能出现 `CrashLoopBackOff`

例如常见链路是：

```text
Pending
ContainerCreating
Error
CrashLoopBackOff
```

再执行：

```bash
kubectl describe pod -n dev -l app=resources-oom-demo
kubectl get pod -n dev -l app=resources-oom-demo -o yaml
```

这里重点看：

- `Restart Count`
- `State`
- `Last State`
- `Exit Code`
- `Reason`
- `Events`

在本地实验里，常见现象包括：

- `Restart Count` 持续增加
- `Exit Code: 137`
- `Reason: Error`
- `Back-off restarting failed container`

有些环境下还会直接看到：

- `Reason: OOMKilled`

如果没有直接出现 `OOMKilled` 字样，但你看到的是：

- 无限吃内存脚本
- 明确的 `memory limit`
- `Exit Code 137`
- 反复重启

那它依然是非常典型的：

`内存打爆后的失败场景`

所以最值得记住的一句话是：

`memory limit 更接近硬上限，超过后容器可能会被系统 kill。`

## 八、为什么这次会进入 CrashLoopBackOff

在 `resources-oom-demo` 里，容器每次一启动就会继续疯狂吃内存，所以它不是偶发失败，而是：

`每次重启后都会再次失败`

这时 Kubernetes 不会无限制地立刻高频重启，而是会进入：

`CrashLoopBackOff`

也就是：

- 重启间隔逐渐变长
- 避免容器疯狂高频重启把节点打爆

所以当你看到：

- `RESTARTS` 持续增加
- 状态在 `Error / Running / CrashLoopBackOff` 之间切换

就说明：

`Kubernetes 正在尝试自动恢复，但恢复一直失败。`

## 九、为什么 Pod AGE 很长，但还是一直在重启

这是另一个非常容易误解的点。

例如你可能会看到类似：

```text
resources-oom-demo-xxxx   0/1   CrashLoopBackOff   5 (0s ago)   4m37s
```

这里的：

- `4m37s`

表示的是：

`这个 Pod 从创建到现在已经存在了 4 分 37 秒`

而不是：

`这次容器已经稳定运行了 4 分 37 秒`

也就是说：

- `Pod` 还在
- 但里面的容器已经被反复重启了很多次

这和前面探针那一篇学到的现象是一致的：

`容器重启，不等于 Pod 一定被重新创建。`

## 十、调度期资源不够：最小 Pending 场景

前面那个实验已经说明：

- Pod 能调度成功
- 但运行中会因为资源问题失败

接下来要看另一类非常不同的问题：

`还没开始运行，就因为 requests 太大而调度不上去。`

下面内容保存为 `resources-pending-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resources-pending-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resources-pending-demo
  template:
    metadata:
      labels:
        app: resources-pending-demo
    spec:
      containers:
        - name: app
          image: python:3.11-slim
          command:
            - sh
            - -c
            - "sleep 3600"
          resources:
            requests:
              cpu: "100"
              memory: "200Gi"
            limits:
              cpu: "100"
              memory: "200Gi"
```

这份配置故意把资源写得很夸张：

- `100` 个 CPU
- `200Gi` 内存

对本地 `kind` 单节点学习环境来说，基本不可能满足。

保存后执行：

```bash
kubectl apply -f resources-pending-demo.yaml
kubectl get pods -n dev
kubectl describe pod -n dev -l app=resources-pending-demo
```

你通常会看到：

- `STATUS = Pending`

并且在 `describe` 的 `Events` 里看到类似：

```text
0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory
```

有时还会看到：

```text
No preemption victims found for incoming pod
```

当前阶段你只需要先这样理解：

`调度器尝试找可用节点，但没有找到能满足这个 Pod requests 的位置。`

## 十一、Pending 和 OOMKilled 的区别

这是这一篇最重要的对照之一。

### `Pending`

表示：

`Pod 还没真正开始运行，调度阶段就卡住了。`

常见原因：

- `requests` 太大
- 节点剩余 CPU 不够
- 节点剩余内存不够
- 其他调度条件不满足

### `OOMKilled`

表示：

`Pod 已经运行起来了，但容器在运行中因为内存超限被 kill。`

所以最适合记忆的一句话是：

`Pending 是调度前问题，OOMKilled 是运行中问题。`

如果再说得更直白一点：

- `Pending`：连“上车”都没上去
- `OOMKilled`：已经跑起来了，但运行中把内存打爆被杀了

## 十二、管理员通常怎么发现资源问题

如果真实业务里出现资源异常，最常用的观察入口有：

### 1. `kubectl get pods`

重点看：

- `STATUS`
- `READY`
- `RESTARTS`

### 2. `kubectl describe pod`

重点看：

- `Requests`
- `Limits`
- `State`
- `Last State`
- `Events`

### 3. `kubectl get pod -o yaml`

重点看：

- `status.containerStatuses`
- `lastState`
- `exitCode`
- `reason`

### 4. `kubectl logs --previous`

对于反复重启的容器，这条命令尤其重要：

```bash
kubectl logs <pod-name> -n dev --previous
```

它更适合看：

- 上一次失败前的日志

## 十三、资源问题最容易踩的几个坑

### 1. requests 写得过大

结果：

- Pod 长期 `Pending`

### 2. memory limit 写得过小

结果：

- 运行中被 kill
- 反复重启
- 可能 `CrashLoopBackOff`

### 3. 误以为 CPU 和内存超限后果一样

实际上不是：

- CPU 超限更常见是变慢
- 内存超限更常见是被杀

### 4. 只看 AGE，不看 RESTARTS

这会误判：

- 以为 Pod 已经稳定运行很久

实际上可能只是：

- Pod 一直存在
- 但容器已经在里面重启很多次

## 十四、这一篇最值得掌握的 8 个结论

1. `requests` 主要影响调度阶段，`limits` 主要影响运行阶段。
2. `requests` 决定调度器是否能为 Pod 找到合适节点。
3. `limits` 决定容器运行时最多能使用多少资源。
4. CPU 超限常见后果是被限速变慢，内存超限常见后果是被 kill。
5. `Pending` 往往说明调度阶段资源不够或约束不满足。
6. 运行中内存打爆后，常见现象是 `Exit Code 137`、反复重启和 `CrashLoopBackOff`。
7. `Pod AGE` 是 Pod 总年龄，不等于单次容器运行时长。
8. `Pending` 和 `OOMKilled` 分别代表调度前问题和运行中问题。

## 十五、这一篇最值得记住的一句话

如果只记一句话，最值得记住的是：

`Kubernetes 的 resources 体系里，requests 决定 Pod 能不能被调度上去，limits 决定容器运行时最多能吃多少资源；Pending 通常是调度前资源不够，运行中内存超限则可能导致容器被 kill 并进入反复重启。`

## 十六、下一步要学什么

完成这一篇之后，下一步最自然的主题就是：

`QoS`

因为学完：

- `requests`
- `limits`
- `Pending`
- `CrashLoopBackOff`
- 资源超限

之后，下一个自然问题就是：

`如果节点资源紧张，Kubernetes 会优先保谁、先淘汰谁？`

这正是 `Guaranteed`、`Burstable`、`BestEffort` 要解决的问题。
