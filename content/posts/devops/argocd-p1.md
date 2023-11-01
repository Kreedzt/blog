---
author: "Kreedzt"
title: "内网 k8s 中部署 argocd 问题记录"
date: "2023-11-01"
description: "内网 k8s 中部署 argocd 问题记录及解决方案"
tags: ["devops", "k8s", "traefik", "ingress", "argocd"]
draft: false
---

## ArgoCD 部署问题记录

### github 克隆问题

> 注意: 克隆仓库问题 log 不会显示在 GUI 界面中, 尽量使用 cli 操作

如遇到 ssh 克隆仓库时被服务器拒绝问题, 此问题多半是代理服务器导致的, 可以尝试使用 ssh over https 方案:

```shell
#旧
git@github.com:Kreedzt/argocd-demo.git
#新
ssh://git@ssh.github.com:443/Kreedzt/argocd-demo.git
```


### ingress 健康检查一直 processing 问题

参考:
- https://github.com/argoproj/argo-cd/issues/5620
- https://github.com/argoproj/argo-cd/issues/1704

这是 argocd 没有针对 ingress 做默认的健康检查兜底导致的, 若不想每个项目都改动, 可参考此回复解决

> https://github.com/argoproj/argo-cd/issues/1704#issuecomment-1605855853

修改 `argocd-cm` 的 configMap, 添加 `resource.customizations` 数据即可
```yaml
apiVersion: v1
data:
  resource.customizations: |
    networking.k8s.io/Ingress:
      health.lua: |
        hs = {}
        hs.status = "Healthy"
        return hs
 
kind: ConfigMap
metadata:
  name: argocd-cm
```