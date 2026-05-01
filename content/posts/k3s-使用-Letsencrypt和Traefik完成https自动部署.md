+++
date = '2026-05-01T16:50:13+08:00'
draft = false
title = 'k8s使用 Letsencrypt 和 Traefik 实现 https 自动部署'
+++


k3s 使用 Letsencrypt 和 Traefik 完成 https 入口部署
### 内容提要
本文介绍 cert-manager 插件的安装，之后以一个简单的 web 服务部署为例，演示 https 服务的部署过程。

安装 cert-manager

首先创建 cert-manager 所需的命名空间
```bash
kubectl create namespace cert-manager  
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml
```
如果没有报错，稍等片刻查看 cert-manager 运行正常，就可以继续下一步了：

```bash
 kubectl get pods --namespace cert-manager
NAME                                     READY   STATUS    RESTARTS   AGE
cert-manager-6575b47cb5-fcbt5            1/1     Running   0          32s
cert-manager-cainjector-b6555495-wrvlx   1/1     Running   0          32s
cert-manager-webhook-787d9fc46-v248j     1/1     Running   0          32s
```

部署 Issuing Certificates#

下面给出一个示例的 letsencrypt.yml 配置，替换其中的邮箱 即可快速查看配置。
~~~yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: git@kxops.com 
    privateKeySecretRef:
      name: prod-issuer-account-key
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: traefik
        selector: {}
~~~
部署，并查看部署描述：

```bash
kubectl apply -f letsencrypt.yml
kubectl describe clusterissuer letsencrypt
```
~~~yaml
Name:         letsencrypt-prod
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         ClusterIssuer
Metadata:
  Creation Timestamp:  2026-05-01T08:10:52Z
  Generation:          1
  Resource Version:    215991
  UID:                 0d36506c-1920-4daf-950f-a9151fcdd4de
Spec:
  Acme:
    Email:  git@kxops.com
    Private Key Secret Ref:
      Name:  prod-issuer-account-key
    Server:  https://acme-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Class:  traefik
      Selector:
Status:
  Acme:
    Last Private Key Hash:  wtuqXcQxJR9qtcw72FMDiWdgCoi18/6rh021MZpsfYg=
    Last Registered Email:  git@kxops.com
  Conditions:
    Last Transition Time:  2026-05-01T08:10:54Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
~~~~
看到 Ready 说明一切正常，可以继续下一步了。

部署 Web 程序#
在这里就以 nginx 为例，该容器对外暴露 80 端口。

首先为此次部署准备一个命名空间 nginx 
```bash 
kubectl create namespace nginx 
```
之后编写 deployment 配置文件：

```bash 
cat deployment.yml
 ```
~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
  namespace: nginx 
  labels:
    app: nginx
spec:
  replicas: 1
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
        resources: 
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "128Mi"
~~~
部署到集群并查看状态：
```bash
kubectl apply -f deployment.yml
kubectl get deployment -n nginx
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
rancher-logo-app   1/1     1            1           29m
```
如果一切正常就可以继续下一步。

接下来部署一个 service 资源，为了下一步的 ingress 作准备：
~~~yaml
$ cat service.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx  
spec:
  selector:
    app: nginx-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
~~~
惯例，部署并查看状态：
```bash
kubectl apply -f service.yml
kubectl get service -n nginx
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.43.102.65   <none>        80/TCP    75s
```

如果一切正常，就可以开始部署入口( ingress )了。

```bash 
cat ingress.yml
```
~~~yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
    name: nginx-ingress
    namespace: nginx
spec:
  tls:
    - secretName: nginx-kxops-com-dev-tls
      hosts:
        - nginx.kxops.com
  rules:
  - host: nginx.kxops.com
    http:
      paths:
        - pathType: Prefix
          path: /
          backend:
            service:
              name: nginx-service
              port:
                number: 80
~~~
这个 ingress 会将流量路由到对应 service 的80 端口，之后进入对应的 pod 中，部署并查看一下吧：
```bash
kubectl apply -f ingress.yml
kubectl get ingress -n nginx
NAME                        CLASS     HOSTS             ADDRESS   PORTS     AGE
nginx-ingress               traefik   nginx.kxops.com             80, 443   4m17s
```


由于使用了 cert-manager 证书资源，该插件会自动完成证书的认证、部署和续签，看一下证书状态：
```bash
kubectl -n nginx describe certificate
```
~~~yaml
Name:         nginx-kxops-com-dev-tls
Namespace:    nginx
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2026-05-01T08:39:25Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  nginx-ingress
    UID:                   d1094fc4-c84d-418d-a2f9-2f1199ea4b83
  Resource Version:        221162
  UID:                     248fff4e-71e5-4172-a58c-58d818d3dbcd
Spec:
  Dns Names:
    nginx.kxops.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  nginx-kxops-com-dev-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:        2026-05-01T08:39:25Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      False
    Type:                        Ready
    Last Transition Time:        2026-05-01T08:39:25Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      True
    Type:                        Issuing
  Next Private Key Secret Name:  nginx-kxops-com-dev-tls-7d8cp
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    28m   cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  28m   cert-manager  Stored new private key in temporary Secret resource "nginx-kxops-com-dev-tls-7d8cp"
  Normal  Requested  28m   cert-manager  Created new CertificateRequest resource "nginx-kxops-com-dev-tls-1"
  Normal  Issuing    28m   cert-manager  The certificate has been successfully issued
~~~
可以看到一切正常，现在使用 https 就可以正常访问服务了。

大功告成