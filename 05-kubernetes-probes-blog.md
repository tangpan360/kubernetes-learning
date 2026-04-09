# Kubernetes 入门学习笔记 05：理解 readinessProbe、livenessProbe 与 startupProbe

@[toc]

## 问题背景

前面已经学会了：

- `Deployment`
- `Service`
- `Endpoints`
- `ConfigMap`
- `Secret`
- `Namespace`

这时会自然遇到几个新的问题：

- `Pod` 只要启动了，就一定代表能对外提供服务吗？
- 如果应用启动很慢，怎么避免流量过早进入？
- 如果容器进程还活着，但程序其实已经卡死了，Kubernetes 怎么知道该不该重启？
- 为什么有时会看到 `READY 0/1`
- 为什么有时会看到 `RESTARTS` 一直增加，甚至进入 `CrashLoopBackOff`

这些问题，本质上都和：

`Probe`

有关。

这一篇的目标就是把探针体系真正讲清楚，并建立下面这层完整理解：

`startupProbe` 负责启动期保护，`readinessProbe` 决定能不能接流量，`livenessProbe` 决定容器是否需要被重启。

## 一、先记住三种 Probe 的分工

### 1. readinessProbe

它回答的问题是：

`这个实例现在能不能接流量？`

如果探针失败，通常会表现为：

- `READY` 变成 `0/1`
- Pod 暂时不进入 `Service` 的 `Endpoints`
- `Service` 不会把流量转发给它

但通常不会因为 readiness 失败就直接重启容器。

### 2. livenessProbe

它回答的问题是：

`这个容器是不是已经坏了，需要被重启？`

如果探针失败，通常会表现为：

- 容器被重启
- `RESTARTS` 增加
- 严重时进入 `CrashLoopBackOff`

### 3. startupProbe

它回答的问题是：

`这个应用启动流程到底有没有完成？`

它的核心作用是：

`给慢启动应用一个启动保护期，在启动真正完成之前，先不要让 livenessProbe 过早误杀。`

所以这一篇最值得先记住的一句话是：

`startup 管启动期，readiness 管接流量，liveness 管重启。`

## 二、本文使用的示例文件

为了方便直接复现，本文统一使用下面这些示例文件：

```text
k8s-demo/
  probe-readiness-demo.yaml
  probe-exec-readiness-demo.yaml
  probe-liveness-demo.yaml
  probe-startup-bad-demo.yaml
  probe-startup-good-demo.yaml
```

本文中的实验都在 `dev` 命名空间中进行。  
如果当前集群里还没有 `dev` 命名空间，可以先执行：

```bash
kubectl create namespace dev
```

如果提示 `AlreadyExists`，说明已经存在，可以直接继续。

## 三、readinessProbe：为什么 Pod 启动了却还不能接流量

很多初学者第一次看到下面这种状态会困惑：

- Pod 已经 `Running`
- 但 `READY` 却还是 `0/1`

这通常说明：

`容器进程已经跑起来了，但应用还没有真正准备好接收请求。`

也就是说：

- 进程启动
- 不等于
- 业务已就绪

这正是 `readinessProbe` 要解决的问题。

## 四、HTTP 类型的 readinessProbe

下面内容保存为 `probe-readiness-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: readiness-demo
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:
        - name: web
          image: python:3.11-slim
          command:
            - sh
            - -c
            - "sleep 15 && python -m http.server 8080"
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 2
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-demo-service
  namespace: dev
spec:
  selector:
    app: readiness-demo
  ports:
    - port: 80
      targetPort: 8080
```

这份 YAML 最关键的地方是：

```yaml
command:
  - sh
  - -c
  - "sleep 15 && python -m http.server 8080"
```

它表示：

- 容器先启动
- 但先睡 15 秒
- 15 秒后才真正启动 HTTP 服务

而探针这边会持续检查：

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 8080
```

也就是说，Kubernetes 会不断去请求：

- `http://<pod-ip>:8080/`

如果请求成功，才认为这个 Pod ready。

保存后执行：

```bash
cd k8s-demo
kubectl apply -f probe-readiness-demo.yaml
kubectl get pods -n dev -w
```

你应该能观察到：

- 刚启动时 `READY 0/1`
- 过十几秒后才变成 `READY 1/1`

再执行：

```bash
kubectl get endpoints -n dev
kubectl describe service readiness-demo-service -n dev
```

这时通常会看到：

- 刚开始 `Endpoints` 为空
- readiness 通过后，`Endpoints` 才出现 Pod IP

这说明：

`Pod 是否进入 Service 后端，不只看它有没有运行，还要看它是否 ready。`

## 五、不是 HTTP 服务怎么办

很多服务并不是网页接口，例如：

- Redis
- 某些 TCP 服务
- 某些后台任务型容器
- 某些没有对外 HTTP 健康检查接口的程序

这时 `readinessProbe` 也仍然可以使用。

Kubernetes 常见的探针方式有 3 种：

1. `httpGet`
2. `tcpSocket`
3. `exec`

也就是说，判断 ready 的关键不是：

`它是不是网页服务`

而是：

`有没有一个可靠信号，能说明这个实例现在已经可以处理业务请求。`

## 六、exec 类型的 readinessProbe

下面内容保存为 `probe-exec-readiness-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exec-readiness-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: exec-readiness-demo
  template:
    metadata:
      labels:
        app: exec-readiness-demo
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - "sleep 15; touch /tmp/ready; sleep 3600"
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - test -f /tmp/ready
            periodSeconds: 2
---
apiVersion: v1
kind: Service
metadata:
  name: exec-readiness-demo-service
  namespace: dev
spec:
  selector:
    app: exec-readiness-demo
  ports:
    - port: 80
      targetPort: 8080
```

这里的核心逻辑是：

- 容器先睡 15 秒
- 15 秒后创建 `/tmp/ready`
- 探针不断执行：

```sh
test -f /tmp/ready
```

如果文件存在：

- 命令退出码是 `0`
- readinessProbe 通过

如果文件不存在：

- 命令退出码不是 `0`
- readinessProbe 失败

保存后执行：

```bash
kubectl apply -f probe-exec-readiness-demo.yaml
kubectl get pods -n dev -w
```

你应该能看到：

- 刚启动时 `READY 0/1`
- 过十几秒后变成 `READY 1/1`

再验证：

```bash
kubectl exec -it deploy/exec-readiness-demo -n dev -- sh
ls /tmp
test -f /tmp/ready && echo "ready file exists"
exit
```

如果看到：

- `/tmp/ready`
- `ready file exists`

就说明这次不是 HTTP 探针让它 ready，而是：

`exec 探针通过容器内部命令退出码判断实例是否满足就绪条件。`

所以可以把这句话记住：

`exec 类型的 readinessProbe 会在容器内部反复执行指定命令，用退出码判断“当前是否满足就绪条件”；退出码为 0 表示 ready，非 0 表示暂时不 ready。`

## 七、livenessProbe：为什么容器会被自动重启

前面两种 readiness 示例解决的是：

`现在能不能接流量`

但有一种情况 readiness 解决不了：

- 容器进程还在
- 程序其实已经卡死了
- 不能再正常工作

这时 Kubernetes 需要解决的问题就变成了：

`这个容器是不是已经坏了，需要重启？`

这就是 `livenessProbe` 的职责。

## 八、最小 livenessProbe 示例

下面内容保存为 `probe-liveness-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liveness-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: liveness-demo
  template:
    metadata:
      labels:
        app: liveness-demo
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - "touch /tmp/alive; sleep 10; rm -f /tmp/alive; sleep 3600"
          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - test -f /tmp/alive
            initialDelaySeconds: 0
            periodSeconds: 2
```

这份 YAML 的设计非常直观：

1. 容器启动时先创建 `/tmp/alive`
2. 10 秒后删掉它
3. 进程本身并不退出
4. livenessProbe 持续执行：

```sh
test -f /tmp/alive
```

文件存在时：

- 探针通过

文件被删掉后：

- 探针失败
- Kubernetes 会重启容器

保存后执行：

```bash
kubectl apply -f probe-liveness-demo.yaml
kubectl get pods -n dev -w
```

你通常会看到：

- 刚开始 `RESTARTS 0`
- 过一会儿 `RESTARTS` 增加

再执行：

```bash
kubectl describe pod -n dev -l app=liveness-demo
```

通常会看到类似事件：

```text
Warning  Unhealthy  ...  Liveness probe failed
Normal   Killing    ...  Container worker failed liveness probe, will be restarted
```

这一步最容易误解的一点是：

`liveness 失败时，通常重启的是容器，不是整个 Pod。`

所以会出现：

- `Pod` 名字不变
- `Pod AGE` 持续增长
- 但 `RESTARTS` 不断增加

## 九、为什么看起来不像“10 秒一次重启”

在上面的例子里，文件确实是 10 秒后被删掉，但这不等于你在 `kubectl get pods` 里看到：

`正好每 10 秒重启一次`

原因有两个：

### 1. AGE 是 Pod 总年龄，不是单次容器运行时长

例如：

```text
liveness-demo-68fd67f78b-zjjqh   1/1   Running   3 (1s ago)   2m20s
```

这里的：

- `3 (1s ago)` 表示容器已经重启 3 次，最近一次发生在 1 秒前
- `2m20s` 表示这个 Pod 从创建到现在已经存在了 2 分 20 秒

不是说：

`这次容器运行了 2 分 20 秒才失败`

### 2. 反复失败后会进入 backoff

如果容器一直重启失败，Kubernetes 不会无限制地立刻重新拉起，而是会进入：

`CrashLoopBackOff`

这意味着：

- 重启间隔会越来越长
- 避免高频重启把节点打爆

所以你看到：

- `RESTARTS` 持续增加
- 状态在 `Running` 和 `CrashLoopBackOff` 间切换

这恰恰说明：

`Kubernetes 正在自动恢复，但恢复一直失败。`

## 十、出现一直重启时，管理员通常怎么发现和排查

如果业务里出现这种现象，最常见的提醒信号就是：

### 1. `kubectl get pods`

重点看：

- `STATUS`
- `RESTARTS`

如果看到：

- `CrashLoopBackOff`
- 重启次数不断增加

这已经是第一层异常提醒。

### 2. `kubectl describe pod`

重点看：

- `Events`
- probe failed
- backoff restarting failed container

### 3. `kubectl logs`

要看当前容器日志：

```bash
kubectl logs <pod-name> -n dev
```

更关键的是看上一次失败前的日志：

```bash
kubectl logs <pod-name> -n dev --previous
```

这个命令非常重要，因为很多崩溃信息只出现在上一个容器实例里。

### 4. 监控与告警

真实生产环境通常还会结合：

- `Prometheus`
- `Alertmanager`
- Grafana
- 平台告警系统

去盯：

- Pod restart 次数
- `CrashLoopBackOff`
- 探针失败次数
- Pod 长时间不 ready

然后再通过：

- 飞书
- 钉钉
- 邮件
- PagerDuty

等方式提醒管理员。

所以更准确地说：

`Kubernetes 会尽量自动恢复，但如果一直恢复失败，它会通过状态、事件、日志和监控信号把问题持续暴露出来，等管理员处理根因。`

## 十一、startupProbe：为什么慢启动应用会被 liveness 误杀

到这里就会碰到另一个很真实的问题：

如果一个应用本身启动就很慢，比如：

- Java 服务冷启动慢
- 模型加载慢
- 缓存预热慢
- 初始化依赖慢

那就有可能出现：

1. 应用还没真正启动完
2. `livenessProbe` 已经开始检查
3. 探针失败
4. 容器被重启
5. 又没等启动完成就再次被杀

这类问题的根因不是应用已经坏了，而是：

`根本没等它启动完。`

这就是 `startupProbe` 要解决的。

## 十二、没有 startupProbe 的坏例子

下面内容保存为 `probe-startup-bad-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: startup-bad-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: startup-bad-demo
  template:
    metadata:
      labels:
        app: startup-bad-demo
    spec:
      containers:
        - name: web
          image: python:3.11-slim
          command:
            - sh
            - -c
            - "sleep 30 && python -m http.server 8080"
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

这份配置里，应用需要：

- `sleep 30`

才会真正启动 HTTP 服务。

但 `livenessProbe` 却在：

- 5 秒后就开始检查

这就很容易导致：

- 应用还没起来
- liveness 已经开始判失败
- 容器被提前重启

保存后执行：

```bash
kubectl apply -f probe-startup-bad-demo.yaml
kubectl get pods -n dev -w
kubectl describe pod -n dev -l app=startup-bad-demo
```

通常会看到：

- 启动不稳定
- `RESTARTS` 增加
- 甚至进入 `CrashLoopBackOff`

## 十三、加上 startupProbe 的正确版本

下面内容保存为 `probe-startup-good-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: startup-good-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: startup-good-demo
  template:
    metadata:
      labels:
        app: startup-good-demo
    spec:
      containers:
        - name: web
          image: python:3.11-slim
          command:
            - sh
            - -c
            - "sleep 30 && python -m http.server 8080"
          ports:
            - containerPort: 8080
          startupProbe:
            httpGet:
              path: /
              port: 8080
            periodSeconds: 5
            failureThreshold: 10
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 2
---
apiVersion: v1
kind: Service
metadata:
  name: startup-good-demo-service
  namespace: dev
spec:
  selector:
    app: startup-good-demo
  ports:
    - port: 80
      targetPort: 8080
```

这份配置最关键的地方是：

```yaml
startupProbe:
  httpGet:
    path: /
    port: 8080
  periodSeconds: 5
  failureThreshold: 10
```

它大致表示：

- 每 5 秒检查一次
- 最多允许失败 10 次

也就是给了应用大约 50 秒的启动保护窗口。

而当前服务只需要：

- 30 秒

所以它就有足够时间完成启动，不会被 liveness 过早误杀。

保存后执行：

```bash
kubectl apply -f probe-startup-good-demo.yaml
kubectl get pods -n dev -w
kubectl describe pod -n dev -l app=startup-good-demo
kubectl get endpoints -n dev
```

通常会看到：

- 启动前一段时间还没 ready
- 但不会像坏版本那样频繁重启
- 启动完成后稳定变成 `1/1`

## 十四、startupProbe 到底和 liveness 是什么关系

这里最容易搞混的一点是：

`startupProbe 不是一个普通替代品，而是启动阶段的独立保护探针。`

可以把它理解成：

### 第一阶段：启动期

先由：

- `startupProbe`

负责判断：

`这个应用到底有没有成功启动起来`

只要 `startupProbe` 还没通过：

- `livenessProbe` 不会按正常方式接管
- `readinessProbe` 也不会按正常方式放行流量

如果 `startupProbe` 在允许的失败次数内成功了：

- 才进入后续阶段

如果它一直失败：

- 就会由 `startupProbe` 自己判定失败并触发容器重启

### 第二阶段：运行期

启动成功后，再分别由：

- `readinessProbe`
- `livenessProbe`

各自处理：

- 能不能接流量
- 要不要重启容器

所以这一步最值得记住的一句话是：

`startupProbe 先决定“应用有没有成功启动起来”，只有它通过之后，readiness 和 liveness 才开始发挥各自正常作用。`

## 十五、三种 Probe 的最终对照

### 1. startupProbe

回答：

`应用启动流程完成了吗？`

主要解决：

- 慢启动应用被过早误杀

### 2. readinessProbe

回答：

`这个实例现在能不能接流量？`

主要解决：

- 流量是否该进入这个 Pod
- 是否进入 `Endpoints`

### 3. livenessProbe

回答：

`这个容器是不是已经坏了，需要重启？`

主要解决：

- 卡死
- 假活着
- 死锁
- 长时间不自愈的异常状态

所以最适合记忆的三句话是：

- `startupProbe`：启动期还没结束前，先别误杀我
- `readinessProbe`：我现在能不能接流量
- `livenessProbe`：我现在是不是已经坏了，需要重启

## 十六、这一篇最值得掌握的 8 个结论

1. `Running` 不等于应用已经可以接流量。
2. `readinessProbe` 决定 Pod 是否进入 `Service` 的后端列表。
3. `readinessProbe` 失败通常是不接流量，不一定重启容器。
4. `exec` 类型探针通过容器内部命令退出码判断是否通过。
5. `livenessProbe` 失败通常会重启容器，而不是直接新建 Pod。
6. `Pod AGE` 是 Pod 总年龄，不是单次容器运行时长。
7. 容器反复失败时可能进入 `CrashLoopBackOff`，重启间隔会越来越长。
8. `startupProbe` 用来保护慢启动应用，避免它在启动完成前就被 liveness 误杀。

## 十七、这一篇最值得记住的一句话

如果只记一句话，最值得记住的是：

`Kubernetes 的探针体系里，startupProbe 负责保护启动期，readinessProbe 负责决定能不能接流量，livenessProbe 负责决定容器是否需要被重启。`

## 十八、下一步要学什么

完成这一篇之后，下一步最自然的主题就是：

- `requests`
- `limits`
- `资源调度`

因为到这里为止，已经把：

- 服务启动
- 流量接入
- 容器重启
- 探针保护

这一整条“应用健康”主线跑通了。

接下来很自然就会进入另一个同样重要的问题：

`Kubernetes 怎么决定一个 Pod 能用多少 CPU 和内存，以及资源不足时会发生什么。`
