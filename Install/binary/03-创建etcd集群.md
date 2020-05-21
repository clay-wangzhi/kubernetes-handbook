## 5. 安装etcd集群

kuberntes 集群使用 etcd 存储所有数据，是最重要的组件之一，注意 etcd集群需要奇数个节点(1,3,5...)，本文档使用3个节点做集群。

### 5.1 下载etcd/etcdctl 二进制文件

https://github.com/etcd-io/etcd/releases

```bash
wget ftp://192.168.166.21/etcd-v3.4.3-linux-amd64.tar.gz
tar -xvf etcd-v3.4.3-linux-amd64.tar.gz 
```



### 5.4 创建etcd 服务文件 etcd.service

```bash
vim /usr/lib/systemd/system/etcd.service
```



```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/opt/bin/etcd \
  --name=etcd-1 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls=https://192.168.166.192:2380 \
  --listen-peer-urls=https://192.168.166.192:2380 \
  --listen-client-urls=https://192.168.166.192:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://192.168.166.192:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=etcd-1=https://192.168.166.192:2380,etcd-2=https://192.168.166.15:2380,etcd-3=https://192.168.166.82:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```



> 根据启动文件，创建相应的目录和文件

```
mkdir -p /var/lib/etcd/ /opt/bin /etc/etcd/ssl
mv etcd-v3.4.3-linux-amd64/{etcd,etcdctl} /opt/bin
scp chrony-server:/root/ssl/{etcd.pem,etcd-key.pem,ca.pem} /etc/etcd/ssl/
```



- 完整参数列表请使用 `etcd --help` 查询
- 注意etcd 即需要服务器证书也需要客户端证书，这里为方便使用一个peer 证书代替两个证书；
- `--initial-cluster-state` 值为 `new` 时，`--name` 的参数值必须位于 `--initial-cluster` 列表中；

将使用到的相关文件，传到另两个节点上，并修改service文件中的`--name`,`  --initial-advertise-peer-urls`
  `--listen-peer-urls`、`  --listen-client-url `、` --advertise-client-urls`等相关参数

```
# etcd-2,etcd-3上执行
mkdir -p /var/lib/etcd/ /opt/bin /etc/etcd/ssl
# etcd-1上执行
scp /usr/lib/systemd/system/etcd.service etcd-2:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/etcd.service etcd-3:/usr/lib/systemd/system/
scp /etc/etcd/ssl/* etcd-2:/etc/etcd/ssl/
scp /etc/etcd/ssl/* etcd-3:/etc/etcd/ssl/
scp /opt/bin/* etcd-2:/opt/bin/
scp /opt/bin/* etcd-3:/opt/bin/
```



### 5.5 启动etcd服务

```
systemctl daemon-reload && systemctl enable etcd && systemctl start etcd
```

### 5.6 验证etcd集群状态

- systemctl status etcd 查看服务状态
- journalctl -u etcd 查看运行日志
- 在任一 etcd 集群节点上执行如下命令

```bash

```



```
# 根据hosts中配置设置shell变量 $NODE_IPS
export NODE_IPS="192.168.166.192 192.168.166.15 192.168.166.82"
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/etcd/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
```

预期结果：

```
https://192.168.166.192:2379 is healthy: successfully committed proposal: took = 23.261966ms
https://192.168.166.15:2379 is healthy: successfully committed proposal: took = 23.790746ms
https://192.168.166.82:2379 is healthy: successfully committed proposal: took = 26.764811ms
```

三台 etcd 的输出均为 healthy 时表示集群服务正常。