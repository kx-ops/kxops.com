+++
date = '2026-04-26T22:39:53+08:00'
draft = false
title = 'K3s安装文档'
+++

国内用户，可以使用以下方法加速安装:  
> curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh |INSTALL_K3S_EXEC='--flannel-backend=none --disable-network-policy'   INSTALL_K3S_MIRROR=cn sh -

普通用户使用kubectl  
```yaml
mkdir -p ~/.kube  
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config  
sudo chown $(id -u):$(id -g) ~/.kube/config  
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc  
source ~/.bashrc "
```

查看k3s运行状态  
> systemctl status k3s  

查看日志   
> journalctl -u k3s -f

解决国内无法拉到镜相的问题 将 /etc/rancher/k3s/registries.yaml 中的 endpoint 替换为以下仍在维护的源：  
Docker / Kubernetes 国内镜像加速源配置

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.1panel.live"
      - "https://docker.m.daocloud.io"
      - "https://hub.rat.dev"
      - "https://docker.nju.edu.cn"
  registry.k8s.io:
    endpoint:
      - "https://registry.lank8s.cn"
  gcr.io:
    endpoint:
      - "https://gcr.lank8s.cn"
  k8s.gcr.io:
    endpoint:
      - "https://gcr.nju.edu.cn"
  ghcr.io:
    endpoint:
      - "https://ghcr.nju.edu.cn"
  quay.io:
    endpoint:
      - "https://quay.nju.edu.cn"
```