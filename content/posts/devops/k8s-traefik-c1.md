---
author: "Kreedzt"
title: "内网 k8s 中使用 traefik 作为 ingress(一)"
date: "2023-10-06"
description: "在内网k8s集群中搭建 traefik ingress 并使其生效"
tags: ["devops", "docker", "traefik", "ingress", "nginx"]
draft: false
---

## 目标

- traefik 服务正常启动
- 部署 nginx 部署测速后可访问

### 环境准备

- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [helm](https://helm.sh/docs/intro/install/)

## 安装 traefik

参考 [官方文档](https://doc.traefik.io/traefik/getting-started/install-traefik/) 安装教程

### 注册仓库

```sh
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

### 部署

从 [helm chart 市场](https://artifacthub.io/packages/helm/traefik/traefik)下载一份默认值的 yaml 文件进行修改:

命名为 `traefick-values.yml`, 着重修改以下地方:
```yaml
ports:
  web:
    # 重定向 http -> https
    redirectTo: websecure
  websecure:
    tls:
      enabled: true

service:
  externalIPs:
    # 对于内网环境, 部署 traefik 会卡在 externalIP 为 pending 的状态, 此处需要显式声明 IP 地址
    - 192.168.x.x

# 作为默认的 ingress, 替换已有的 ingress
ingressClass:
  enabled: true
  isDefaultClass: true
```

执行命令以部署:

```sh
helm install traefik traefik/traefik --values traefik-values.yml
```

正常情况下的输出结果:
```txt
NAME: traefik
LAST DEPLOYED: Fri Oct  6 11:56:10 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Traefik Proxy v2.10.4 has been deployed successfully on default namespace !
```

部署完成后, 等待一段时间, 成功部署结果后用 `kubectl` 查询状态:
```sh
kubectl get all
```

正常情况下的输出如下, 应当有 pod / service / deployment 存在 traefik, service 的 type 为 `LoadBalancer`
```txt
NAME                          READY   STATUS    RESTARTS   AGE
pod/traefik-b767ddb99-r8vx4   1/1     Running   0          17m

NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
service/kubernetes   ClusterIP      10.43.0.1      <none>          443/TCP                      22h
service/traefik      LoadBalancer   10.43.84.235   192.168.2.149   80:31231/TCP,443:31900/TCP   17m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   1/1     1            1           17m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-b767ddb99   1         1         1       17m
zhao@master1:~/traefik$ kubectl get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/traefik-b767ddb99-r8vx4   1/1     Running   0          20m

NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
service/kubernetes   ClusterIP      10.43.0.1      <none>          443/TCP                      22h
service/traefik      LoadBalancer   10.43.84.235   192.168.2.149   80:31231/TCP,443:31900/TCP   20m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   1/1     1            1           20m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-b767ddb99   1         1         1       20m
```


## 部署 nginx 测试

编写部署文件 `nginx-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx # 部署名称
  namespace: nginx-test # 命名空间, 所有命名空间需要一致
  labels:
    app: nginx #应用名称
spec:
  selector:
    matchLabels:
      app: nginx # 应用名称
  replicas: 1 # 副本数
  template:
    metadata:
      labels:
        app: nginx # 应用名称
    spec:
      containers:
      - name: nginx # container 名称
        image: nginx:latest # docker 镜像
        ports:
        - containerPort: 80 # 暴露 80 端口以访问
---
apiVersion: v1
kind: Service
metadata:
  name:  nginx # 服务名称, 被入口配置引用
  namespace: nginx-test # 命名空间, 需要一致
spec:
  selector:
    app:  nginx
  type:  ClusterIP
  # ClusterIP 标明可以被任意 pod 访问
  ports:
  - name:  http
    port:  80 # 服务端口
---
apiVersion: networking.k8s.io/v1
# 入口配置
kind: Ingress
metadata:
  name: nginx
  namespace: nginx-test # 命名空间, 需要一致
spec:
  rules:
  - host: "nginx-test.k8s.io"  # 自定义域名
    http:
      paths:
      # 路由配置,  / 表示该域名下所有路径都指向服务
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx  # 服务名称
            port:
              number: 80  # 服务端口
```

使用 `kubectl` 部署:
```sh
kubectl apply -f nginx-deployment.yml
```

正常输出:
```txt
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-6b7f675859-js6zc   1/1     Running   0          21h

NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/nginx   ClusterIP   10.43.254.9   <none>        80/TCP    21h

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           21h

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-6b7f675859   1         1         1       21h
```

此时修改 `hosts`, 将自定义的域名指向 traefik 服务的 `EXTERNAL-IP`:
```hosts
192.168.2.149 nginx-test.k8s.io
```

通过浏览器访问, 应该正确重定向至https: `https://nginx-test.k8s.io`, 并显示 nginx 欢迎界面

![traefik 成功页面](../images/traefik-1.png)