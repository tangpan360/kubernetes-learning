# Kubernetes 入门学习笔记 04：理解 Secret、RBAC、Namespace 与 Service 的作用范围

@[toc]

## 问题背景

在已经学会：

- `Pod`
- `Deployment`
- `Service`
- `kubectl`
- `ConfigMap`

这些基础之后，下一步最容易出现的新问题其实不止一个，而是一串彼此关联的问题：

- 敏感配置为什么不能直接继续放在 `ConfigMap`
- 谁可以读取 `Secret`
- 为什么很多资源都要结合 `Namespace` 一起看
- 为什么同一个服务名在不同命名空间里行为不同
- 为什么 `Service` 背后的 Pod 变了，访问入口却还能稳定
- 一个 `Service` 后面有多个 Pod 时，流量到底怎么分发

这一篇就把这条主线一次串起来，目标是建立下面这层完整理解：

`Secret` 负责承载敏感配置，`RBAC` 控制哪些身份可以通过 Kubernetes API 读取资源，`Namespace` 决定资源和服务发现的作用范围，`Service` 则通过 `selector` 和 `Endpoints` 把流量转发到一组后端 Pod。

## 一、为什么有了 ConfigMap 还要有 Secret

前一篇已经讲过：

- `ConfigMap` 用来保存普通配置
- 可以注入成环境变量
- 也可以挂载成文件

但有一类数据不适合继续放进 `ConfigMap`：

- 数据库密码
- token
- API key
- 证书私钥

这类数据在 Kubernetes 里更适合放进：

`Secret`

最值得先记住的一句话是：

`ConfigMap 放普通配置，Secret 放敏感配置。`

## 二、本文使用的示例文件

为了方便直接复现，本文统一使用下面这些示例文件：

```text
k8s-demo/
  configmap-demo.yaml
  nginx-demo.yaml
  secret-demo.yaml
  rbac-demo.yaml
  namespace-config-secret-demo.yaml
  namespace-pod-config-demo.yaml
  namespace-demo.yaml
  echo-demo.yaml
```

后面的每个实验，都会对应使用其中一个文件。

## 三、最小 Secret 示例

下面内容保存为 `secret-demo.yaml`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_USER: YWRtaW4=
  DB_PASSWORD: MTIzNDU2
```

这里最容易误解的一点是：

`data` 里的值看起来像“加密后内容”，但实际上只是 base64 编码。`

例如：

- `admin -> YWRtaW4=`
- `123456 -> MTIzNDU2`

所以一定要分清：

`base64 不是加密，只是编码。`

保存后执行：

```bash
kubectl apply -f secret-demo.yaml
kubectl get secret app-secret -n default
```

## 四、为什么 Secret 不能简单理解成“保险箱”

很多初学者第一次看到 `Secret`，会自然以为：

`只要放进 Secret，就已经安全了。`

这个理解是不够准确的。

因为如果某个身份本来就有读取 `Secret` 的权限，那么它仍然可以通过 Kubernetes API 取到内容，再做 base64 解码。

例如：

```bash
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

这就意味着：

`Secret` 的安全重点不在“别人看不懂 base64”，而在“别人有没有权限读到这个 Secret 对象”。`

所以更准确的说法是：

`Secret 是 Kubernetes 中专门承载敏感配置的对象类型，但真正的安全关键在访问控制，而不是 base64 本身。`

## 五、Secret 注入 Pod 的两种常见方式

和 `ConfigMap` 一样，`Secret` 最常见也有两种注入方式：

1. 注入成环境变量
2. 挂载成文件

例如下面这段 `env` 配置：

```yaml
env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_USER
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_PASSWORD
```

而如果写成 `volume`：

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

再配合：

```yaml
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secret
    readOnly: true
```

那么容器里就会看到：

- `/etc/secret/DB_USER`
- `/etc/secret/DB_PASSWORD`

但无论是环境变量还是文件挂载，Pod 最终拿到的都是原始值，而不是那串 base64。

## 六、RBAC 到底在控制什么

既然 `Secret` 的关键不在编码形式，那真正应该限制的到底是什么？

答案就是：

`哪些身份可以通过 Kubernetes API 对哪些资源执行哪些动作。`

这正是 `RBAC` 的作用。

`RBAC` 全称是：

`Role-Based Access Control`

也就是：

`基于角色的访问控制。`

当前阶段最值得先分清 3 个对象：

### 1. Role

`Role` 定义一组权限规则，也就是：

`能做什么`

例如：

- 能读取 `ConfigMap`
- 能列出 `ConfigMap`
- 不能读取 `Secret`

### 2. ServiceAccount

`ServiceAccount` 可以先理解成：

`Pod 在 Kubernetes 里的身份`

也就是：

`是谁`

### 3. RoleBinding

`RoleBinding` 的作用是：

`把某个 Role 赋给某个身份`

也就是把：

- `能做什么`
- 和
- `是谁`

绑定起来。

所以可以把这三者记成一句话：

`Role 定义能做什么，ServiceAccount 表示是谁，RoleBinding 负责把权限赋给这个身份。`

## 七、最小 RBAC 示例：允许读 ConfigMap，不允许读 Secret

下面内容保存为 `rbac-demo.yaml`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reader-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
subjects:
  - kind: ServiceAccount
    name: reader-sa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: configmap-reader
```

这份配置表达的意思很直接：

- 创建一个身份 `reader-sa`
- 创建一组权限 `configmap-reader`
- 只允许它读取 `configmaps`
- 不给它读取 `secrets` 的权限

保存后执行：

```bash
kubectl apply -f rbac-demo.yaml
kubectl get sa -n default
kubectl get role -n default
kubectl get rolebinding -n default
```

验证时可以直接执行：

```bash
kubectl auth can-i get configmaps --as=system:serviceaccount:default:reader-sa
kubectl auth can-i get secrets --as=system:serviceaccount:default:reader-sa
```

如果结果是：

- `configmaps -> yes`
- `secrets -> no`

那就说明真正起作用的不是 `Secret` 的 base64，而是：

`RBAC 是否把读取 Secret 的权限放给这个身份。`

## 八、Kubernetes API 到底是什么

前面一直在说：

- 读取 `Secret`
- 读取 `ConfigMap`
- `RBAC` 控制 API 权限

这里就必须分清另一个核心概念：

`Kubernetes API`

它可以先理解成：

`整个集群的统一管理入口。`

你平时执行的这些命令：

```bash
kubectl get pods
kubectl apply -f nginx-demo.yaml
kubectl get secret app-secret -o yaml
```

并不是 `kubectl` 自己在管理集群。  
真正处理请求的是：

`Kubernetes API Server`

所以可以先把链路理解成：

```text
用户或程序
  ->
kubectl / 客户端
  ->
Kubernetes API Server
  ->
集群对象与控制器
```

因此更准确的一句话是：

`RBAC 控制的是哪些身份可以通过 Kubernetes API 对哪些资源执行哪些动作。`

## 九、Namespace 是什么

理解完 `Secret` 和 `RBAC` 后，下一个自然问题就是：

`这些资源到底活在哪个范围里？`

答案就是：

`Namespace`

它不是：

- 另一套独立集群
- 另一台机器
- 另一套 Python 依赖环境

它更像是：

`同一个 Kubernetes 集群里的逻辑分区。`

例如一个集群里可以同时有：

- `default`
- `dev`
- `test`
- `prod`

它解决的主要是：

- 资源组织
- 名字隔离
- 配置隔离
- 权限范围隔离

所以最值得记住的一句话是：

`Namespace 是集群内部的逻辑分区，不是物理隔离，也不是另一套独立 Kubernetes。`

## 十、为什么很多资源都要结合 Namespace 一起看

前面学过的这些对象，大多数时候都是 namespaced 资源：

- `Pod`
- `Deployment`
- `Service`
- `ConfigMap`
- `Secret`
- `ServiceAccount`
- `Role`
- `RoleBinding`

这意味着：

`资源名本身往往还不够完整，必须结合 Namespace 才能准确定位。`

例如：

- `default` 里的 `app-secret`
- `dev` 里的 `app-secret`

虽然名字一样，但它们是两个不同对象。

## 十一、在 dev 中创建自己的 ConfigMap 和 Secret

下面内容保存为 `namespace-config-secret-demo.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  APP_ENV: "dev"
  APP_NAME: "demo-in-dev"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: dev
type: Opaque
stringData:
  DB_USER: "devuser"
  DB_PASSWORD: "devpass"
```

这一步最重要的意义不是创建对象本身，而是建立一个非常关键的认知：

`不同 Namespace 里的 ConfigMap 和 Secret 可以同名，但它们不是同一个资源。`

如果当前集群里还没有 `dev` 命名空间，先执行：

```bash
kubectl create namespace dev
```

如果提示 `AlreadyExists`，说明这个命名空间已经存在，可以继续往下执行。

保存后再执行：

```bash
kubectl apply -f namespace-config-secret-demo.yaml
kubectl get configmaps -n dev
kubectl get secrets -n dev
```

## 十二、Pod 为什么默认只能用自己 Namespace 里的配置

下面内容保存为 `namespace-pod-config-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-reader
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-reader
  template:
    metadata:
      labels:
        app: app-reader
    spec:
      containers:
        - name: busybox
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: APP_NAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_NAME
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: DB_PASSWORD
```

这份 YAML 能正常工作的前提是：

- Pod 在 `dev`
- `ConfigMap` 在 `dev`
- `Secret` 也在 `dev`

而如果把其中某个引用改成：

- `nginx-config`

但这个对象只存在于 `default` 而不在 `dev`，那么 `dev` 里的 Pod 会报找不到配置对象。  
如果想复现这一点，可以先确保默认命名空间里已经有这个对象：

```bash
kubectl apply -f configmap-demo.yaml
kubectl get configmap nginx-config -n default
```

这说明：

`Pod 默认只能引用自己所在 Namespace 内的 ConfigMap 和 Secret。`

这一步很重要，因为它说明 `Namespace` 不只是查看范围，也会影响资源引用范围。

保存后执行：

```bash
kubectl apply -f namespace-pod-config-demo.yaml
kubectl get pods -n dev
kubectl exec -it deploy/app-reader -n dev -- sh
env | grep -E 'APP_|DB_'
exit
```

## 十三、为什么很多 kubectl 命令不写 -n 时总在 default

当执行：

```bash
kubectl get pods
```

如果不显式写 `-n`，那么 `kubectl` 默认会使用当前上下文里的默认命名空间，通常就是：

- `default`

这就是为什么：

```bash
kubectl get deployments
```

常常等价于：

```bash
kubectl get deployments -n default
```

所以 Namespace 学习里必须养成一个习惯：

`一旦资源是 namespaced 的，就先问自己：它在哪个 Namespace 里？`

## 十四、Service 在不同 Namespace 中怎么访问

前面已经知道：

- `Pod` 默认只能引用自己所在 Namespace 的配置

那 `Service` 呢？

答案是：

`Service 名称解析也默认优先在当前 Namespace 内查找。`

这一节会同时用到：

- `dev` 里的 `nginx-dev-service`
- `default` 里的 `nginx-service`

所以如果前面还没有创建这两个服务，可以先执行：

```bash
kubectl apply -f namespace-demo.yaml
kubectl apply -f configmap-demo.yaml
kubectl apply -f secret-demo.yaml
kubectl apply -f nginx-demo.yaml
```

例如在 `dev` 里的 Pod 内访问：

```bash
curl http://nginx-dev-service
```

这会默认解析成：

- `dev` 里的 `nginx-dev-service`

而如果想访问 `default` 里的服务，就需要带上命名空间，例如：

```bash
curl http://nginx-service.default
```

或者使用完整域名：

```bash
curl http://nginx-service.default.svc.cluster.local
```

所以最值得记住的一句话是：

`同 Namespace 内访问 Service 时通常可以直接用短服务名，跨 Namespace 访问时通常要带上目标 Namespace。`

## 十五、Service 为什么能稳定访问，而 Pod IP 却会变化

`Pod` 本身并不稳定：

- 可能被删掉
- 可能被重建
- Pod 名可能变
- Pod IP 也可能变

如果应用之间直接写死 Pod IP，会非常脆弱。

Kubernetes 之所以引入 `Service`，就是为了提供一个稳定入口。

这里最关键的一句话是：

`应用应该访问 Service，而不是直接依赖会变化的 Pod IP。`

## 十六、Service、selector、Endpoints 和 Pod 到底是什么关系

这是理解服务发现最核心的一条链。

### 1. Deployment

`Deployment` 负责：

- 定义期望副本数
- 创建 Pod
- 维持 Pod 数量

### 2. Pod

`Pod` 负责：

- 真正运行业务容器
- 带有 `label`
- 拥有自己的 Pod IP

### 3. Service

`Service` 负责：

- 提供稳定访问入口
- 通过 `selector` 选择一组 Pod

这里要特别注意：

`selector 选择的是 Pod 的 label，不是 Pod 名字。`

### 4. Endpoints

`Endpoints` 表示的是：

`当前这个 Service 后面实际可转发的 Pod IP:Port 列表。`

所以可以把完整链路理解成：

```text
Deployment
  -> 创建带 label 的 Pod
Service
  -> 用 selector 匹配 Pod label
Kubernetes
  -> 自动维护这组 Pod 的 Endpoints
客户端
  -> 访问 Service
  -> 流量再被转发到这些 Endpoints
```

## 十七、如果 selector 没匹配到 Pod，会发生什么

这里使用的示例文件是 `namespace-demo.yaml`。

例如下面这种情况：

- Pod label 是 `app=nginx-dev`
- Service selector 却写成 `app=nginx-dev-v2`

这时对象本身仍然可以存在，但会看到：

- `Service` 还在
- `Endpoints` 为空

例如：

```text
Selector: app=nginx-dev-v2
Endpoints: <none>
```

这说明：

`Service 是否有后端，不取决于 Pod 只是“存在”，而取决于 selector 是否真的选中了 Pod。`

为了先得到正常的后端服务，可以先执行：

```bash
kubectl apply -f namespace-demo.yaml
kubectl get pods -n dev -o wide
kubectl get endpoints -n dev
```

如果要观察 `selector` 不匹配的情况，可以临时把 `Service` 的 `selector` 改成 `app: nginx-dev-v2`，重新执行：

```bash
kubectl apply -f namespace-demo.yaml
kubectl describe service nginx-dev-service -n dev
kubectl get endpoints -n dev
```

观察完成后，再把 `selector` 改回 `app: nginx-dev`，继续后面的实验。

## 十八、多副本 Pod 为什么还能稳定工作

如果 `Deployment` 把副本数扩到 `3`，那么就会出现：

- 3 个 Pod
- 3 个不同 Pod IP
- 1 个 Service
- 一组包含 3 个地址的 `Endpoints`

当删除其中一个 Pod 时，通常会发生两件事：

1. `Deployment` 立刻补一个新的 Pod
2. `Endpoints` 自动更新成当前存活 Pod 的新地址

这说明：

`多副本 Service 的稳定性，不是因为单个 Pod 不会坏，而是因为 Kubernetes 会自动补副本并同步更新 Endpoints。`

## 十九、一个 Service 后面有多个 Pod 时，请求会怎么样

为了补充观察多后端流量分发，文中还使用了 `echo-demo.yaml` 作为辅助示例。

当一个 `Service` 背后挂了多个 `Endpoints`，请求并不会固定只落到某一个 Pod。

如果直接访问 `nginx-dev-service`，3 个 nginx Pod 默认返回的页面内容完全一样，这样很难看出请求到底落到了哪个后端。

所以在这个实验里，需要先手动把 3 个 Pod 的首页内容改成不同文本。

先获取 3 个 Pod 名：

```bash
kubectl get pods -n dev -l app=nginx-dev
```

然后分别执行：

```bash
kubectl exec -n dev nginx-dev-AAA -- sh -c 'echo pod-A > /usr/share/nginx/html/index.html'
kubectl exec -n dev nginx-dev-BBB -- sh -c 'echo pod-B > /usr/share/nginx/html/index.html'
kubectl exec -n dev nginx-dev-CCC -- sh -c 'echo pod-C > /usr/share/nginx/html/index.html'
```

把上面的：

- `nginx-dev-AAA`
- `nginx-dev-BBB`
- `nginx-dev-CCC`

替换成自己实际看到的 3 个 Pod 名。

这样做完之后，3 个 Pod 虽然仍然属于同一个 `Deployment` 和同一个 `Service`，但各自返回的页面内容已经变成了不同文本：

- `pod-A`
- `pod-B`
- `pod-C`

接着需要在集群内部的测试 Pod 中访问这个 `Service`，而不是在宿主机直接访问短服务名。  
如果还没有创建测试 Pod，先执行：

```bash
kubectl run curl-test -n dev --image=curlimages/curl:8.7.1 --restart=Never -- sleep 3600
```

然后进入它：

```bash
kubectl exec -it curl-test -n dev -- sh
```

再在容器里连续访问同一个服务名：

```bash
for i in 1 2 3 4 5 6 7 8 9 10; do curl -s http://nginx-dev-service; echo; done
```

之所以要在 `dev` 里的测试 Pod 中执行，是因为：

- `nginx-dev-service` 是集群内部的 `ClusterIP Service`
- 短服务名解析默认依赖集群内部 DNS
- 宿主机通常不能直接解析 `nginx-dev-service` 这种名字

如果看到类似结果：

```text
pod-B
pod-B
pod-C
pod-A
pod-C
pod-B
pod-C
pod-B
pod-C
pod-A
```

就说明：

`Service 确实把请求分发到了多个后端 Pod，而不是固定落到单个实例。`

这里还要额外记住一点：

`从肉眼观察结果看，返回顺序不一定严格按 A -> B -> C 完全轮询。`

但这并不影响核心结论，因为你已经明确看到了：

- 请求来自多个不同 Pod
- 而不是同一个 Pod 一直响应

## 二十、这一篇最值得掌握的 8 个结论

1. `Secret` 用来承载敏感配置，但 `base64` 不是加密。
2. `Secret` 的安全重点在访问控制，而不是编码形式。
3. `RBAC` 控制的是哪些身份可以通过 Kubernetes API 对哪些资源执行哪些动作。
4. `Role` 定义权限，`ServiceAccount` 表示身份，`RoleBinding` 负责绑定。
5. `Namespace` 是集群内部的逻辑分区，不是另一套独立集群。
6. `Pod` 默认只能引用自己所在 Namespace 里的 `ConfigMap` 和 `Secret`。
7. `Service` 通过 `selector` 选择带有特定 label 的 Pod，并维护成一组 `Endpoints`。
8. 一个 `Service` 后面可以有多个后端 Pod，请求会被分发到这些 Pod 上。

## 二十一、这一篇最值得记住的一句话

如果只记一句话，最值得记住的是：

`Kubernetes 里的 Secret 负责承载敏感配置，RBAC 决定谁能通过 Kubernetes API 读取资源，Namespace 决定资源和服务发现的作用范围，而 Service 则通过 selector 和 Endpoints 把请求稳定地转发到一组后端 Pod。`

## 二十二、下一步要学什么

完成这一篇之后，下一步最自然的主题就是：

- `探针`
- `资源限制`
- `Ingress`

因为到这里为止，已经把：

- 部署对象
- 配置对象
- 访问控制
- 命名空间
- 服务发现
- 基础流量分发

这条主线跑通了。

接下来就会进入另一个非常关键的问题：

`Kubernetes 怎么判断一个 Pod 真正可用，以及外部流量怎么更正式地进入集群。`
