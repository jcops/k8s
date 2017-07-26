# 部署Flanneld网络


+ Flanneld：用于解决容器之间网络互通，这里我们要配置TLS认证。
+ Docker1.12.5：docker的安装很简单，这里也不说了。




## 配置Flanneld
+ 这里我们使用yum的方式部署Flanneld和docker
```
# yum install flannel docker -y
```
service配置文件`/etc/systemd/system/flanneld.service`

```ini
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/flanneld
EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=/usr/bin/flanneld-start $FLANNEL_OPTIONS
ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service

```

`/etc/sysconfig/flanneld`配置文件

```ini
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="https://192.168.1.121:2379,https://192.168.1.122:2379,https://192.168.1.123:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/k8s-ks/network"

# Any additional options that you want to pass
FLANNEL_OPTIONS="-etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem"

```

在FLANNEL_OPTIONS中增加TLS的配置

**在etcd中创建网络配置**

执行下面的命令为docker分配IP地址段

```shell
# etcdctl --endpoints=https://192.168.1.121:2379,https://192.168.1.122:2379,https://192.168.1.123:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  mkdir /k8s-ks/network
# etcdctl --endpoints=https://192.168.1.121:2379,https://192.168.1.122:2379,https://192.168.1.123:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  mk /k8s-ks/network/config "{ \"Network\": \"172.30.0.0/16\", \"SubnetLen\": 24, \"Backend\": { \"Type\": \"vxlan\" } }"
```

+ 注意：vxlan的性能损耗大约是40%～50%，如果将Type设置为host-gw，网络性能损耗只有10%左右，而配置没有什么不同，只是要保证kubernetes的所有node都在同一个二层网络中。
+ 注意：这两条语句只需要在其中一台机器上执行即可  

**启动flanneld**

在各节点上启动flanneld
```
# systemctl  enable flanneld
# systemctl  start flanneld
```
+ 启动flanneld后会在/run/flannel目录下生成subnet.env和docker文件
```
# ls /run/flannel/
docker  subnet.env
```
Flannel的[文档](https://github.com/coreos/flannel/blob/master/Documentation/running.md)中有写**Docker Integration**：

Docker daemon accepts `--bip` argument to configure the subnet of the docker0 bridge. It also accepts `--mtu` to set the MTU for docker0 and veth devices that it will be creating. Since flannel writes out the acquired subnet and MTU values into a file, the script starting Docker can source in the values and pass them to Docker daemon:

执行如下命令：
```
# source /run/flannel/subnet.env
```

Systemd users can use `EnvironmentFile` directive in the .service file to pull in `/run/flannel/subnet.env`

+ 如果你不是使用yum安装的flanneld，那么需要下载flannel github release中的tar包，解压后会获得一个**mk-docker-opts.sh**文件。

这个文件是用来`Generate Docker daemon options based on flannel env file`。

执行`./mk-docker-opts.sh -i`将会生成如下两个文件环境变量文件。

/run/flannel/subnet.env

```
FLANNEL_NETWORK=172.30.0.0/16
FLANNEL_SUBNET=172.30.46.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false
```

/run/docker_opts.env

```
DOCKER_OPT_BIP="--bip=172.30.46.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1450"
```
**启动docker**
```
# systemctl enable docker
# systemctl start docker #各节点上执行
# systemctl status docker
```

现在查询etcd中的内容可以看到：

```
ETCD_ENDPOINTS=https://192.168.1.121:2379,https://192.168.1.122:2379,https://192.168.1.123:2379 
# etcdctl --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  ls /k8s-ks/network/subnets
2017-07-25 09:57:47.969181 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
/k8s-ks/network/subnets/172.30.25.0-24
/k8s-ks/network/subnets/172.30.99.0-24
/k8s-ks/network/subnets/172.30.59.0-24

# etcdctl --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  get /k8s-ks/network/config
2017-07-25 10:03:01.516739 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
{ "Network": "172.30.0.0/16", "SubnetLen": 24, "Backend": { "Type": "vxlan" } }

#etcdctl --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  get /k8s-ks/network/subnets/172.30.0.0-24
2017-07-25 10:12:17.911371 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
{"PublicIP":"192.168.1.121","BackendType":"vxlan","BackendData":{"VtepMAC":"b2:bf:15:fb:f8:14"}}
# etcdctl --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  get /k8s-ks/network/subnets/172.30.99.0-24
2017-07-25 10:20:22.044716 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
{"PublicIP":"192.168.1.122","BackendType":"vxlan","BackendData":{"VtepMAC":"42:5d:f5:9c:72:ae"}}

# etcdctl --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  get /k8s-ks/network/subnets/172.30.59.0-24
2017-07-25 10:20:27.175817 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
{"PublicIP":"192.168.1.123","BackendType":"vxlan","BackendData":{"VtepMAC":"52:ad:8d:17:23:7a"}}

```
+ 可以使用ip addr查看网卡ip，比如如下：
+ 使用docker0的ip能互相ping通flanneld就算搭建完毕
```
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether b2:bf:15:fb:f8:14 brd ff:ff:ff:ff:ff:ff
    inet 172.30.25.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:c2:bd:a6:ae brd ff:ff:ff:ff:ff:ff
    inet 172.30.25.1/24 scope global docker0
       valid_lft forever preferred_lft forever
```
```
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether 42:5d:f5:9c:72:ae brd ff:ff:ff:ff:ff:ff
    inet 172.30.99.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:38:28:40:88 brd ff:ff:ff:ff:ff:ff
    inet 172.30.99.1/24 scope global docker0
       valid_lft forever preferred_lft forever

```
