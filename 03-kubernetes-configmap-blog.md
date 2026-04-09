# Kubernetes 入门学习笔记 03：用 ConfigMap 管理配置，并理解环境变量与文件挂载

@[toc]

## 问题背景

在已经学会：

- `Deployment`
- `Service`
- `kubectl get`
- `kubectl logs`
- `kubectl rollout`

这些基础能力之后，下一个非常自然的问题就是：

`应用的配置应该放在哪里？`

如果把配置直接写死在镜像里，会遇到几个明显问题：

- 开发、测试、生产环境配置不同
- 改一个配置可能就要重新构建镜像
- 同一个镜像很难复用到不同环境
- 配置和业务代码混在一起，不方便维护

所以 Kubernetes 提供了一个专门管理普通配置的对象：

`ConfigMap`

这一篇要解决的核心问题有 4 个：

1. `ConfigMap` 到底是什么
2. 怎么把 `ConfigMap` 注入到 Pod 里
3. 为什么有时在 Pod 里看到的是环境变量，有时又会变成文件
4. 这种挂载出来的“卷”是不是持久化存储

## 一、ConfigMap 是什么

`ConfigMap` 可以理解成：

`Kubernetes 中专门存放普通配置数据的对象。`

它通常保存的是：

- 环境标识
- 应用标题
- 普通配置项
- 配置文件内容

例如：

- `APP_ENV=dev`
- `APP_TITLE=hello-k8s`
- `nginx.conf`
- `application.yaml`

这里最重要的一点是：

`ConfigMap 存的是配置，不是业务数据。`

也就是说，它不是拿来存：

- 用户订单
- 数据库表
- 上传文件
- 长期业务数据

这些真正需要长期保存的数据，后面要靠持久化存储体系来解决，而不是靠 `ConfigMap`。

## 二、本文使用的示例文件

本文继续使用前两篇中的 `k8s-demo/` 目录，并新增一个 `ConfigMap` 文件：

```text
k8s-demo/
  kind-config.yaml
  configmap-demo.yaml
  nginx-demo.yaml
```

其中 `configmap-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  APP_ENV: "dev"
  APP_TITLE: "hello-k8s"
  app-message.txt: |
    hello from configmap
    this file is mounted inside the pod
```

当前使用的 `nginx-demo.yaml` 内容如下：

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
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: nginx-config
                  key: APP_ENV
            - name: APP_TITLE
              valueFrom:
                configMapKeyRef:
                  name: nginx-config
                  key: APP_TITLE
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
      volumes:
        - name: config-volume
          configMap:
            name: nginx-config
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

## 三、先创建 ConfigMap

先进入目录并创建 `ConfigMap`：

```bash
cd k8s-demo
kubectl apply -f configmap-demo.yaml
kubectl get configmap
kubectl describe configmap nginx-config
```

正常情况下会看到：

- `nginx-config` 已经创建成功
- `Data` 中能看到 `APP_ENV`、`APP_TITLE`、`app-message.txt`

这里可以先建立一个最基本的认识：

`ConfigMap 本身只是集群中的一个配置对象，此时它还没有自动进入 Pod。`

也就是说，先创建 `ConfigMap` 不等于容器已经能使用这些配置。  
必须在 `Pod` 模板里显式声明“怎么使用它”。

## 四、方式一：把 ConfigMap 注入成环境变量

很多应用会直接从环境变量里读取配置，所以第一种最常见方式就是：

`env + configMapKeyRef`

例如：

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: nginx-config
        key: APP_ENV
  - name: APP_TITLE
    valueFrom:
      configMapKeyRef:
        name: nginx-config
        key: APP_TITLE
```

这段配置的含义是：

- 从 `nginx-config` 这个 `ConfigMap` 中取值
- 把 `APP_ENV` 注入为容器环境变量
- 把 `APP_TITLE` 注入为容器环境变量

更新 Deployment 后，可以执行：

```bash
kubectl apply -f nginx-demo.yaml
kubectl get pods
kubectl exec -it deploy/nginx-deployment -- sh
env | grep APP_
echo "$APP_ENV"
echo "$APP_TITLE"
exit
```

如果配置生效，通常会看到：

- `APP_ENV=dev`
- `APP_TITLE=hello-k8s`

这一层最重要的结论是：

`当 ConfigMap 通过 env 注入时，它在容器里表现为环境变量。`

## 五、方式二：把 ConfigMap 挂载成文件

不是所有应用都从环境变量读配置。  
很多程序更习惯从文件读配置，比如：

- `nginx.conf`
- `application.yaml`
- `config.ini`
- `settings.json`

所以 Kubernetes 还提供了另一种方式：

`把 ConfigMap 当成 volume 挂载到容器中`

对应配置是：

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/config
    readOnly: true

volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

这段配置的含义是：

- 定义一个名为 `config-volume` 的卷
- 这个卷的数据来源不是磁盘，而是 `ConfigMap`
- 把这个卷挂载到容器里的 `/etc/config`

更新后可以验证：

```bash
kubectl apply -f nginx-demo.yaml
kubectl exec -it deploy/nginx-deployment -- sh
ls /etc/config
cat /etc/config/APP_ENV
cat /etc/config/APP_TITLE
cat /etc/config/app-message.txt
exit
```

通常会看到：

- `/etc/config/APP_ENV`
- `/etc/config/APP_TITLE`
- `/etc/config/app-message.txt`

并且这些文件内容分别就是 `ConfigMap` 里的 value。

例如：

- 文件 `APP_ENV` 的内容是 `dev`
- 文件 `APP_TITLE` 的内容是 `hello-k8s`
- 文件 `app-message.txt` 的内容是多行文本

这里最关键的一句话是：

`ConfigMap 并不是天然就是文件，而是当它被作为 volume 挂载时，Kubernetes 会把每个 key 投影成一个文件。`

## 六、为什么同一个 ConfigMap 既像变量，又像文件

很多初学者会在这里困惑：

`明明是同一个 ConfigMap，为什么有时能用 env 读到，有时又在目录里看到文件？`

原因并不复杂：

`因为这是两种不同的注入方式。`

### 1. 通过 `env` 注入

结果是：

- 容器里会出现环境变量
- 不会因为这一步自动生成文件

### 2. 通过 `volume` 挂载

结果是：

- `ConfigMap` 中的 key 会出现在挂载目录里
- 每个 key 对应一个文件

所以你现在这个示例里，之所以两种现象同时出现，是因为：

`nginx-demo.yaml 同时使用了 env 和 volumes 两套机制。`

也正因为如此，你才会同时看到：

- `APP_ENV=dev`
- `/etc/config/APP_ENV`

这不是冲突，也不是重复报错，而是因为你同时走了两条不同路径。

## 七、这和 Docker 里传环境变量有什么区别

在 Docker 里，最常见的环境变量写法是：

```bash
docker run -e APP_ENV=dev -e APP_TITLE=hello nginx
```

这时容器里得到的是：

- 环境变量

而不是：

- 自动生成的配置文件

也就是说，Docker 里普通的 `-e` 传参，一般只会让程序通过：

- `env`
- `$APP_ENV`
- `os.getenv()`

这些方式去读取变量，而不会自动在容器里创建：

- `/etc/config/APP_ENV`

如果 Docker 容器里出现了文件，通常是因为：

- 镜像本身就包含这个文件
- 或者使用了挂载目录

所以这一层最值得记住的区分是：

`Docker 里环境变量默认只是环境变量；Kubernetes 里的 ConfigMap 只有在作为 volume 挂载时，才会以文件形式出现在容器里。`

## 八、这种 volume 是不是持久化卷

这一点非常容易和前面学过的 Docker volume 混在一起，所以必须单独讲清楚。

这里的：

```yaml
volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

虽然名字也叫 `volume`，但它的作用不是：

- 长期保存业务数据
- 存数据库文件
- 存用户上传内容
- 跨应用长期持久化

它更准确的角色是：

`把配置数据投影到 Pod 文件系统中的一种挂载方式。`

也就是说，它是：

- 配置卷
- 临时投影
- 面向容器使用的配置入口

而不是传统意义上的“持久化数据卷”。

所以对于这个问题，最准确的回答是：

`它是 Pod 使用的配置型 volume，不是后面存业务数据的持久化 volume。`

## 九、ConfigMap 一般适合放什么，不适合放什么

### 适合放的内容

- 普通环境变量
- 应用标题
- 功能开关
- 非敏感配置
- 文本配置文件

### 不适合放的内容

- 密码
- token
- 证书私钥
- 数据库真实业务数据

这里先提前记住一句话：

`普通配置放 ConfigMap，敏感配置放 Secret。`

## 十、完整操作流程回顾

如果想从零再跑一遍，顺序可以按下面来：

```bash
cd k8s-demo
kubectl apply -f configmap-demo.yaml
kubectl apply -f nginx-demo.yaml
kubectl get configmap
kubectl get pods
kubectl exec -it deploy/nginx-deployment -- sh
env | grep APP_
ls /etc/config
cat /etc/config/APP_ENV
cat /etc/config/APP_TITLE
cat /etc/config/app-message.txt
exit
```

这条链路跑通后，你应该能同时验证两件事：

1. `ConfigMap` 可以被注入成环境变量
2. `ConfigMap` 也可以被挂载成文件

## 十一、这一篇最值得掌握的 6 个结论

1. `ConfigMap` 用来保存普通配置，不用来保存业务数据。
2. `ConfigMap` 本身先是集群对象，只有被 Pod 引用后，应用才能真正使用它。
3. 通过 `env` 注入时，配置会表现为环境变量。
4. 通过 `volume` 挂载时，`ConfigMap` 的 key 会被投影成文件。
5. `ConfigMap` 挂载出来的 volume 是配置卷，不是业务数据持久化卷。
6. 同一个 `ConfigMap` 可以同时用环境变量和文件两种方式注入。

## 十二、这一篇最值得记住的一句话

如果只记一句话，最值得记住的是：

`ConfigMap 是 Kubernetes 中管理普通配置的对象，它既可以注入成环境变量，也可以通过 volume 挂载成文件，但这类 volume 的本质是配置投影，不是持久化存储。`

## 十三、下一步要学什么

完成这一篇之后，下一步最自然的主题就是：

`Secret`

因为它和 `ConfigMap` 的使用方式非常相似，但处理的是：

- 密码
- token
- 证书
- 私密配置

也就是下一层必须建立的区分：

`为什么普通配置可以放 ConfigMap，而敏感信息不能直接放进去。`
