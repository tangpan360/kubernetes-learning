# Kubernetes 入门学习笔记 02：用 kubectl 完成查看、扩缩容、滚动更新与回滚

@[toc]

## 问题背景

在已经跑通：

- `kind`
- `Deployment`
- `Service`
- `NodePort`

这条最小主线之后，下一个自然问题不是立刻进入 `ConfigMap` 或 `Ingress`，而是：

`一个已经跑起来的 Kubernetes 应用，平时到底怎么查看、怎么扩缩容、怎么更新、怎么回滚？`

如果这一层不会，前面的 YAML 只是“写出来了”，还不能算真正具备了基本操作能力。

这一篇就围绕最常用的 `kubectl` 动作，完成一条完整的小闭环：

- 查看对象
- 查看详细状态
- 查看日志
- 扩缩容
- 删除 `Pod`
- 删除 `Deployment`
- 滚动更新
- 失败发布
- 回滚恢复

## 一、本文基于什么环境和文件

本文默认已经具备上一篇中的本地学习环境：

- `kind` 集群已创建成功
- 集群名为 `demo`
- `NodePort` 访问链已打通

示例目录结构如下：

```text
k8s-demo/
  kind-config.yaml
  nginx-demo.yaml
```

其中本文使用的 `nginx-demo.yaml` 内容如下：

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

在开始本文内容之前，可以先执行：

```bash
cd k8s-demo
kubectl apply -f nginx-demo.yaml
```

确保环境处于可用状态。

## 二、先建立最常用的 5 个 kubectl 动作

学习 Kubernetes 时，最先要熟悉的不是更多对象，而是最常用的操作动作。

当前阶段最值得优先掌握的有 5 个：

1. `kubectl get`
2. `kubectl describe`
3. `kubectl logs`
4. `kubectl scale`
5. `kubectl delete`

可以把它们理解成：

- `get`：先看对象有没有、状态怎么样
- `describe`：再看对象详细状态和事件
- `logs`：看容器日志
- `scale`：调整副本数量
- `delete`：删除对象，看控制器会怎么响应

## 三、先看当前对象状态

执行：

```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

这里最应该建立的观察习惯是：

### 1. 先看 Deployment

例如：

- `READY`
- `UP-TO-DATE`
- `AVAILABLE`

如果看到 `2/2`，说明期望的两个副本都已经就绪并可用了。

### 2. 再看 Pod

重点看：

- Pod 数量
- Pod 名称
- `STATUS`
- `RESTARTS`
- `AGE`

如果是正常状态，通常会看到两个 Pod 都处于 `Running`。

### 3. 再看 Service

重点看：

- `TYPE`
- `CLUSTER-IP`
- `PORT(S)`

如果 `TYPE=NodePort` 且 `PORT(S)=80:30080/TCP`，说明外部访问入口仍然是 `30080`。

## 四、describe 是怎么看细节的

`kubectl get` 适合快速总览，但很多问题只看 `get` 是不够的。

这时就要用：

```bash
kubectl describe deployment nginx-deployment
kubectl describe pod <pod-name>
kubectl describe service nginx-service
```

`describe` 最重要的价值是：

- 看对象详细配置
- 看当前状态
- 看调度信息
- 看最近事件

尤其是最后的 `Events`，在排查问题时非常重要。

## 五、logs 是看什么的

执行：

```bash
kubectl logs <pod-name>
```

如果一个 Pod 里只有一个主容器，这样看日志通常就够了。

这里最值得先建立的认知是：

`logs 看的是容器标准输出，不是对象状态总览。`

也就是说：

- 容器启动失败时，`logs` 可能有帮助
- 镜像拉取失败、调度失败时，很多关键信息反而更常出现在 `describe` 的事件里

所以：

`logs` 和 `describe` 不是互相替代，而是互相补充。

## 六、扩容：把副本从 2 调到 3

执行：

```bash
kubectl scale deployment nginx-deployment --replicas=3
kubectl get deployments
kubectl get pods
```

正常情况下会看到：

- `Deployment` 的期望副本数变成 3
- 下面会多出来一个新的 Pod

这里最值得记住的是：

`scale 改的不是某一个 Pod，而是 Deployment 的期望状态。`

Kubernetes 看到新的目标值后，会自己补齐到 3 个副本。

## 七、删一个 Pod，为什么又自动补回来了

先找一个 Pod 名字，再执行：

```bash
kubectl delete pod <pod-name>
kubectl get pods -w
```

通常会看到：

- 旧 Pod 被删除
- 很快又出现一个新 Pod

这背后的原因不是“删失败了”，而是：

`Deployment 仍然存在，它会继续维持期望副本数。`

例如现在期望值是 `3`，那它就必须始终想办法保持 3 个 Pod。

所以这里最核心的结论是：

`删 Pod 更像是在模拟某个实例坏掉了，而 Deployment 会负责重新补一个新的回来。`

## 八、删 Deployment 会发生什么

执行：

```bash
kubectl delete deployment nginx-deployment
kubectl get deployments
kubectl get pods
kubectl get service nginx-service
kubectl describe service nginx-service
```

这时通常会看到：

- `Deployment` 消失了
- 它管理的 Pod 也消失了
- `Service` 还在
- 但 `Endpoints` 为空

这一步非常关键，因为它能帮你彻底分清：

### 1. 删除 Pod

- 管理器还在
- 会自动补回来

### 2. 删除 Deployment

- 管理器没了
- Pod 不会再被维持
- `Service` 只剩一个空入口

所以可以把 `Service` 理解成：

`入口本身仍然存在，但后面已经没有可转发的 Pod 了。`

## 九、重新部署应用

为了继续后面的实验，再执行一次：

```bash
kubectl apply -f nginx-demo.yaml
kubectl get deployments
kubectl get pods
```

确保环境重新回到：

- 2 个可用 Pod
- 1 个可访问的 `NodePort` Service

## 十、滚动更新是什么

Kubernetes 不会粗暴地先把所有旧 Pod 全删掉，再一次性起新 Pod。

更常见的默认方式是：

`滚动更新`

也就是：

- 逐步替换旧 Pod
- 逐步拉起新 Pod
- 尽量保持服务不中断

这也是 `Deployment` 最重要的能力之一。

## 十一、执行一次正常的滚动更新

执行：

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
kubectl rollout status deployment/nginx-deployment
kubectl get pods
kubectl rollout history deployment/nginx-deployment
```

这组命令分别在做什么：

### 1. `kubectl set image`

把 `Deployment` 中容器 `nginx` 的镜像改成 `nginx:1.25`。

### 2. `kubectl rollout status`

观察这次滚动发布是否成功完成。

### 3. `kubectl get pods`

查看新旧 Pod 替换过程是否已经结束。

### 4. `kubectl rollout history`

查看这个 `Deployment` 的发布历史版本。

如果发布成功，通常会看到：

- 新 Pod 已全部就绪
- 旧 Pod 已被替换掉
- `rollout status` 返回 successfully rolled out

## 十二、故意制造一次失败发布

为了真正理解回滚，再故意把镜像改成一个不存在的标签：

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:not-exist
kubectl rollout status deployment/nginx-deployment
kubectl get pods
kubectl describe pod <new-pod-name>
kubectl logs <new-pod-name>
```

这时很可能会看到：

- 发布卡住
- 新 Pod 处于 `ErrImagePull` 或 `ImagePullBackOff`
- `rollout status` 一直等待

这里最关键的是要区分两个观察点：

### 1. `kubectl logs`

如果容器根本没启动起来，`logs` 往往帮不了太多。

### 2. `kubectl describe pod`

这时最关键的信息通常在 `Events` 里，例如：

- 拉取镜像失败
- 镜像标签不存在
- 仓库返回错误

所以失败发布时，要优先看：

`describe + Events`

## 十三、回滚：把服务恢复到上一个可用版本

执行：

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment
kubectl get pods
kubectl rollout history deployment/nginx-deployment
```

这一步通常会看到：

- Deployment 再次开始滚动替换
- 失败的新 Pod 被替换掉
- 应用恢复到上一个可用版本

这里最值得记住的一点是：

`回滚不是把 revision 数字减回去，而是基于旧版本配置再触发一次新的发布。`

所以 rollout history 中的 revision 通常仍然是：

- 不断递增

## 十四、这一篇最值得掌握的 6 个结论

1. `kubectl get` 负责快速看对象状态。
2. `kubectl describe` 更适合看详细状态和事件。
3. `kubectl logs` 看的是容器日志，不一定能解释所有问题。
4. 删除 `Pod` 时，如果 `Deployment` 还在，Pod 会被自动补回。
5. 删除 `Deployment` 后，这组 `Pod` 就不会再被维持。
6. `rollout` 负责平滑发布新版本，`undo` 负责回到上一个可用版本。

## 十五、这一篇最值得记住的一句话

如果只记一句话，最值得记住的是：

`Kubernetes 里的 Deployment 不只是“把 Pod 跑起来”，而是持续维持副本状态、负责版本替换，并在失败发布时支持回滚恢复。`

## 十六、下一步要学什么

完成这一篇之后，下一步最自然的主题就是：

`ConfigMap`

因为前面已经学会了：

- 对象怎么部署
- 对象怎么观察
- 应用怎么更新和回滚

接下来就要进入另一个非常关键的问题：

`应用配置应该放在哪里，而不是硬编码在镜像里？`
