## 安装kube-master节点

部署master节点主要包含三个组件`apiserver` `scheduler` `controller-manager`，其中：

- apiserver提供集群管理的REST API接口，包括认证授权、数据校验以及集群状态变更等
  - 只有API Server才直接操作etcd
  - 其他模块通过API Server查询或修改数据
  - 提供其他模块之间的数据交互和通信的枢纽
- scheduler负责分配调度Pod到集群内的node节点
  - 监听kube-apiserver，查询还未分配Node的Pod
  - 根据调度策略为这些Pod分配节点
- controller-manager由一系列的控制器组成，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态

master节点的高可用主要就是实现apiserver组件的高可用，后面我们会看到每个 node 节点会启用 4 层负载均衡来请求 apiserver 服务。

```bash
# k8s-master节点操作
mkdir -p /etc/kubernetes/ssl /opt/kube/bin
# chrony-server节点操作
scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json,admin-key.pem,admin.pem} k8s-master-1:/etc/kubernetes/ssl
scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json,admin-key.pem,admin.pem} k8s-master-2:/etc/kubernetes/ssl
scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json,admin-key.pem,admin.pem} k8s-master-3:/etc/kubernetes/ssl
scp /root/ssl/{kube-controller-manager.kubeconfig,kube-proxy.kubeconfig,kube-scheduler.kubeconfig} k8s-master-1:/etc/kubernetes/
scp /root/ssl/{kube-controller-manager.kubeconfig,kube-proxy.kubeconfig,kube-scheduler.kubeconfig} k8s-master-2:/etc/kubernetes/
scp /root/ssl/{kube-controller-manager.kubeconfig,kube-proxy.kubeconfig,kube-scheduler.kubeconfig} k8s-master-3:/etc/kubernetes/
scp -r /root/.kube/ k8s-master-1:/root/
scp -r /root/.kube/ k8s-master-2:/root/
scp -r /root/.kube/ k8s-master-3:/root/
```

```
修改kube-controller-manager.kubeconfig,kube-proxy.kubeconfig,kube-scheduler.kubeconfig，/root/.kube/config文件中的server
为    server: https://192.168.166.95:6443   本机地址
```



```
wget ftp://192.168.166.21/kubernetes-server-linux-amd64.tar.gz
tar -xvf kubernetes-server-linux-amd64.tar.gz 
cp kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /opt/kube/bin
cp kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /usr/local/bin
```



### 创建 kubernetes 证书签名请求



```bash
cd /etc/kubernetes/ssl/
vim kubernetes-csr.json
```

```
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.166.170",
    "10.68.0.1",
    "10.1.1.1",
    "k8s.test.io",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- kubernetes 证书既是服务器证书，同时apiserver又作为客户端证书去访问etcd 集群；作为服务器证书需要设置hosts 指定使用该证书的IP 或域名列表，需要注意的是：
  - 如果配置 ex-lb，需要把 EX_APISERVER_VIP 也配置进去
  - 如果需要外部访问 apiserver，需要在 defaults/main.yml 配置 MASTER_CERT_HOSTS
  - `kubectl get svc` 将看到集群中由api-server 创建的默认服务 `kubernetes`，因此也要把 `kubernetes` 服务名和各个服务域名也添加进去

```bash
curl -s -L -o /usr/local/bin/cfssl ftp://192.168.166.21/cfssl_linux-amd64
curl -s -L -o /usr/local/bin/cfssljson ftp://192.168.166.21/cfssljson_linux-amd64
chmod +x /usr/local/bin/{cfssl,cfssljson}
```



```bash
cfssl gencert         -ca=ca.pem         -ca-key=ca-key.pem         -config=ca-config.json         -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```



### 【可选】创建基础用户名/密码认证配置

若未创建任何基础认证配置，K8S集群部署完毕后访问dashboard将会提示`401`错误。

当前如需创建基础认证，需单独修改`roles/kube-master/defaults/main.yml`文件，将`BASIC_AUTH_ENABLE`改为`yes`，并相应配置用户名`BASIC_AUTH_USER`（默认用户名为`admin`）及密码`BASIC_AUTH_PASS`。

### 创建apiserver的服务配置文件

```
vim /usr/lib/systemd/system/kube-apiserver.service
```

```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook \
  --advertise-address=192.168.166.170 \
  --allow-privileged=true \
  --anonymous-auth=false \
  --authorization-mode=Node,RBAC \
  --bind-address=192.168.166.170 \
  --insecure-bind-address=127.0.0.1 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --endpoint-reconciler-type=lease \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://192.168.166.192:2379,https://192.168.166.15:2379,https://192.168.166.82:2379 \
  --kubelet-certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/admin.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/admin-key.pem \
  --kubelet-https=true \
  --service-account-key-file=/etc/kubernetes/ssl/ca.pem \
  --service-cluster-ip-range=10.68.0.0/16 \
  --service-node-port-range=20000-40000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --requestheader-client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --requestheader-allowed-names= \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  #--proxy-client-cert-file=/etc/kubernetes/ssl/aggregator-proxy.pem \
  #--proxy-client-key-file=/etc/kubernetes/ssl/aggregator-proxy-key.pem \
  --enable-aggregator-routing=true \
  --logtostderr=true \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=always
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

- Kubernetes 对 API 访问需要依次经过认证、授权和准入控制(admission controll)，认证解决用户是谁的问题，授权解决用户能做什么的问题，Admission Control则是资源管理方面的作用。
- 支持同时提供https（默认监听在6443端口）和http API（默认监听在127.0.0.1的8080端口），其中http API是非安全接口，不做任何认证授权机制，kube-scheduler、kube-controller-manager 一般和 kube-apiserver 部署在同一台机器上，它们使用非安全端口和 kube-apiserver通信; 其他集群外部就使用HTTPS访问 apiserver
- 关于authorization-mode=Node,RBAC v1.7+支持Node授权，配合NodeRestriction准入控制来限制kubelet仅可访问node、endpoint、pod、service以及secret、configmap、PV和PVC等相关的资源；需要注意的是v1.7中Node 授权是默认开启的，v1.8中需要显式配置开启，否则 Node无法正常工作
- 缺省情况下 kubernetes 对象保存在 etcd /registry 路径下，可以通过 --etcd-prefix 参数进行调整
- 详细参数配置请参考`kube-apiserver --help`，关于认证、授权和准入控制请[阅读](https://github.com/feiskyer/kubernetes-handbook/blob/master/components/apiserver.md)
- 增加了访问kubelet使用的证书配置，防止匿名访问kubelet的安全漏洞，详见[漏洞说明](https://github.com/easzlab/kubeasz/blob/master/docs/mixes/01.fix_kubelet_annoymous_access.md)

### 创建controller-manager 的服务文件

```
vim /usr/lib/systemd/system/kube-controller-manager.service

```

```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kube/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --allocate-node-cidrs=true \
  --cluster-cidr=172.20.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
  --leader-elect=true \
  --node-cidr-mask-size=24 \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.68.0.0/16 \
  --use-service-account-credentials=true \
  --v=2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- --address 值必须为 127.0.0.1，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器
- --master=[http://127.0.0.1:8080](http://127.0.0.1:8080/) 使用非安全 8080 端口与 kube-apiserver 通信
- --cluster-cidr 指定 Cluster 中 Pod 的 CIDR 范围，该网段在各 Node 间必须路由可达(calico 实现)
- --service-cluster-ip-range 参数指定 Cluster 中 Service 的CIDR范围，必须和 kube-apiserver 中的参数一致
- --cluster-signing-* 指定的证书和私钥文件用来签名为 TLS BootStrap 创建的证书和私钥
- --root-ca-file 用来对 kube-apiserver 证书进行校验，指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件
- --leader-elect=true 使用多节点选主的方式选择主节点。只有主节点才会启动所有控制器，而其他从节点则仅执行选主算法

### 创建scheduler 的服务文件

```bash
vim /usr/lib/systemd/system/kube-scheduler.service
```



```
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kube/bin/kube-scheduler \
  --address=127.0.0.1 \
  --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
  --leader-elect=true \
  --v=2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- --address 同样值必须为 127.0.0.1
- --master=[http://127.0.0.1:8080](http://127.0.0.1:8080/) 使用非安全 8080 端口与 kube-apiserver 通信
- --leader-elect=true 部署多台机器组成的 master 集群时选举产生一个处于工作状态的 kube-controller-manager 进程

### 启动服务

```
systemctl daemon-reload 
systemctl restart kube-apiserver.service 
systemctl restart kube-controller-manager.service 
systemctl restart kube-scheduler
```

### master 集群的验证

安装补全命令

```
cat <<EOF >> /root/.bashrc
source <(kubectl completion bash)
EOF
```

```bash
source /root/.bashrc
```

```
# 查看进程状态
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
# 查看进程运行日志
journalctl -u kube-apiserver
journalctl -u kube-controller-manager
journalctl -u kube-scheduler
```

执行 `kubectl get componentstatus` 可以看到

```
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"} 
```

## 