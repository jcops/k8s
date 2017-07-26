# 部署kubernetes node节点

kubernetes node 节点包含如下组件：

+ Flanneld： 省略，参照之前部署的文档
+ Docker1.12.5： 省略，参照之前部署的文档
+ kubelet
+ kube-proxy

## 目录和文件

我们再检查一下三个节点上，经过前几步操作已经生成的配置文件

``` bash
# #master节点：
# ls /etc/kubernetes/ssl
admin-key.pem  admin.pem  ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  kubernetes-key.pem  kubernetes.pem
# ls /etc/kubernetes/
apiserver  bootstrap.kubeconfig  config  controller-manager  kubelet.kubeconfig  kube-proxy.kubeconfig  scheduler  ssl  token.csv
# #node节点：
# ls /etc/kubernetes/ssl
admin-key.pem  admin.pem  ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  kubernetes-key.pem  kubernetes.pem
# ls /etc/kubernetes/
bootstrap.kubeconfig  config  kubelet.kubeconfig  kube-proxy.kubeconfig  ssl token.csv
```



### 下载最新的 kubelet 和 kube-proxy 二进制文件

``` bash
# wget https://dl.k8s.io/v1.6.0/kubernetes-server-linux-amd64.tar.gz #可忽略，因之前已下载过
# tar -xzvf kubernetes-server-linux-amd64.tar.gz
# cd kubernetes
# cp -r ./server/bin/{kube-proxy,kubelet,kubectl,kubefed} /usr/bin/
```
## 安装和配置 kubelet

kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper cluster 角色(role)，
然后 kubelet 才能有权限创建认证请求(certificate signing requests)：

``` bash
$ cd /etc/kubernetes
$ kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
clusterrolebinding "kubelet-bootstrap" created
```

+ `--user=kubelet-bootstrap` 是在 `/etc/kubernetes/token.csv` 文件中指定的用户名，同时也写入了 `/etc/kubernetes/bootstrap.kubeconfig` 文件；
+ 只需要在其中一台node节点上执行即可


### 创建 kubelet 的service配置文件

文件位置`/etc/systemd/system/kubelet.serivce`。

```ini
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBELET_API_SERVER \
	    $KUBELET_ADDRESS \
	    $KUBELET_PORT \
	    $KUBELET_HOSTNAME \
	    $KUBE_ALLOW_PRIV \
	    $KUBELET_POD_INFRA_CONTAINER \
	    $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
完整 unit 见 [kubelet.service](./systemd/kubelet.service)

+ 注意：需要先创建/var/lib/kubelet目录，不然稍后启动kubelet会报如下错误：
 ```
 Failed at step CHDIR spawning /usr/bin/kubelet: No such file or directory
 ```
kubelet的配置文件`/etc/kubernetes/kubelet`。其中的IP地址更改为你的每台node节点的IP地址。

``` bash
###
## kubernetes kubelet (minion) config
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=192.168.1.29"
#
## The port for the info server to serve on
#KUBELET_PORT="--port=10250"
#
## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=192.168.1.29"
#
## location of the api-server
KUBELET_API_SERVER="--api-servers=http://192.168.1.19:8080"
#
## pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
#
## Add your own!
KUBELET_ARGS="--cgroup-driver=systemd --cluster-dns=10.254.0.2 --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --require-kubeconfig --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local. --hairpin-mode promiscuous-bridge --serialize-image-pulls=false"

```

+ `--address` 不能设置为 `127.0.0.1`，否则后续 Pods 访问 kubelet 的 API 接口时会失败，因为 Pods 访问的 `127.0.0.1` 指向自己而不是 kubelet；
+ 如果设置了 `--hostname-override` 选项，则 `kube-proxy` 也需要设置该选项，否则会出现找不到 Node 的情况；
+ `--experimental-bootstrap-kubeconfig` 指向 bootstrap kubeconfig 文件，kubelet 使用该文件中的用户名和 token 向 kube-apiserver 发送 TLS Bootstrapping 请求；
+ 管理员通过了 CSR 请求后，kubelet 自动在 `--cert-dir` 目录创建证书和私钥文件(`kubelet-client.crt` 和 `kubelet-client.key`)，然后写入 `--kubeconfig` 文件；
+ 建议在 `--kubeconfig` 配置文件中指定 `kube-apiserver` 地址，如果未指定 `--api-servers` 选项，则必须指定 `--require-kubeconfig` 选项后才从配置文件中读取 kube-apiserver 的地址，否则 kubelet 启动后将找不到 kube-apiserver (日志中提示未找到 API Server），`kubectl get nodes` 不会返回对应的 Node 信息;
+ `--cluster-dns` 指定 kubedns 的 Service IP(可以先分配，后续创建 kubedns 服务时指定该 IP)，`--cluster-domain` 指定域名后缀，这两个参数同时指定后才会生效；

完整配置 见 [kubelet](./config/kubelet)

### 启动kublet

``` bash
# systemctl daemon-reload
# systemctl enable kubelet
3 systemctl start kubelet
# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet Server
   Loaded: loaded (/etc/systemd/system/kubelet.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2017-07-25 13:11:37 CST; 2s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 22236 (kubelet)
   CGroup: /system.slice/kubelet.service
           ├─22236 /usr/bin/kubelet --logtostderr=true --v=0 --api-servers=http://192.168.1.121:8080 --address=192.168.1.122 --host...
           └─22259 journalctl -k -f
...
```

### 通过 kublet 的 TLS 证书请求

kubelet 首次启动时向 kube-apiserver 发送证书签名请求，必须通过后 kubernetes 系统才会将该 Node 加入到集群

+ 注意：如果kubelet是使用的master节点上生成的那个token.csv来请求认证的，master就会自动通过认证请求直接加入集群，就不需要手动来通过csr请求

手动查看csr请求
>在master上查看未授权的 CSR 请求

``` bash
$ kubectl get csr
NAME        AGE       REQUESTOR           CONDITION
csr-2b308   4m        kubelet-bootstrap   Pending
$ kubectl get nodes
No resources found.
```

手动通过 CSR 请求

``` bash
$ kubectl certificate approve csr-2b308
certificatesigningrequest "csr-2b308" approved
$ kubectl get nodes
NAME            STATUS    AGE       VERSION
192.168.1.122   Ready     10m       v1.6.2
192.168.1.123   Ready     13s       v1.6.2
```
+ 注意：我这里使用的token.csv是和master上生成的那个是一致的，所以是自动认证csr请求的

查看自动生成了kubelet的公私钥

``` bash
# ls -l /etc/kubernetes/ssl/kubelet*
-rw-r--r-- 1 root root 1115 Jul 25 13:22 ssl/kubelet.crt
-rw------- 1 root root 1679 Jul 25 13:22 ssl/kubelet.key

```

## 配置 kube-proxy

**创建 kube-proxy 的service配置文件**

文件路径`/etc/systemd/system/kube-proxy.service`

```ini
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/bin/kube-proxy \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_MASTER \
	    $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

完整 unit 见 [kube-proxy.service](./systemd/kube-proxy.service)

``` bash
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--bind-address=192.168.1.122 --hostname-override=192.168.1.122 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"

```

+ `--hostname-override` 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则；
+ kube-proxy 根据 `--cluster-cidr` 判断集群内部和外部流量，指定 `--cluster-cidr` 或 `--masquerade-all` 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；
+ `--kubeconfig` 指定的配置文件嵌入了 kube-apiserver 的地址、用户名、证书、秘钥等请求和认证信息；
+ 预定义的 RoleBinding `cluster-admin` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限；

完整 配置 见 [proxy](./config/proxy)

### 启动 kube-proxy

``` bash
# systemctl daemon-reload
# systemctl enable kube-proxy
# systemctl start kube-proxy
# systemctl status kube-proxy
● kube-proxy.service - Kubernetes Kube-Proxy Server
   Loaded: loaded (/etc/systemd/system/kube-proxy.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2017-07-25 13:38:32 CST; 5s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 23097 (kube-proxy)
   CGroup: /system.slice/kube-proxy.service
           └─23097 /usr/bin/kube-proxy --logtostderr=true --v=0 --master=http://192.168.1.121:8080 --bind-address=192.168.1.122 --..
```
## 验证测试

我们创建一个niginx的service试一下集群是否可用。

```bash
# kubectl run nginx --replicas=2 --labels="run=load-balancer-example" --image=index.tenxcloud.com/docker_library/nginx:1.9.0  --port=80
deployment "nginx" created
# #通过日志可以看到pods已经被调度到以下节点
Jul 25 13:40:46 localhost kube-scheduler: I0725 13:40:46.810065   21421 event.go:217] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-4240121780-rhx4g", UID:"ced5ffeb-70fb-11e7-878e-00163e0006d3", APIVersion:"v1", ResourceVersion:"66394", FieldPath:""}): type: 'Normal' reason: 'Scheduled' Successfully assigned nginx-4240121780-rhx4g to 192.168.1.122
Jul 25 13:40:46 localhost kube-scheduler: I0725 13:40:46.810166   21421 event.go:217] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-4240121780-c9tvm", UID:"ced5fc8b-70fb-11e7-878e-00163e0006d3", APIVersion:"v1", ResourceVersion:"66393", FieldPath:""}): type: 'Normal' reason: 'Scheduled' Successfully assigned nginx-4240121780-c9tvm to 192.168.1.123
# #映射服务器端口
# kubectl expose deployment nginx --type=NodePort --name=example-service
service "example-service" exposed
# kubectl describe svc example-service
Name:			example-service
Namespace:		default
Labels:			run=load-balancer-example
Annotations:		<none>
Selector:		run=load-balancer-example
Type:			NodePort
IP:			10.254.242.15
Port:			<unset>	80/TCP
NodePort:		<unset>	31164/TCP
Endpoints:		172.30.59.2:80,172.30.99.2:80
Session Affinity:	None
Events:			<none>
# #可以看到在node节点上已经对外开放了31164端口
tcp6       0      0 :::31164                :::*                    LISTEN      23097/kube-proxy    
# # 在node节点上可以访问pod集群ip测试
# curl "10.254.242.15:80"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

访问`192.168.1.122:31164`或`192.168.1.123:31164`都可以得到nginx的页面。

