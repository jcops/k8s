# 创建高可用 etcd 集群

kuberntes 系统使用 etcd 存储所有数据，本文档介绍部署一个三节点高可用 etcd 集群的步骤，这三个节点使用以下机器：

+ 192.168.1.121
+ 192.168.1.122
+ 192.168.1.123

## TLS 认证文件

需要为 etcd 集群创建加密通信的 TLS 证书，这里复用之前创建的 kubernetes 证书

``` bash
# cp ca.pem kubernetes-key.pem kubernetes.pem /etc/kubernetes/ssl #之前执行过cp *.pem /etc/kubernetes/ssl就忽略
```

+ kubernetes 证书的 `hosts` 字段列表中必须包含上面三台机器的 IP，否则后续证书校验会失败；

## 下载二进制文件

到 `https://github.com/coreos/etcd/releases` 页面下载最新版本的二进制文件

``` bash
# https://github.com/coreos/etcd/releases/download/v3.1.5/etcd-v3.1.5-linux-amd64.tar.gz
# tar -xvf etcd-v3.1.4-linux-amd64.tar.gz
# cp etcd-v3.1.4-linux-amd64/etcd* /usr/bin
```

## 创建 etcd 的 systemd unit 文件


``` bash
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/usr/bin/etcd \
--name=${ETCD_NAME} \
--data-dir=${ETCD_DATA_DIR} \
--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
--initial-cluster=${ETCD_INITIAL_CLUSTER}  \
--initial-cluster-state=${ETCD_INITIAL_CLUSTER_STATE} \
--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS}
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

+ 指定 `etcd` 的工作目录和数据目录为 `/var/lib/etcd` 需在启动服务前创建这个目录；


完整 unit 文件见：[etcd.service](./systemd/etcd.service)

配置文件在`/etc/etcd/etcd.conf`

```Ini
# [member]
ETCD_NAME=infra1
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.1.121:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.121:2379,http://127.0.0.1:2379"

# #[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.121:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.121:2379"
ETCD_INITIAL_CLUSTER="infra1=https://192.168.1.121:2380,infra2=https://192.168.1.122:2380,infra3=https://192.168.1.123:2380"

#[security]
ETCD_CERT_FILE="/etc/kubernetes/ssl/kubernetes.pem"
ETCD_KEY_FILE="/etc/kubernetes/ssl/kubernetes-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/kubernetes/ssl/kubernetes.pem"
ETCD_PEER_KEY_FILE="/etc/kubernetes/ssl/kubernetes-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_PEER_AUTO_TLS="true"

```
+ 为了保证通信安全，需要指定 etcd 的公私钥(cert-file和key-file)、Peers 通信的公私钥和 CA 证书(peer-cert-file、peer-key-file、peer-trusted-ca-file)、客户端的CA证书（trusted-ca-file）；
+ 创建 `kubernetes.pem` 证书时使用的 `kubernetes-csr.json` 文件的 `hosts` 字段**包含所有 etcd 节点的IP**，否则证书校验会出错；
+ `--initial-cluster-state` 值为 `new` 时，`--name` 的参数值必须位于 `--initial-cluster` 列表中；
+ 这是192.168.1.121节点的配置，其他两个etcd节点只要将上面的IP地址改成相应节点的IP地址，ETCD_NAME改成配置文件中定义的即可。

## 启动 etcd 服务

``` bash
# mv etcd.service /etc/systemd/system/
# systemctl daemon-reload
# systemctl enable etcd
# systemctl start etcd
# systemctl status etcd
```

在所有的 kubernetes master 节点重复上面的步骤，直到所有机器上的 etcd 服务都已正常启动。

## 验证服务

在任一 kubernetes master 机器上执行如下命令：

``` bash
# etcdctl \
--ca-file=/etc/kubernetes/ssl/ca.pem \
--cert-file=/etc/kubernetes/ssl/kubernetes.pem \
--key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
cluster-health
2017-07-24 15:28:43.051637 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
2017-07-24 15:28:43.052674 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
member 669bc6472fb13679 is healthy: got healthy result from https://192.168.1.121:2379
member aba9edaf5d433902 is healthy: got healthy result from https://192.168.1.122:2379
member d250ef9d0d70c7c9 is healthy: got healthy result from https://192.168.1.123:2379
cluster is healthy
```

结果最后一行为 `cluster is healthy` 时表示集群服务正常。