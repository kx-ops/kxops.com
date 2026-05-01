+++
date = '2026-04-27T22:44:35+08:00'
draft = false
title = 'Github ArgoCD K8s Gitops搭建流水线'
+++


全部示范代码在 https://github.com/kx-ops/thinkphp   
这是份部署 thinkphp 用的版本 镜相大小为 250M 
可以使用下面指令拉取
```bash
docker pull ghcr.io/kx-ops/thinkphp:91b2e9f9f579a7c9eb44e065b8696421339f4fed 
```
打包Debian 11 nginx unitd 1.3 + php 7.4.33 (非常老的版本)

流程
~~~ mermaid 
graph LR
    %% 节点定义
    Developer[代码提交]
    Github[Github Repository]
    GHA[Github Actions CI]
    GHCR[(Github Packages)]
    Manifest[k8s/deploy.yaml 更新]
    ArgoCD[ArgoCD]
    K8s[K8s 集群]

    %% 流程连接
    Developer -->|git push| Github
    Github -->|触发 Workflow| GHA
    GHA -->|构建镜相| GHCR
    GHA -->|更新镜相| Manifest
    Manifest -->|监听变化| ArgoCD
    ArgoCD -->|拉取/应用| K8s

    %% 样式美化
    style Github fill:#f9f,stroke:#333
    style GHCR fill:#bbf,stroke:#333
    style ArgoCD fill:#f96,stroke:#333
    style K8s fill:#7cf,stroke:#333
~~~

1 . 代码提交        ----- Github  
2 . 打包镜相CI      ----- 根据Dockerfile 打包 Github Packages 并更新k8s/deploy.yaml 文件中的images ID  
3 . 持续部署CD      ----- ArgoCD 监听 Github 仓库中的 k8s/deploy.yaml 变化 部署到K8s集群  

前提条件：
设置 GitHub 仓库  GitHub Personal Access Token（PAT）具有 write:packages 权限  
生成一个SSH KEY 用于argo CD 连github 并注入到github 用户 SSH中

代码结构
~~~ mermaid 
treeView-beta
"thinkphp"
        ".dockerignore"
        ".gitignore"
        ".github"
            "workflows"
                "main.yaml"
        "public"
        "k8s"
            "deploy.yaml"
        "Docker"
            "Dockerfile"
~~~ 

略去K8s 安装  
记录 ArgoCD 安装过程  

1. 创建命名空间
> kubectl create namespace argocd

2. 应用官方安装清单（stable 版本）
> kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

3. 暴露 Argo CD 服务
> kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

4. 获取初始密码并登录
> kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo "用户名: admin"

5. 进入 Argo CD   连接github 仓库 建应用
登入Argo CD Web 界面 

左侧菜单 → Settings → Repositories。
点击右上角 + Connect Repo。
选择连接方式：HTTPS：输入仓库 URL、Username、Password（PAT）。
SSH：输入 git@github.com:kx-ops/thinkphp.git，并粘贴私钥内容。
点击 Connect，测试通过后保存。

建应用 NEW APP  
填写：Application Name: my-app  
Project: default  
Sync Policy: Automated（开启自动同步）  
Repository URL: 你的部署 Git 仓库  
Path: k8s（或具体子目录）  
Cluster: https://kubernetes.default.svc  
Namespace: default  

点击 CREATE，然后 SYNC。

这样就即大功告成