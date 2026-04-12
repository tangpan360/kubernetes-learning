# Kubernetes 入门学习笔记 07：理解 Guaranteed、Burstable 与 BestEffort

@[toc]

## 问题背景

前一篇已经学会了：

- `requests`
- `limits`
- `Pending`
- 运行中资源超限
- `CrashLoopBackOff`

这时会自然遇到一个新问题：

`如果节点资源紧张，Kubernetes 会优先保谁、先清理谁？`

这个问题的答案，和：

`QoS`

直接相关。

这一篇的目标就是把下面这层逻辑讲清楚：

`QoS 不是单独配置出来的对象，而是 Kubernetes 根据 requests 和 limits 的最终组合，自动推导出的资源服务等级。`

## 一、QoS 到底是什么

`QoS` 全称可以理解成：

`Quality of Service`

也就是：

`资源服务质量等级`

它不是一个你手动单独声明的字段，而是 Kubernetes 根据 Pod 的资源配置方式自动计算出来的结果。

所以更准确地说：

`你不是直接配置 QoS，而是通过 requests / limits 的写法，让 Kubernetes 推导出这个 Pod 属于哪一种 QoS。`

## 二、QoS 的 3 个最核心等级

Kubernetes 最常见的 QoS Class 有 3 个：

1. `Guaranteed`
2. `Burstable`
3. `BestEffort`

它们的稳定性和资源边界清晰程度，大致可以先记成：

`Guaranteed > Burstable > BestEffort`

这里最值得先记住的一句话是：

`资源边界越明确、越稳定，在节点资源紧张时通常越容易被优先保留。`

## 三、Guaranteed 是什么

`Guaranteed` 是资源边界最明确、最稳定的一类。

通常要求每个容器都同时设置：

- `requests`
- `limits`

并且对应资源上：

- `requests = limits`

例如：

- `cpu request = cpu limit`
- `memory request = memory limit`

它的关键意义不在于“更省资源”，而在于：

`资源承诺最明确、最稳定、最可预测。`

也就是说：

- 调度时系统知道你明确需要多少
- 运行时系统也知道你最多就到多少
- 不存在“先占很少、后面又猛涨很多”的弹性波动区间

所以在资源压力下，它通常最容易被优先保住。

## 四、Burstable 是什么

`Burstable` 是中间层。

它通常意味着：

- 你已经配置了一部分资源边界
- 但没有达到 `Guaranteed` 那么严格

例如：

- 配置了 `requests` 和 `limits`
- 但两者不相等

或者：

- 只配置了一部分资源项

例如前面学过的这种：

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

它的特点是：

- 调度阶段有最低资源承诺
- 运行阶段也有上限
- 但中间仍然存在弹性波动区间

所以 Kubernetes 会认为：

`它不是完全没资源承诺，但也不像 Guaranteed 那样最稳定。`

## 五、BestEffort 是什么

`BestEffort` 是资源边界最模糊的一类。

它的特点是：

- 没有 `requests`
- 没有 `limits`

也就是说：

- 没有最低资源承诺
- 没有明确资源上限
- 资源画像最不清楚

所以在节点资源紧张时，它通常是最容易先出问题、最容易先被牺牲的一类。

这里最值得记住的一句话是：

`BestEffort 最弱，不是因为它一定会占满整个节点，而是因为它没有给系统提供明确的资源边界信息。`

## 六、为什么 request = limit 反而更“高级”

这是很多初学者最容易困惑的一点。

直觉上看，好像：

- 弹性更大不是更好吗

但 Kubernetes 更看重的是：

`资源承诺是否清晰、是否可预测。`

例如：

### 情况 1：request < limit

比如：

- `request = 100m`
- `limit = 500m`

这表示：

- 调度时只按 `100m` 给你找位置
- 运行起来后，你却可能冲到 `500m`

这对节点来说，就是：

`最低只占一点，但运行中可能突然膨胀很多。`

所以它更难做最稳定的资源保障。

### 情况 2：request = limit

比如：

- `request = 500m`
- `limit = 500m`

这表示：

- 调度时按 `500m` 安排位置
- 运行时最多也就是 `500m`
- 没有额外弹性区间

所以系统会觉得：

`你需要多少，就是多少；你最多能到多少，也就是多少。`

这就是 `Guaranteed` 更高一级的根本原因。

## 七、QoS 和 requests / limits 到底是什么关系

这是这一篇最核心的主线。

更准确地说：

`QoS 不是独立的新配置，而是 Kubernetes 根据最终生效后的 requests / limits 组合自动推导出来的资源等级。`

所以可以先记成下面这组规则：

### 1. 什么都不写

- 没有 requests
- 没有 limits
- => `BestEffort`

### 2. 写了一部分，或者 request 和 limit 不相等

- => `Burstable`

### 3. 每个容器都写了 requests 和 limits，并且相等

- => `Guaranteed`

所以最简单的一句话就是：

`QoS 是 resources 写法的结果。`

## 八、最小 QoS 实验：三种等级对照

下面内容保存为 `qos-demo.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

这 3 个 Pod 分别代表：

- `qos-guaranteed`：request 和 limit 相等
- `qos-burstable`：request 和 limit 不相等
- `qos-besteffort`：完全不写 resources

保存后执行：

```bash
cd k8s-demo
kubectl apply -f qos-demo.yaml
kubectl get pods -n dev
kubectl describe pod qos-guaranteed -n dev
kubectl describe pod qos-burstable -n dev
kubectl describe pod qos-besteffort -n dev
```

重点看每个 Pod 里的：

- `QoS Class`

你应该会看到：

- `qos-guaranteed -> Guaranteed`
- `qos-burstable -> Burstable`
- `qos-besteffort -> BestEffort`

这一步的意义是把三种等级先直观跑通。

## 九、只写 requests 会怎样

下面内容保存为 `qos-partial-demo.yaml` 的前半部分：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-request-only
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
```

这种情况的典型现象是：

- 有 `requests`
- 没有对应 `limits`
- `QoS Class` 通常是 `Burstable`

也就是说：

`只写 requests，主要是给调度一个最低资源门槛，但并没有把运行时上限也明确锁死。`

## 十、只写 limits 会怎样

下面内容保存为 `qos-partial-demo.yaml` 的后半部分：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-limit-only
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      resources:
        limits:
          cpu: "100m"
          memory: "256Mi"
```

这里最关键的点是：

`只写 limits，不一定只是“只限制上限”。`

在常见行为下，对应资源的 `requests` 往往也会被补成和 `limits` 一样。

也就是说，最终你可能观察到：

- `Requests` 被自动补出
- 数值和 `Limits` 一样

如果 CPU 和内存都满足这一条件，那么它最终甚至可能变成：

- `Guaranteed`

所以一定要记住：

`只写 limits，很多时候不仅会限制运行时上限，还会连调度门槛一起抬高。`

## 十一、部分 resources 配置实验

下面内容保存为 `qos-partial-demo.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-request-only
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-limit-only
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      resources:
        limits:
          cpu: "100m"
          memory: "256Mi"
```

保存后执行：

```bash
kubectl apply -f qos-partial-demo.yaml
kubectl get pods -n dev
kubectl describe pod qos-request-only -n dev
kubectl describe pod qos-limit-only -n dev
```

重点看：

- `Requests`
- `Limits`
- `QoS Class`

这个实验最重要的观察结论通常是：

### `qos-request-only`

- 有 `Requests`
- 没有对应 `Limits`
- `QoS Class = Burstable`

### `qos-limit-only`

- `Requests` 被自动补出来
- 数值和 `Limits` 对应资源保持一致
- 最终可能进入 `Guaranteed`

这一步的真正意义，不是背结果，而是建立一个非常重要的认识：

`只写 requests 和只写 limits 的影响并不对称。`

## 十二、为什么 Guaranteed 最稳、BestEffort 最弱

你现在可以把这三者放在一条线上理解：

### `BestEffort`

`我没给你任何资源承诺。`

### `Burstable`

`我给了你部分资源承诺，但运行时还可能有弹性波动。`

### `Guaranteed`

`我给了你最完整、最稳定的资源承诺。`

所以：

- 配置越完整
- 边界越明确
- 资源画像越稳定
- QoS 等级通常越高

这也是为什么在节点资源紧张时，大方向上通常可以先记成：

- `Guaranteed` 最稳
- `Burstable` 中间
- `BestEffort` 最弱

## 十三、QoS 不是“免死金牌”

这里一定要再补一个现实中的关键点：

`Guaranteed 最稳`

不等于：

`Guaranteed 永远不会出问题`

它只表示：

- 在资源压力下更容易被优先保留
- 资源边界更稳定

但如果应用本身有问题，或者你自己配置的资源上限就不够，那么它仍然可能：

- OOM
- CrashLoop
- Probe 失败
- 服务异常

所以更准确的理解是：

`QoS 是资源压力下的优先级和稳定性等级，不是绝对安全等级。`

## 十四、这一篇最值得掌握的 8 个结论

1. `QoS` 是 Kubernetes 根据最终生效后的 requests / limits 组合自动推导出来的资源等级。
2. `Guaranteed` 通常要求每个容器都设置了 requests 和 limits，并且两者相等。
3. `Burstable` 通常表示已经声明了部分资源边界，但还存在弹性波动区间。
4. `BestEffort` 表示既没有 requests，也没有 limits，资源边界最模糊。
5. `request = limit` 的意义不在于更省资源，而在于资源承诺最明确、最稳定、最可预测。
6. `QoS` 越高，通常说明资源边界越清晰，在节点资源紧张时越容易被优先保留。
7. 只写 `requests` 通常更偏向提供调度门槛，常见结果是 `Burstable`。
8. 只写 `limits` 往往会让对应资源的 `requests` 也被补出来，因此影响可能比看起来更大。

## 十五、这一篇最值得记住的一句话

如果只记一句话，最值得记住的是：

`QoS 本质上不是单独配置出来的对象，而是 Kubernetes 根据 requests 和 limits 的最终组合自动推导出的资源服务等级；资源边界越明确、越稳定，在节点资源紧张时通常越容易被优先保留。`

## 十六、下一步要学什么

完成这一篇之后，下一步最自然的主题就是：

`PV / PVC / StorageClass`

因为到这里为止，已经把：

- 资源申请
- 运行上限
- 调度前问题
- 运行中问题
- 资源等级

这条“计算资源”主线跑通了。

接下来最自然就要进入另一条同样重要的主线：

`Pod 重建以后，数据到底怎么保存。`
