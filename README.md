# kubernetes集群从入门到进阶

**FYI：本文档仅作为kubernetes安装文档，<u>持续更新中</u>，欢迎大家围观和贡献

本系列文档使用的二进制方式部署 `kubernetes` 集群的所有步骤，而不是使用 `kubeadm` 等自动化方式来部署集群，同时开启了集群的TLS安全认证；

在部署的过程中，将详细列出各组件的启动参数，给出配置文件，详解它们的含义和可能遇到的问题。

部署完成后，你将理解系统各组件的交互原理，进而能快速解决实际问题。

本文档主要适合于想通过一步步部署的方式来学习和了解系统配置、运行原理的人。


注：本文档中不包括docker和私有镜像仓库的安装。(后续可增加)

## 提供所有的配置文件

集群安装时所有组件用到的配置文件，包含在以下目录中：

- **config**： service主配置文件
- **yaml**： kubernetes相关模块的yaml文件
- **systemd** ：systemd init配置文件
- **images** ：相关教程图片
- **cfssl** ：ca证书生成工具
## 集群详情

+ Kubernetes 1.6.2
+ Docker  1.12.5（使用yum安装）
+ Etcd 3.1.5
+ Flanneld 0.7 vxlan 网络
+ TLS 认证通信 (所有组件，如 etcd、kubernetes master 和 node)
+ RBAC 授权
+ kublet TLS BootStrapping
+ kubedns、dashboard、heapster(influxdb、grafana)、EFK(elasticsearch、fluentd、kibana) 集群插件
+ 私有docker镜像仓库[harbor](github.com/vmware/harbor)（请自行部署，harbor提供离线安装包，直接使用docker-compose启动即可）(后续可增加)

## 步骤介绍

1. [集群环境及组件介绍](01-集群环境及组件介绍.md)
2. [创建TLS-CA证书及密钥](02-创建TLS-CA证书及密钥.md)
3. [创建高可用etcd集群](03-创建高可用etcd集群.md)
4. [创建kubeconfig认证文件](04-创建kubeconfig认证文件.md)
5. [创建kubectl-kubeconfig文件](05-创建kubectl-kubeconfig文件.md)
6. [搭建master集群](06-搭建master集群.md)
7. [部署Flanneld网络](07-部署Flanneld网络.md)
8. [部署node节点](08-部署node节点].md)
9. [部署配置kubedns插件](09-部署配置kubedns插件.md)
10. [部署配置dashboard](10-部署配置dashboard.md)
11. [部署Heapster插件](11-部署Heapster插件.md)
12. [部署EFK插件](12-部署EFK插件.md)


## 提醒

1. 由于启用了 TLS 双向认证、RBAC 授权等严格的安全机制，建议**从头开始部署**，而不要从中间开始，否则可能会认证、授权等失败！

