# k8s-demo

### 安装 docker
```
$ brew install docker
```

```
$ docker --version
Docker version 19.03.5, build 633a0ea
```

### Docker 速成
```
# Create html
$ mkdir k8s-demo && cd k8s-demo
$ echo '<h1>Hello Docker!</h1>' > index.html

# Create dockerfile
$ cat Dockerfile
FROM nginx
COPY index.html /usr/share/nginx/html

# Build image
$ docker build -t k8s-demo:0.1 .

# Run the docker
$ docker run --name k8s-demo -d -p 8080:80 k8s-demo:0.1
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
fd255a632cf6        k8s-demo:0.1        "nginx -g 'daemon of…"   3 seconds ago       Up 2 seconds        0.0.0.0:8080->80/tcp   k8s-demo

$ curl http://localhost:8080
<h1>Hello Docker!</h1>

$ docker stop k8s-demo
$ docker rmi k8s-demo
```

### 安装 Kubernetes
[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
[minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)


Minikube 启动时会自动配置 kubectl，把它指向 Minikube 提供的 Kubernetes API 服务。可以用下面的命令确认：
```
$ kubectl config current-context
minikube
```

### Kubernetes 架构简介
典型的 Kubernetes 集群包含一个 master 和很多 node。Master 是控制集群的中心，node 是提供 CPU、内存和存储资源的节点。Master 上运行着多个进程，包括面向用户的 API 服务、负责维护集群状态的 Controller Manager、负责调度任务的 Scheduler 等。每个 node 上运行着维护 node 状态并和 master 通信的 kubelet，以及实现集群网络服务的 kube-proxy。

作为一个开发和测试的环境，Minikube 会建立一个有一个 node 的集群，用下面的命令可以看到：
```
$ kubectl get nodes             
NAME       STATUS   ROLES    AGE    VERSION
minikube   Ready    master   134m   v1.17.2
```

### 部署一个单实例服务
我们先尝试像文章开始介绍 Docker 时一样，部署一个简单的服务。Kubernetes 中部署的最小单位是 pod，而不是 Docker 容器。实时上 Kubernetes 是不依赖于 Docker 的，完全可以使用其他的容器引擎在 Kubernetes 管理的集群中替代 Docker。在与 Docker 结合使用时，一个 pod 中可以包含一个或多个 Docker 容器。但除了有紧密耦合的情况下，通常一个 pod 中只有一个容器，这样方便不同的服务各自独立地扩展。

Minikube 自带了 Docker 引擎，所以我们需要重新配置客户端，让 docker 命令行与 Minikube 中的 Docker 进程通讯
```
$ eval $(minikube docker-env)
```
在运行上面的命令后，再运行 docker image ls 时只能看到一些 Minikube 自带的镜像，就看不到我们刚才构建的 docker-demo:0.1 镜像了。所以在继续之前，要重新构建一遍我们的镜像，这里顺便改一下名字，叫它 k8s-demo:0.1。
```
$ docker build -t k8s-demo:0.1 .
```

创建一个叫 pod.yml 的定义文件：
```
$ cat pod.yml   
apiVersion: v1
kind: Pod
metadata:
  name: k8s-demo
  labels:
    app: k8s-demo
spec:
  containers:
    - name: k8s-demo
      image: k8s-demo:0.1
      ports:
        - containerPort: 80
```

这里定义了一个叫 k8s-demo 的 Pod，使用刚才构建的 k8s-demo:0.1 镜像。这个文件也告诉 Kubernetes 容器内的进程会监听 80 端口。然后把它跑起来
```
$ kubectl create -f pod.yml 
```
kubectl 把这个文件提交给 Kubernetes API 服务，然后 Kubernetes Master 会按照要求把 Pod 分配到 node 上。用下面的命令可以看到这个新建的 Pod
```
$ kubectl get pods         
NAME       READY   STATUS    RESTARTS   AGE
k8s-demo   1/1     Running   0          6s

$ kubectl describe pods | grep Labels
Labels:       app=k8s-demo
```
因为镜像在本地，并且这个服务也很简单，所以运行 kubectl get pods 的时候 STATUS 已经是 running。要是使用远程镜像（比如 Docker Hub 上的镜像），你看到的状态可能不是 Running，就需要再等待一下。

虽然这个 pod 在运行，但是无法像之前测试 Docker 时一样用浏览器访问它运行的服务的。可以理解为 pod 都运行在一个内网，我们无法从外部直接访问。要把服务暴露出来，我们需要创建一个 Service。Service 的作用有点像建立了一个反向代理和负载均衡器，负责把请求分发给后面的 pod。

创建一个 Service 的定义文件 svc.yml：
```
$ cat svc.yml                        
apiVersion: v1
kind: Service
metadata:
  name: k8s-demo-svc
  labels:
    app: k8s-demo
spec:
  selector:
    app: k8s-demo
  type: NodePort
  ports:
    - port: 80
      nodePort: 30050
```

创建 service 
```
$ kubectl create -f svc.yml          
service/k8s-demo-svc created
```
用下面的命令可以得到暴露出来的 URL，在浏览器里访问，就能看到我们之前创建的网页了
```
$ minikube service k8s-demo-svc --url
http://192.168.64.6:30050
```

### 横向扩展、滚动更新、版本回滚
在这一节，我们来实验一下在一个高可用服务的生产环境会常用到的一些操作。在继续之前，先把刚才部署的 pod 删除（但是保留 service，下面还会用到）：
```
$ kubectl delete pod k8s-demo        
pod "k8s-demo" deleted
```

在正式环境中我们需要让一个服务不受单个节点故障的影响，并且还要根据负载变化动态调整节点数量，所以不可能像上面一样逐个管理 pod。Kubernetes 的用户通常是用 Deployment 来管理服务的。一个 deployment 可以创建指定数量的 pod 部署到各个 node 上，并可完成更新、回滚等操作。

首先创建一个定义文件 deployment.yml：
```
$ cat deployment.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-deployment
spec:
  selector:
    matchLabels:
      app: k8s-demo
  replicas: 10
  template:
    metadata:
      labels:
        app: k8s-demo
    spec:
      containers:
        - name: k8s-demo-pod
          image: k8s-demo:0.1
          ports:
            - containerPort: 80
```

注意开始的 apiVersion 和之前不一样，因为 Deployment API 没有包含在 v1 里，replicas: 10 指定了这个 deployment 要有 10 个 pod，后面的部分和之前的 pod 定义类似。提交这个文件，创建一个 deployment：
```
$ kubectl create --validate -f deployment.yml
deployment.apps/k8s-demo-deployment created
```
用下面的命令可以看到这个 deployment 的副本集（replica set），有 10 个 pod 在运行。
```
$ kubectl get rs                             
NAME                             DESIRED   CURRENT   READY   AGE
k8s-demo-deployment-7c4cf5fbbf   10        10        10      20s
```

假设我们对项目做了一些改动，要发布一个新版本。这里作为示例，我们只把 HTML 文件的内容改一下, 然后构建一个新版镜像 k8s-demo:0.2：
```
$ echo '<h1>Hello Kubernetes!</h1>' > index.html
$ docker build -t k8s-demo:0.2 .
```
然后更新 deployment.yml
```
$ cat deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-deployment
spec:
  selector:
    matchLabels:
      app: k8s-demo
  replicas: 10
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: k8s-demo
    spec:
      containers:
        - name: k8s-demo-pod
          image: k8s-demo:0.2
          ports:
            - containerPort: 80
```
这里有两个改动，第一个是更新了镜像版本号 image: k8s-demo:0.2，第二是增加了 minReadySeconds: 10 和 strategy 部分。新增的部分定义了更新策略：minReadySeconds: 10 指在更新了一个 pod 后，需要在它进入正常状态后 10 秒再更新下一个 pod；maxUnavailable: 1 指同时处于不可用状态的 pod 不能超过一个；maxSurge: 1 指多余的 pod 不能超过一个。这样 Kubernetes 就会逐个替换 service 后面的 pod。运行下面的命令开始更新
```
$ kubectl apply -f deployment.yml --record=true
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
deployment.apps/k8s-demo-deployment configured
```
这里的 --record=true 让 Kubernetes 把这行命令记到发布历史中备查。这时可以马上运行下面的命令查看各个 pod 的状态：
```
$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
k8s-demo-deployment-545967bfcb-7msb9   1/1     Running   0          2m20s
k8s-demo-deployment-545967bfcb-8lh7l   1/1     Running   0          2m34s
k8s-demo-deployment-545967bfcb-8sjpd   1/1     Running   0          3m1s
k8s-demo-deployment-545967bfcb-b962r   1/1     Running   0          3m1s
k8s-demo-deployment-545967bfcb-c8rqx   1/1     Running   0          2m20s
k8s-demo-deployment-545967bfcb-ktgwm   1/1     Running   0          2m48s
k8s-demo-deployment-545967bfcb-pw2hr   1/1     Running   0          3m13s
k8s-demo-deployment-545967bfcb-t77ps   1/1     Running   0          2m48s
k8s-demo-deployment-545967bfcb-tx8zb   1/1     Running   0          3m13s
k8s-demo-deployment-545967bfcb-x2mvz   1/1     Running   0          2m34s
```
从 AGE 列就能看到有一部分 pod 是刚刚新建的，有的 pod 则还是老的。下面的命令可以显示发布的实时状态：
```
$ kubectl rollout status deployment k8s-demo-deployment
deployment "k8s-demo-deployment" successfully rolled out
```
由于输入得比较晚，发布已经快要结束，所以只有三行输出。下面的命令可以查看发布历史，因为第二次发布使用了 --record=true 所以可以看到用于发布的命令
```
$ kubectl rollout history deployment k8s-demo-deployment
deployment.apps/k8s-demo-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply --filename=deployment.yml --record=true
```
这时如果刷新浏览器，就可以看到更新的内容「Hello Kubernetes!」。假设新版发布后，我们发现有严重的 bug，需要马上回滚到上个版本，可以用这个很简单的操作：
```
$ kubectl rollout undo deployment k8s-demo-deployment --to-revision=1
deployment.apps/k8s-demo-deployment rolled back
```
Kubernetes 会按照既定的策略替换各个 pod，与发布新版本类似，只是这次是用老版本替换新版本：
```
$ kubectl rollout status deployment k8s-demo-deployment
Waiting for deployment "k8s-demo-deployment" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "k8s-demo-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "k8s-demo-deployment" successfully rolled out
```
在回滚结束之后，刷新浏览器就可以确认网页内容又改回了「Hello Docker!」。

### 延伸阅读
https://www.dongwm.com/post/use-kubernetes-1/
https://1byte.io/developer-guide-to-docker-and-kubernetes/