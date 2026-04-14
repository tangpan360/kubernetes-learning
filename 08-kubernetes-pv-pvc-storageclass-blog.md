# Kubernetes 入门学习笔记 08：理解 PV、PVC、StorageClass 与持久化数据

@[toc]

## 问题背景

前面已经学会了：

- `Deployment`
- `Service`
- `ConfigMap`
- `Secret`
- `Probe`
- `resources`
- `QoS`

这时会自然遇到另一个非常关键的问题：

- `Pod` 被删掉以后，数据怎么办
- 容器里写入的文件为什么有时会丢
- 为什么删掉 `Pod` 后数据还在，删掉 `PVC` 后数据却没了
- `PV`、`PVC`、`StorageClass` 三者到底是什么关系

这些问题，本质上都和：

`持久化存储`

有关。

这一篇的目标就是把存储主线讲清楚，并建立下面这层完整理解：

`PVC` 是存储申请单，`PV` 是实际卷，`StorageClass` 是动态供卷规则；`ConfigMap/Secret` 主要是配置卷，`PVC` 才是真正的数据卷。

## 一、为什么一定要学存储

前面已经反复看到：

- `Pod` 会被删除
- `Pod` 会被重建
- 容器会重启
- `Pod IP` 会变化

如果数据只是写在容器自己的文件系统里，那么：

- `Pod` 没了
- 容器重建了
- 数据通常也就没了

所以只要你要保存这些内容：

- 数据库数据
- 用户上传文件
- 业务实际写入的数据
- Pod 重建后仍然要保留的数据

就需要：

`持久化卷`

## 二、先分清配置卷和数据卷

这一点非常重要。

你前面已经学过：

### `ConfigMap volume`

它主要解决的是：

- 普通配置注入
- 文本配置文件挂载

### `Secret volume`

它主要解决的是：

- 密码
- token
- 证书
- 敏感配置注入

它们的共同特点是：

`更像把配置投影给程序读取，而不是给程序长期写业务数据。`

而 `PVC volume` 完全不同，它解决的是：

`真正的业务数据持久化存储。`

所以最值得先记住的一句话是：

`ConfigMap / Secret volume 主要解决配置注入，PVC volume 主要解决持久化数据。`

## 三、先分清 3 个最核心对象

### 1. PV

`PV` 全称是：

`PersistentVolume`

可以先理解成：

`集群里的实际持久化卷资源。`

也就是：

- 一块真实可用的卷
- 一块实际存数据的存储资源

### 2. PVC

`PVC` 全称是：

`PersistentVolumeClaim`

可以先理解成：

`Pod 对持久化卷提出的申请单。`

也就是：

- 我要多大空间
- 我要什么访问方式
- 请给我一块合适的卷

### 3. StorageClass

可以先理解成：

`动态供卷规则或模板。`

也就是：

- 用什么类型的存储
- 由谁创建
- 按什么规则回收和绑定

所以最适合记住的一句话是：

`PV 是实际卷，PVC 是申请单，StorageClass 是动态供卷规则。`

## 四、先看当前环境支不支持动态供卷

在开始实验之前，先执行：

```bash
kubectl get storageclass
```

如果能看到：

- 有默认 `StorageClass`

例如你的本地环境里看到的是：

```text
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  ...
```

就说明当前环境可以直接走：

`动态供卷`

这意味着：

- 不需要先手工创建静态 `PV`
- 创建 `PVC` 后，Kubernetes 可以按默认 `StorageClass` 动态准备对应卷

## 五、本文使用的示例文件

为了方便直接复现，本文统一使用下面这些示例文件：

```text
k8s-demo/
  pvc-demo.yaml
  pvc-pod-demo.yaml
```

本文中的实验都在 `dev` 命名空间中进行。  
如果当前集群里还没有 `dev` 命名空间，可以先执行：

```bash
kubectl create namespace dev
```

如果提示 `AlreadyExists`，说明已经存在，可以直接继续。

## 六、最小 PVC 示例

下面内容保存为 `pvc-demo.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

这份 YAML 最关键的字段有两个：

### 1. `accessModes`

这里写的是：

```yaml
ReadWriteOnce
```

当前阶段可以先粗略理解成：

`这块卷通常适合被一个节点上的工作负载以读写方式使用。`

这里的“一个节点上的工作负载”，可以先理解成：

`运行在同一个节点上的 Pod。`

也就是说，它通常不适合：

- 多个节点同时共享读写

### 2. `resources.requests.storage`

这里写的是：

```yaml
storage: 1Gi
```

表示：

`我申请一块 1Gi 的持久化卷。`

保存后执行：

```bash
cd k8s-demo
kubectl apply -f pvc-demo.yaml
kubectl get pvc -n dev
```

如果你当前环境里的 `StorageClass` 使用的是：

- `WaitForFirstConsumer`

那么你很可能会先看到：

- `STATUS = Pending`

这不一定是出错，而是正常现象。

## 七、为什么 PVC 一开始会是 Pending

你当前环境里的默认 `StorageClass` 带有：

- `volumeBindingMode = WaitForFirstConsumer`

这表示：

`PVC 不会在创建瞬间立刻完成绑定，而是要等真正有 Pod 使用它之后，才会触发绑定和供卷。`

所以当前阶段：

- 只有 `PVC`
- 还没有真正消费它的 Pod

于是会看到：

- `demo-pvc = Pending`

这是正常行为。

## 八、创建使用 PVC 的 Pod

下面内容保存为 `pvc-pod-demo.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data-volume
          mountPath: /data
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: demo-pvc
```

这里最关键的是：

```yaml
volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: demo-pvc
```

表示：

- 这个 Pod 要用一块卷
- 这块卷的数据来源不是 `ConfigMap`
- 也不是 `Secret`
- 而是：
- `demo-pvc` 这张申请单对应的持久化卷

而：

```yaml
volumeMounts:
  - name: data-volume
    mountPath: /data
```

表示：

- 把这块持久化卷挂载到容器里的 `/data`

保存后执行：

```bash
kubectl apply -f pvc-pod-demo.yaml
kubectl get pods -n dev
kubectl get pvc -n dev
kubectl get pv
```

这一步通常会看到：

- `pvc-demo-pod` 创建成功
- `demo-pvc` 从 `Pending` 变成 `Bound`
- 自动多出一个对应的 `PV`

这说明：

`PVC 已经通过默认 StorageClass 触发了动态供卷。`

## 九、往卷里写数据

接着进入 Pod：

```bash
kubectl exec -it pvc-demo-pod -n dev -- sh
```

在容器里执行：

```sh
echo "hello-pvc" > /data/hello.txt
cat /data/hello.txt
exit
```

如果能看到：

```text
hello-pvc
```

说明写入成功。

这一步非常关键，因为这里写入的已经不是：

- 容器自己的临时文件

而是：

- 这块持久化卷

## 十、为什么删 Pod 后数据还在

现在执行：

```bash
kubectl delete pod pvc-demo-pod -n dev
kubectl apply -f pvc-pod-demo.yaml
kubectl get pods -n dev
kubectl exec -it pvc-demo-pod -n dev -- sh
cat /data/hello.txt
exit
```

如果还能看到：

```text
hello-pvc
```

那就说明这个实验跑通了。

这里最关键的不是“新 Pod 记住了旧 Pod 的文件”，而是：

`旧 Pod 和新 Pod 挂载的是同一块外部持久化卷。`

所以真正发生的是：

1. 旧 Pod 挂载 `demo-pvc`
2. 数据写进卷
3. 旧 Pod 删除
4. 卷本身还在
5. 新 Pod 再次挂载同一个 `demo-pvc`
6. 数据仍然可见

所以最值得记住的一句话是：

`Pod 和 PVC 的生命周期不是绑死在一起的。`

## 十一、为什么删 PVC 后数据没了

前面那个实验已经证明：

- 删 Pod 不一定删卷

现在再看另一种情况：

执行：

```bash
kubectl delete pod pvc-demo-pod -n dev
kubectl get pvc -n dev
kubectl get pv
kubectl delete pvc demo-pvc -n dev
kubectl get pvc -n dev
kubectl get pv
```

你当前环境里，`StorageClass` 的回收策略是：

- `Delete`

这意味着：

`PVC 删除后，不只是申请单没了，和它绑定的卷资源也会被回收。`

于是通常会看到：

- `PVC` 消失
- 对应 `PV` 也跟着消失

如果这时再重新创建：

```bash
kubectl apply -f pvc-demo.yaml
kubectl apply -f pvc-pod-demo.yaml
kubectl exec -it pvc-demo-pod -n dev -- sh
cat /data/hello.txt
exit
```

你大概率会发现：

- 文件已经不在了

这是因为现在拿到的是：

- 一块新卷

而不是之前那块已经被回收掉的旧卷。

## 十二、Delete 回收策略到底是什么意思

这里最容易误解的一点是：

`Delete` 不只是“PV 对象从列表里没了”。`

更准确地说：

`Delete 表示卷释放后，不只是 Kubernetes 里的 PV 记录会结束，和它关联的真实存储空间也会被一起回收。`

这里的“底层存储”，可以先理解成：

`真正放数据的地方。`

例如可能是：

- 节点本地目录
- 节点磁盘
- 云盘
- NFS
- 分布式存储系统

在你当前这个本地 `local-path` 场景里，它可以先粗略理解成：

`系统动态给你准备的一块真实目录或本地存储空间。`

所以 `Delete` 的直观效果就是：

- 卷不用了
- 真实数据空间也一起删

这也是为什么你删掉 `PVC` 后，再重建就看不到旧数据了。

## 十三、StorageClass 到底在控制什么

你当前环境里的 `StorageClass` 至少已经体现了 3 个很重要的作用：

### 1. 它是默认类

所以创建 `PVC` 时如果不显式写 `storageClassName`，就会默认走它。

### 2. 它决定怎么动态供卷

也就是：

- 用哪个 provisioner
- 按什么方式创建卷

### 3. 它决定卷行为的一部分规则

例如：

- `reclaimPolicy`
- `volumeBindingMode`

所以可以把 `StorageClass` 理解成：

`动态供卷模板 + 存储规则集合`

## 十四、这一篇最值得掌握的 8 个结论

1. `ConfigMap / Secret volume` 主要解决配置注入，`PVC volume` 主要解决业务数据持久化。
2. `PV` 是实际卷，`PVC` 是申请单，`StorageClass` 是动态供卷规则。
3. `ReadWriteOnce` 当前阶段可以先理解成：这块卷通常适合被一个节点上的 Pod 以读写方式使用。
4. 在 `WaitForFirstConsumer` 模式下，PVC 刚创建时先 `Pending` 是正常现象。
5. 当真正有 Pod 使用 PVC 时，Kubernetes 才会完成卷绑定并自动准备对应 PV。
6. 删 `Pod` 不一定删卷，所以 Pod 重建后数据仍然可能在。
7. 删 `PVC` 可能触发卷回收，在 `Delete` 策略下旧数据也会一起消失。
8. 持久化的关键不是 Pod 稳定，而是数据被写进了独立于 Pod 生命周期之外的卷。

## 十五、这一篇最值得记住的一句话

如果只记一句话，最值得记住的是：

`PVC 是存储申请单，PV 是实际卷，StorageClass 负责动态供卷；Pod 会变，但只要卷还在，数据就能继续保留，而在 Delete 回收策略下删掉 PVC 后，卷和旧数据也会一起被回收。`

## 十六、下一步要学什么

完成这一篇之后，下一步最自然的主题就是：

`Ingress`

因为到这里为止，已经把：

- 配置对象
- 权限对象
- 健康检查
- 资源控制
- 存储对象

这几条 K8s 内部主线跑通了。

接下来最自然就会进入另一个非常重要的问题：

`外部 HTTP/HTTPS 流量怎么更正式地进入集群，而不是一直依赖 NodePort。`
