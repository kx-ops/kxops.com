+++
date = '2026-05-06T23:14:04+08:00'
draft = false
title = 'Argocd 简述及使用 上'
+++

ArgoCD 专为 Kubernetes 而生的，遵循声明式 GitOps 理念的持续部署工具。使用 Git 仓库更改时自动同步和部署应用程序。
 ![argocd 架构图](https://cdn.shortpixel.ai/spai/w_1248+q_lossless+ret_img+to_webp/www.apptio.com/wp-content/uploads/argocd-arch.png)

#### 官方说明
~~~
Why Argo CD? 为什么选择 ArgoCD 
1. Application definitions, configurations, and environments should be declarative and version controlled.
让构建的应用程序的定义、配置及环境应具备声明式特性，并使代码版本进行控制。
2. Application deployment and lifecycle management should be automated, auditable, and easy to understand.
定议应用程序的部署与生命周期管理应实现自动化、可审计，且易于理解。
~~~

Argo CD 可在指定的目标环境中自动部署所需的应用程序状态，应用程序部署可以在 Git 提交时跟踪对分支、标签的更新，或固定到清单的指定版本。


下面简单介绍下 Argo CD 中的几个主要组件：

一. API 服务：API 服务是一个 gRPC/REST 服务，它暴露了 Web UI、CLI 和 CI/CD 系统使用的接口，主要有以下几个功能：
1. 应用程序管理和状态报告
2. 执行应用程序操作（例如同步、回滚、用户定义的操作）
3. 存储仓库和集群凭据管理（存储为 K8S Secrets 对象）
4. 认证和授权给外部身份提供者
5. RBAC
6. Git webhook 事件的侦听器/转发器

二.  仓库服务：存储仓库服务是一个内部服务，负责维护保存应用程序清单 Git 仓库的本地缓存。当提供以下输入时，它负责生成并返回 Kubernetes 清单：
1. 存储 URL
2. revision 版本（commit、tag、branch）
3. 应用路径
4. 模板配置：参数、ksonnet 环境、helm values.yaml 等

三 . 应用控制器：应用控制器是一个 Kubernetes 控制器，它持续 watch 正在运行的应用程序并将当前的实时状态与所期望的目标状态（ repo 中指定的）进行比较。它检测应用程序的 OutOfSync 状态，并采取一些措施来同步状态，它负责调用任何用户定义的生命周期事件的钩子（PreSync、Sync、PostSync）。

### ArgoCD 功能
~~~
自动部署应用程序到指定的目标环境
支持多种配置管理/模板工具（Kustomize、Helm、Ksonnet、Jsonnet、plain-YAML）
能够管理和部署到多个集群
SSO 集成（OIDC、OAuth2、LDAP、SAML 2.0、GitHub、GitLab、Microsoft、LinkedIn）
用于授权的多租户和 RBAC 策略
回滚/随时回滚到 Git 存储库中提交的任何应用配置
应用资源的健康状况分析
自动配置检测和可视化
自动或手动将应用程序同步到所需状态
提供应用程序活动实时视图的 Web UI
用于自动化和 CI 集成的 CLI
Webhook 集成（GitHub、BitBucket、GitLab）
用于自动化的 AccessTokens
PreSync、Sync、PostSync Hooks，以支持复杂的应用程序部署（例如蓝/绿和金丝雀发布）
应用程序事件和 API 调用的审计
Prometheus 监控指标
用于覆盖 Git 中的 ksonnet/helm 参数
~~~

### 核心概念
~~~
Application：应用，一组由资源清单定义的 Kubernetes 资源，这是一个 CRD 资源对象
Application source type：用来构建应用的工具
Target state：目标状态，指应用程序所需的期望状态，由 Git 存储库中的文件表示
Live state：实时状态，指应用程序实时的状态，比如部署了哪些 Pods 等真实状态
Sync status：同步状态表示实时状态是否与目标状态一致，部署的应用是否与 Git 所描述的一样？
Sync：同步指将应用程序迁移到其目标状态的过程，比如通过对 Kubernetes 集群应用变更
Sync operation status：同步操作状态指的是同步是否成功
Refresh：刷新是指将 Git 中的最新代码与实时状态进行比较，弄清楚有什么不同
Health：应用程序的健康状况，它是否正常运行？能否为请求提供服务？
Tool：工具指从文件目录创建清单的工具，例如 Kustomize 或 Ksonnet 等
Configuration management tool：配置管理工具
Configuration management plugin：配置管理插件
~~~

### 安装非常简易 二行指令
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

下期写argocd 使用过程