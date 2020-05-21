# 快速安装

## 1 环境的快速准备

### 1.1 安装ansible并设置免秘钥登录

**chrony-server主机操作**

```bash
# 安装ansible
yum -y install ansible
# 配置文件优化
sed -i "s/#host_key_checking/host_key_checking/g" /etc/ansible/ansible.cfg
sed -i "s/#deprecation_warnings = True/deprecation_warnings = False/g" /etc/ansible/ansible.cfg 
# 生成密钥
ssh-keygen
# 将公钥传到原有的ansible主机上
scp /root/.ssh/id_rsa.pub 192.168.162.119:/tmp/a.pub
```

**原有ansible主机操作**

```bash
# 将新开的9(除去chrony-server主机)台主机加到ansible主机列表中
cat > hosts <<EOF
192.168.166.178
192.168.166.217
192.168.166.38
192.168.166.99
192.168.166.51
192.168.166.205
192.168.166.73
192.168.166.111
192.168.166.221
EOF
# 传输chrony-server的主机公钥到9台新主机上
ansible -i hosts all -m authorized_key -a "user=root key='{{ lookup('file', '/tmp/a.pub') }}'"
# 传出主机列表到chrony-server主机上
scp hosts 192.168.166.229:/etc/ansible/hosts
```

### 1.2 修改主机名

**chrony-server主机操作**

```bash
# 修改本机主机名
hostnamectl set-hostname chrony-server
# 本机添加hosts解析
cat >> /etc/hosts << EOF
192.168.166.229 chrony-server
192.168.166.178 k8s-master-1
192.168.166.217 k8s-master-2
192.168.166.38 k8s-master-3
192.168.166.99 k8s-node-1
192.168.166.51 k8s-node-2
192.168.166.205 k8s-node-3
192.168.166.73 etcd-1
192.168.166.111 etcd-2
192.168.166.221 etcd-3
EOF
# 修改ansible 主机列表
cat > /etc/ansible/hosts <<EOF
[k8s-master-1]
192.168.166.178
[k8s-master-2]
192.168.166.217
[k8s-master-3]
192.168.166.38
[k8s-node-1]
192.168.166.99
[k8s-node-2]
192.168.166.51
[k8s-node-3]
192.168.166.205
[etcd-1]
192.168.166.73
[etcd-2]
192.168.166.111
[etcd-3]
192.168.166.221
EOF
# 修改9台新主机主机名
for i in k8s-master-1 k8s-master-2 k8s-master-3 k8s-node-1 k8s-node-2 k8s-node-3 etcd-1 etcd-2 etcd-3; do ansible $i -m raw -a "hostnamectl set-hostname $i"; done
# 9台新主机更新hosts文件
ansible all -m copy -a "src=/etc/hosts dest=/etc/hosts"
```

### 1.3 升级内核 && 配置时间同步 && 系统参数调整

**chrony-server主机操作**

```bash
# 更新系统内核,执行过程中报错没关系，因为远程调用的shell脚本中有reboot命令
ansible all -m shell -a "curl -sSL ftp://192.168.166.21/kernel_upgrade.sh | bash"
# 检查系统内核是否更新
ansible all -m raw -a "uname -r"
# 安装chrony-server
curl -sSL ftp://192.168.166.21/chrony_server_install.sh | bash
# 安装chrony-client
ansible all -m shell -a "curl -sSL ftp://192.168.166.21/chrony_client_install.sh | bash"
# 系统优化
ansible all -m shell -a "curl -sSL ftp://192.168.166.21/system_optimize.sh | bash"
```

## 2 创建各种证书

### 2.1 创建CA证书

**chrony-server主机操作**

```bash
# 下载cfssl工具
curl -s -L -o /usr/local/bin/cfssl ftp://192.168.166.21/cfssl_linux-amd64
curl -s -L -o /usr/local/bin/cfssljson ftp://192.168.166.21/cfssljson_linux-amd64
chmod +x /usr/local/bin/{cfssl,cfssljson}
# 创建 CA 配置文件 ca-config.json
cd ~ && mkdir ssl
cd ssl
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
# 创建 CA 证书签名请求 ca-csr.json
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
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
  ],
  "ca": {
    "expiry": "876000h"
  }
}
EOF
# 生成CA 证书和私钥
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

### 2.2 生成 kubeconfig 配置文件

**chrony-server主机操作**

```bash
#  创建kubectl使用的admin 证书签名请求 admin-csr.json
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
# 生成 admin 用户证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
# 下载kubectl,生成 ~/.kube/config 配置文件
curl -s -L -o /usr/local/bin/kubectl ftp://192.168.166.21/kubectl
chmod +x /usr/local/bin/kubectl
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443
kubectl config set-credentials admin --client-certificate=admin.pem --embed-certs=true --client-key=admin-key.pem
kubectl config set-context kubernetes --cluster=kubernetes --user=admin
kubectl config use-context kubernetes
```

### 2.3 生成 kube-proxy.kubeconfig 配置文件

**chrony-server主机操作**

```bash
# 创建 kube-proxy 证书请求kube-proxy-csr.json
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
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
EOF
# 生成 system:kube-proxy 用户证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
# 生成 kube-proxy.kubeconfig
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443 --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --embed-certs=true --client-key=kube-proxy-key.pem --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

### 2.4 生成kube-controller-manager.kubeconfig 配置文件

**chrony-server主机操作**

```bash
# 创建 kube-controller-manager 证书请求kube-controller-manager-csr.json
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
EOF
# 生成 system:kube-controller-manager 用户证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
# 生成 kube-controller-manager.kubeconfig
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443 --kubeconfig=kube-controller-manager.kubeconfig
kubectl config set-credentials kube-controller-manager --client-certificate=kube-controller-manager.pem --embed-certs=true --client-key=kube-controller-manager-key.pem --kubeconfig=kube-controller-manager.kubeconfig
kubectl config set-context default --cluster=kubernetes --user=kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

### 2.5 生成kube-scheduler.kubeconfig 配置文件

**chrony-server主机操作**

```bash
# 创建 kube-scheduler 证书请求kube-scheduler-csr.json
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
EOF
# 生成 system:kube-scheduler 用户证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
# 生成 kube-scheduler.kubeconfig
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443 --kubeconfig=kube-scheduler.kubeconfig
kubectl config set-credentials kube-scheduler --client-certificate=kube-scheduler.pem --embed-certs=true --client-key=kube-scheduler-key.pem --kubeconfig=kube-scheduler.kubeconfig
kubectl config set-context default --cluster=kubernetes --user=kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

### 2.6 创建etcd证书

**chrony-server主机操作**

```bash
# 创建请求 etcd-csr.json
export etcd1="192.168.166.73" etcd2="192.168.166.111" etcd3="192.168.166.221"
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "$etcd1",
    "$etcd2",
    "$etcd3"
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
EOF
# 创建证书和私钥
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

## 3 安装etcd集群

### 3.1 下载etcd/etcdctl 二进制文件

**etcd三台主机操作**

```bash
# 下载压缩包
wget ftp://192.168.166.21/etcd-v3.4.3-linux-amd64.tar.gz
tar -xvf etcd-v3.4.3-linux-amd64.tar.gz
# 创建相关文件夹
mkdir -p /var/lib/etcd/ /opt/bin /etc/etcd/ssl
# 移动二进制文件到相应目录
mv etcd-v3.4.3-linux-amd64/{etcd,etcdctl} /opt/bin
cp /opt/bin/etcdctl /usr/local/bin
```

### 3.2 生成etcd证书

**chrony-server主机操作**

```bash
# 传证书到etcd节点
scp /root/ssl/{etcd.pem,etcd-key.pem,ca.pem} etcd-1:/etc/etcd/ssl/
scp /root/ssl/{etcd.pem,etcd-key.pem,ca.pem} etcd-2:/etc/etcd/ssl/
scp /root/ssl/{etcd.pem,etcd-key.pem,ca.pem} etcd-3:/etc/etcd/ssl/
```

### 3.3 创建etcd 服务文件 etcd.service

**etcd三台主机操作**

```bash
vim /usr/lib/systemd/system/etcd.service
```

```bash
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
  --initial-advertise-peer-urls=https://192.168.166.73:2380 \
  --listen-peer-urls=https://192.168.166.73:2380 \
  --listen-client-urls=https://192.168.166.73:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://192.168.166.73:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=etcd-1=https://192.168.166.73:2380,etcd-2=https://192.168.166.111:2380,etcd-3=https://192.168.166.221:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

> 根据实际情况，修改相关配置，需要修改的选项有：
>
> --name=
>
> --initial-advertise-peer-urls=
>
> --listen-peer-urls
>
> --listen-client-urls=
>
> --advertise-client-urls=
>
> --initial-cluster-token=
>
> --initial-cluster=

### 3.4 启动并验证etcd服务

**etcd三台主机操作**

```bash
# 启动
systemctl daemon-reload && systemctl enable etcd && systemctl start etcd
# 查看服务状态
systemctl status etcd
# 查看运行日志
journalctl -u etcd
```

```bash
# 在任一 etcd 集群节点上执行即可
export NODE_IPS="192.168.166.73 192.168.166.111 192.168.166.221"
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/etcd/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
```

预期结果：

```text
https://192.168.166.73:2379 is healthy: successfully committed proposal: took = 25.92351ms
https://192.168.166.111:2379 is healthy: successfully committed proposal: took = 25.952056ms
https://192.168.166.221:2379 is healthy: successfully committed proposal: took = 14.149766ms
```

## 4 安装docker服务

**chrony-server主机操作**

```bash
# 安装docker
ansible k8s-master-1,k8s-master-2,k8s-master-3,k8s-node-1,k8s-node-2,k8s-node-3 -m shell -a "curl -sSL get.docker.com | bash -s -- --mirror Aliyun" 
# 配置加速器
ansible k8s-master-1,k8s-master-2,k8s-master-3,k8s-node-1,k8s-node-2,k8s-node-3 -m shell -a "curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s https://pclhthp0.mirror.aliyuncs.com"
# 清空防火墙策略
ansible k8s-master-1,k8s-master-2,k8s-master-3,k8s-node-1,k8s-node-2,k8s-node-3 -m shell -a "iptables -F && iptables -X \
        && iptables -F -t nat && iptables -X -t nat \
        && iptables -F -t raw && iptables -X -t raw \
        && iptables -F -t mangle && iptables -X -t mangle"
# 启动docker
ansible k8s-master-1,k8s-master-2,k8s-master-3,k8s-node-1,k8s-node-2,k8s-node-3 -m shell -a "systemctl daemon-reload && systemctl enable docker && systemctl start docker"
```

## 5 安装kube-master节点

### 5.1 下载k8s-server二进制文件

**k8s-master节点操作**

```bash
# 创建相关目录
mkdir -p /etc/kubernetes/ssl /opt/kube/bin
# 下载并解压压缩包
wget ftp://192.168.166.21/kubernetes-server-linux-amd64.tar.gz
tar -xvf kubernetes-server-linux-amd64.tar.gz 
cp kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /opt/kube/bin
cp kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /usr/local/bin
```

### 5.2 生成相关证书文件

**chrony-server主机操作**

```bash
# 把相关证书传到k8s-master节点
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

**k8s-master节点操作**

```bash
# 下载cfssl相关工具
curl -s -L -o /usr/local/bin/cfssl ftp://192.168.166.21/cfssl_linux-amd64
curl -s -L -o /usr/local/bin/cfssljson ftp://192.168.166.21/cfssljson_linux-amd64
chmod +x /usr/local/bin/{cfssl,cfssljson}
# 创建 kubernetes 证书签名请求,k8smaster变量写自己主机的ip地址，三个主机不同
export k8smaster="192.168.166.178"
cat > /etc/kubernetes/ssl/kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "$k8smaster",
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
EOF
# 创建证书和私钥
cd /etc/kubernetes/ssl/
cfssl gencert         -ca=ca.pem         -ca-key=ca-key.pem         -config=ca-config.json         -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
# 创建aggregator proxy相关证书
cat > /etc/kubernetes/ssl/aggregator-proxy-csr.json <<EOF
{
  "CN": "aggregator",
  "hosts": [],
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
EOF
# 创建证书和私钥
cfssl gencert         -ca=ca.pem         -ca-key=ca-key.pem         -config=ca-config.json         -profile=kubernetes aggregator-proxy-csr.json | cfssljson -bare aggregator-proxy
```

### 5.3 创建apiserver的服务配置文件

**k8s-master节点操作**

```bash
vim /usr/lib/systemd/system/kube-apiserver.service
```

```bash
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook \
  --advertise-address=192.168.166.178 \
  --allow-privileged=true \
  --anonymous-auth=false \
  --authorization-mode=Node,RBAC \
  --bind-address=192.168.166.178 \
  --insecure-bind-address=127.0.0.1 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --endpoint-reconciler-type=lease \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://192.168.166.73:2379,https://192.168.166.111:2379,https://192.168.166.221:2379 \
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
  --proxy-client-cert-file=/etc/kubernetes/ssl/aggregator-proxy.pem \
  --proxy-client-key-file=/etc/kubernetes/ssl/aggregator-proxy-key.pem \
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

> 根据实际情况，修改相关配置，需要修改的选项有：
>
> --advertise-address=
>
> --bind-address=
>
> --etcd-servers=

```bash
# 启动服务
systemctl daemon-reload && systemctl enable kube-apiserver.service && systemctl start kube-apiserver.service 
# 查看服务状态
systemctl status kube-apiserver
# 查看运行日志
journalctl -u kube-apiserver
```

### 5.4 创建controller-manager 的服务文件

**k8s-master节点操作**

```bash
vim /usr/lib/systemd/system/kube-controller-manager.service
```

```bash
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

```bash
# 修改kube-controller-manager.kubeconfig文件,每个节点的ip都不通，根据实际情况修改
sed -i "s#127.0.0.1:8443#https://${k8smaster}:6443#g" /etc/kubernetes/kube-controller-manager.kubeconfig
# 启动服务
systemctl daemon-reload && systemctl enable kube-controller-manager.service && systemctl start kube-controller-manager.service 
# 查看服务状态
systemctl status kube-controller-manager
# 查看运行日志
journalctl -u kube-controller-manager
```

### 5.5 创建scheduler 的服务文件

**k8s-master节点操作**

```bash
vim /usr/lib/systemd/system/kube-scheduler.service
```

```bash
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

```bash
# 修改kube-scheduler.kubeconfig文件,每个节点的ip都不通，根据实际情况修改
sed -i "s#127.0.0.1:8443#https://${k8smaster}:6443#g" /etc/kubernetes/kube-scheduler.kubeconfig
# 启动服务
systemctl daemon-reload && systemctl enable kube-scheduler.service && systemctl start kube-scheduler.service 
# 查看服务状态
systemctl status kube-scheduler
# 查看运行日志
journalctl -u kube-scheduler
```

### 5.6 master 集群的验证

**k8s-master节点操作**

```bash
# 修改/root/.kube/config文件,每个节点的ip都不通，根据实际情况修改
sed -i "s#127.0.0.1:8443#https://${k8smaster}:6443#g" /root/.kube/config 
# 安装kubectl补全命令
cat <<EOF >> /root/.bashrc
source <(kubectl completion bash)
EOF
source /root/.bashrc
# 查看组件状态
kubectl get componentstatus
```

预期结果：

```bash
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}
```

## 6 安装kube-node节点

### 6.1 下载k8s-node二进制文件

**k8s-node节点操作**

```bash
mkdir -p /var/lib/kubelet /var/lib/kube-proxy /etc/cni/net.d /opt/kube/bin /etc/kubernetes/ssl
wget ftp://192.168.166.21:/kubernetes-node-linux-amd64.tar.gz
tar -xvf kubernetes-node-linux-amd64.tar.gz
cp kubernetes/node/bin/* /opt/kube/bin/
cp kubernetes/node/bin/* /usr/local/bin/
```

### 6.2 安装配置haproxy

**k8s-node节点操作**

```bash
# 安装haproxy
yum -y install haproxy
```

```bash
# 修改 haproxy.service文件
vim /usr/lib/systemd/system/haproxy.service
```

```bash
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target

[Service]
EnvironmentFile=/etc/sysconfig/haproxy
ExecStartPre=/usr/bin/mkdir -p /run/haproxy
ExecStart=/usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid $OPTIONS
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# 修改haproxy.cfg文件,根据实际情况修改server地址
vim /etc/haproxy/haproxy.cfg
```

```bash
global
        log /dev/log    local1 warning
        chroot /var/lib/haproxy
        user haproxy
        group haproxy
        daemon
        nbproc 1

defaults
        log     global
        timeout connect 5s
        timeout client  10m
        timeout server  10m

listen kube-master
        bind 127.0.0.1:6443
        mode tcp
        option tcplog
        option dontlognull
        option dontlog-normal
        balance roundrobin
        server 192.168.166.178 192.168.166.178:6443 check inter 10s fall 2 rise 2 weight 1
        server 192.168.166.217 192.168.166.217:6443 check inter 10s fall 2 rise 2 weight 1
        server 192.168.166.38 192.168.166.38:6443 check inter 10s fall 2 rise 2 weight 1
```

```bash
# 启动haproxy
systemctl daemon-reload  && systemctl enable haproxy && systemctl start haproxy
```

### 6.3 生产相关证书文件

**chrony-server主机操作**

```bash
# 把相关证书传到k8s-node节点
scp -r /root/.kube/ k8s-node-1:/root/
scp -r /root/.kube/ k8s-node-2:/root/
scp -r /root/.kube/ k8s-node-3:/root/
scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json} k8s-node-1:/etc/kubernetes/ssl
scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json} k8s-node-2:/etc/kubernetes/ssl
scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json} k8s-node-3:/etc/kubernetes/ssl
```

**k8s-node节点操作**

```bash
# 下载cfssl相关工具
curl -s -L -o /usr/local/bin/cfssl ftp://192.168.166.21/cfssl_linux-amd64
curl -s -L -o /usr/local/bin/cfssljson ftp://192.168.166.21/cfssljson_linux-amd64
chmod +x /usr/local/bin/{cfssl,cfssljson}
```

### 6.4 创建kubelet的kubeconfig文件

**k8s-node节点操作**

```bash
# 创建 kubelet 证书签名请求,k8smaster变量写自己主机的ip地址，三个主机不同
export k8snode="192.168.166.99"
cat > /etc/kubernetes/ssl/kubelet-csr.json <<EOF
{
  "CN": "system:node:$k8snode",
  "hosts": [
    "127.0.0.1",
    "$k8snode"
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
      "O": "system:nodes",
      "OU": "System"
    }
  ]
}
EOF
# 创建证书和私钥
cd /etc/kubernetes/ssl/
cfssl gencert         -ca=ca.pem         -ca-key=ca-key.pem         -config=ca-config.json         -profile=kubernetes kubelet-csr.json | cfssljson -bare kubelet
# 创建kubelet 的kubeconfig文件,注意每个节点的node ip各不相同
kubectl config set-cluster kubernetes         --certificate-authority=ca.pem         --embed-certs=true         --server=https://127.0.0.1:6443    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config set-credentials system:node:${k8snode}         --client-certificate=kubelet.pem         --embed-certs=true         --client-key=kubelet-key.pem    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config set-context default         --cluster=kubernetes    --user=system:node:${k8snode}    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config use-context default    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
```

### 6.5 创建kubelet服务文件

**k8s-node节点操作**

```bash
# 修改kubelet的config.yaml文件,注意每个节点的node ip各不相同,需要修改相关ip
vim /var/lib/kubelet/config.yaml
```

```bash
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.166.99
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: cgroupfs
cgroupsPerQOS: true
clusterDNS:
- 10.68.0.2
clusterDomain: cluster.local.
configMapAndSecretChangeDetectionStrategy: Watch
containerLogMaxFiles: 3 
containerLogMaxSize: 10Mi
enforceNodeAllocatable:
- pods
- kube-reserved
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 200Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 40s
hairpinMode: hairpin-veth 
healthzBindAddress: 192.168.166.99
healthzPort: 10248
httpCheckFrequency: 40s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
kubeReservedCgroup: /system.slice/kubelet.service
kubeReserved: {'cpu':'200m','memory':'500Mi','ephemeral-storage':'1Gi'}
kubeAPIBurst: 100
kubeAPIQPS: 50
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeLeaseDurationSeconds: 40
nodeStatusReportFrequency: 1m0s
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
port: 10250
# disable readOnlyPort 
readOnlyPort: 0
resolvConf: /etc/resolv.conf
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
tlsCertFile: /etc/kubernetes/ssl/kubelet.pem
tlsPrivateKeyFile: /etc/kubernetes/ssl/kubelet-key.pem
```

```bash
# 修改service文件,注意每个节点的node ip各不相同,需要修改相关ip
vim /usr/lib/systemd/system/kubelet.service
```

```bash
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStartPre=/bin/mount -o remount,rw '/sys/fs/cgroup'
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/memory/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/pids/system.slice/kubelet.service
ExecStart=/opt/kube/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  --cni-bin-dir=/opt/kube/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --hostname-override=192.168.166.99 \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --network-plugin=cni \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.1 \
  --root-dir=/var/lib/kubelet \
  --v=2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# 启动服务
systemctl daemon-reload && systemctl enable kubelet && systemctl start kubelet.service 
# 查看服务状态
systemctl status kubelet
# 查看运行日志
journalctl -u kubelet
```

### 6.6 创建 kube-proxy的kubeconfig 文件

**chrony-server主机操作**

```bash
# 将kube-proxy.kubeconfig文件传到node节点上
scp /root/ssl/kube-proxy.kubeconfig k8s-node-1:/etc/kubernetes
scp /root/ssl/kube-proxy.kubeconfig k8s-node-2:/etc/kubernetes
scp /root/ssl/kube-proxy.kubeconfig k8s-node-3:/etc/kubernetes
```

**k8s-node节点操作**

```bash
# 修改kube-proxy.kubeconfig中的server
sed -i "s#127.0.0.1:8443#https://127.0.0.1:6443#g" /etc/kubernetes/kube-proxy.kubeconfig
```

### 6.7 创建 kube-proxy服务文件

**k8s-node节点操作**

```bash
# 修改service文件,注意每个节点的node ip各不相同,需要修改相关ip
vim /usr/lib/systemd/system/kube-proxy.service
```

```bash
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
# kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后，kube-proxy 会对访问 Service IP 的请求做 SNAT
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kube/bin/kube-proxy \
  --bind-address=192.168.166.99 \
  --cluster-cidr=172.20.0.0/16 \
  --hostname-override=192.168.166.99 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --proxy-mode=ipvs
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# 启动服务
systemctl daemon-reload && systemctl enable kube-proxy && systemctl start kube-proxy.service 
# 查看服务状态
systemctl status kube-proxy
# 查看运行日志
journalctl -u kube-proxy
```

### 6.8 验证node状态

**k8s-node节点操作**

```bash
# 修改/root/.kube/config文件
sed -i "s#127.0.0.1:8443#https://127.0.0.1:6443#g" /root/.kube/config 
# 设置kubectl补全命令
cat <<EOF >> /root/.bashrc
source <(kubectl completion bash)
EOF
source /root/.bashrc
# 设置node角色
kubectl label node 192.168.166.99 kubernetes.io/role=node --overwrite
kubectl label node 192.168.166.51 kubernetes.io/role=node --overwrite
kubectl label node 192.168.166.205 kubernetes.io/role=node --overwrite
# 查看node状态
kubectl get nodes -o wide
```

预期结果为：

```bash
NAME              STATUS     ROLES   AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
192.168.166.128   NotReady   node    16h   v1.18.2   192.168.166.128   <none>        CentOS Linux 7 (Core)   5.6.13-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.131   NotReady   node    16h   v1.18.2   192.168.166.131   <none>        CentOS Linux 7 (Core)   5.6.13-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.182   NotReady   node    16h   v1.18.2   192.168.166.182   <none>        CentOS Linux 7 (Core)   5.6.13-1.el7.elrepo.x86_64   docker://19.3.9
```

> 状态为NotReady，因为我们还没有安装cni网络插件

## 7 kube-master节点安装kubelet和kube-proxy

### 7.1 下载k8s-node二进制文件

**k8s-node节点操作**

```bash
mkdir -p /var/lib/kubelet /var/lib/kube-proxy /etc/cni/net.d /opt/kube/bin /etc/kubernetes/ssl
wget ftp://192.168.166.21:/kubernetes-node-linux-amd64.tar.gz
tar -xvf kubernetes-node-linux-amd64.tar.gz
cp kubernetes/node/bin/{kubelet,kube-proxy} /opt/kube/bin/
cp kubernetes/node/bin/{kubelet,kube-proxy} /usr/local/bin/
```

### 7.2 安装配置kubelet

**k8s-master节点操作**

```bash
# 创建 kubelet 证书签名请求,k8smaster变量写自己主机的ip地址，三个主机不同
export k8snode="192.168.166.178"
cat > /etc/kubernetes/ssl/kubelet-csr.json <<EOF
{
  "CN": "system:node:$k8snode",
  "hosts": [
    "127.0.0.1",
    "$k8snode"
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
      "O": "system:nodes",
      "OU": "System"
    }
  ]
}
EOF
# 创建证书和私钥
cd /etc/kubernetes/ssl/
cfssl gencert         -ca=ca.pem         -ca-key=ca-key.pem         -config=ca-config.json         -profile=kubernetes kubelet-csr.json | cfssljson -bare kubelet
# 创建kubelet 的kubeconfig文件,注意每个节点的node ip各不相同
kubectl config set-cluster kubernetes         --certificate-authority=ca.pem         --embed-certs=true         --server=https://${k8snode}:6443    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config set-credentials system:node:${k8snode}         --client-certificate=kubelet.pem         --embed-certs=true         --client-key=kubelet-key.pem    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config set-context default         --cluster=kubernetes    --user=system:node:${k8snode}    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config use-context default    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
```

```bash
# 修改kubelet的config.yaml文件,注意每个节点的node ip各不相同,需要修改相关ip
vim /var/lib/kubelet/config.yaml
```

```bash
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.166.178
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: cgroupfs
cgroupsPerQOS: true
clusterDNS:
- 10.68.0.2
clusterDomain: cluster.local.
configMapAndSecretChangeDetectionStrategy: Watch
containerLogMaxFiles: 3 
containerLogMaxSize: 10Mi
enforceNodeAllocatable:
- pods
- kube-reserved
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 200Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 40s
hairpinMode: hairpin-veth 
healthzBindAddress: 192.168.166.178
healthzPort: 10248
httpCheckFrequency: 40s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
kubeReservedCgroup: /system.slice/kubelet.service
kubeReserved: {'cpu':'200m','memory':'500Mi','ephemeral-storage':'1Gi'}
kubeAPIBurst: 100
kubeAPIQPS: 50
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeLeaseDurationSeconds: 40
nodeStatusReportFrequency: 1m0s
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
port: 10250
# disable readOnlyPort 
readOnlyPort: 0
resolvConf: /etc/resolv.conf
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
tlsCertFile: /etc/kubernetes/ssl/kubelet.pem
tlsPrivateKeyFile: /etc/kubernetes/ssl/kubelet-key.pem
```

```bash
# 修改service文件,注意每个节点的node ip各不相同,需要修改相关ip
vim /usr/lib/systemd/system/kubelet.service
```

```bash
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStartPre=/bin/mount -o remount,rw '/sys/fs/cgroup'
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/memory/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/pids/system.slice/kubelet.service
ExecStart=/opt/kube/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  --cni-bin-dir=/opt/kube/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --hostname-override=192.168.166.178 \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --network-plugin=cni \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.1 \
  --root-dir=/var/lib/kubelet \
  --v=2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# 启动服务
systemctl daemon-reload && systemctl enable kubelet && systemctl start kubelet.service 
# 查看服务状态
systemctl status kubelet
# 查看运行日志
journalctl -u kubelet
```

### 7.3 安装配置kube-proxy

**k8s-master节点操作**

```bash
# 修改kube-proxy.kubeconfig中的server,每个节点的ip不通
sed -i "s#127.0.0.1:8443#https://${k8snode}:6443#g" /etc/kubernetes/kube-proxy.kubeconfig
```

```bash
# 修改service文件,注意每个节点的node ip各不相同,需要修改相关ip
vim /usr/lib/systemd/system/kube-proxy.service
```

```bash
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
# kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后，kube-proxy 会对访问 Service IP 的请求做 SNAT
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kube/bin/kube-proxy \
  --bind-address=192.168.166.178 \
  --cluster-cidr=172.20.0.0/16 \
  --hostname-override=192.168.166.178 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --proxy-mode=ipvs
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# 启动服务
systemctl daemon-reload && systemctl enable kube-proxy && systemctl start kube-proxy.service 
# 查看服务状态
systemctl status kube-proxy
# 查看运行日志
journalctl -u kube-proxy
```

### 7.4 验证node状态

**k8s-master节点操作**

```bash
# 禁用主节点调度
kubectl cordon 192.168.166.178
kubectl cordon 192.168.166.217
kubectl cordon 192.168.166.38
# 设置node角色
kubectl label node 192.168.166.178 kubernetes.io/role=master --overwrite
kubectl label node 192.168.166.217 kubernetes.io/role=master --overwrite
kubectl label node 192.168.166.38 kubernetes.io/role=master --overwrite
# 查看node状态
kubectl get nodes -o wide
```

预期结果：

```bash
NAME              STATUS                        ROLES    AGE    VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
192.168.166.178   NotReady,SchedulingDisabled   master   9m4s   v1.18.2   192.168.166.178   <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.205   NotReady                      node     19m    v1.18.2   192.168.166.205   <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.217   NotReady,SchedulingDisabled   master   9m4s   v1.18.2   192.168.166.217   <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.38    NotReady,SchedulingDisabled   master   9m4s   v1.18.2   192.168.166.38    <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.51    NotReady                      node     19m    v1.18.2   192.168.166.51    <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.99    NotReady                      node     19m    v1.18.2   192.168.166.99    <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
```

> 状态为NotReady，因为我们还没有安装cni网络插件

## 8 安装网络

### 8.1 安装flannel

**k8s-master，kube-node节点操作**

```bash
mkdir cni
cd cni
wget ftp://192.168.166.21/cni-plugins-linux-amd64-v0.8.5.tgz
tar -xvf cni-plugins-linux-amd64-v0.8.5.tgz 
cp bridge host-local loopback flannel portmap /opt/kube/bin/
systemctl restart kubelet
systemctl restart kube-proxy
```

```bash
wget ftp://192.168.166.21/kube-flannel.yml
# 注意修改里面的cluster_ip的CIDR
# 在任意节点，apply一次即可
kubectl apply -f kube-flannel.yml
kubectl get nodes -o wide
```

预期结果：

```bash
NAME              STATUS                     ROLES    AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
192.168.166.178   Ready,SchedulingDisabled   master   12m   v1.18.2   192.168.166.178   <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.205   Ready                      node     22m   v1.18.2   192.168.166.205   <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.217   Ready,SchedulingDisabled   master   12m   v1.18.2   192.168.166.217   <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.38    Ready,SchedulingDisabled   master   12m   v1.18.2   192.168.166.38    <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.51    Ready                      node     22m   v1.18.2   192.168.166.51    <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
192.168.166.99    Ready                      node     22m   v1.18.2   192.168.166.99    <none>        CentOS Linux 7 (Core)   5.6.14-1.el7.elrepo.x86_64   docker://19.3.9
```

### 8.2 检验测试，网络的连通性

```bash
wget ftp://192.168.166.21/nginx-deployment.yaml
kubectl apply -f nginx-deployment.yaml 
kubectl get pods -o wide
```

```bash
NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE              NOMINATED NODE   READINESS GATES
nginx-deployment-6dd8bc586b-4z47l   1/1     Running   0          2m12s   172.20.1.2   192.168.166.205   <none>           <none>
nginx-deployment-6dd8bc586b-55z98   1/1     Running   0          2m12s   172.20.1.3   192.168.166.205   <none>           <none>
nginx-deployment-6dd8bc586b-sgln5   1/1     Running   0          2m12s   172.20.1.4   192.168.166.205   <none>           <none>
```

```bash
ping 172.20.1.2
```

```bash
PING 172.20.1.2 (172.20.1.2) 56(84) bytes of data.
64 bytes from 172.20.1.2: icmp_seq=1 ttl=63 time=3.28 ms
64 bytes from 172.20.1.2: icmp_seq=2 ttl=63 time=0.509 ms
```

> 在任意主机ping pod内的ip，如果能ping通，代表网络连通性正常

## 9 安装coredns

### 9.1 安装coredns

从kubernetes github上下载coredns的yaml文件

修改以下字段

我根据yaml\_base文件来改

```bash
__PILLAR__DNS__DOMAIN__ 变更为cluster.local.
__PILLAR__DNS__MEMORY__LIMIT__ 变更为170Mi
__PILLAR__DNS__SERVER__ 变更为10.68.0.2
```

**chrony-server主机操作**

```bash
# 将提前下载好的coredns镜像传到node节点
scp coredns-1.6.6.tar k8s-node-1:/root/
scp coredns-1.6.6.tar k8s-node-2:/root/
scp coredns-1.6.6.tar k8s-node-3:/root/
```

**k8s-node节点操作**

```bash
# 加载镜像
docker load -i coredns-1.6.6.tar 
# 下载yaml文件
wget ftp://192.168.166.21/coredns.yaml
# 在任意节点，apply一次即可
kubectl apply -f  coredns.yaml
kubectl get pods -n kube-system
```

预期结果：

```bash
NAME                          READY   STATUS    RESTARTS   AGE
coredns-5bf47f866c-nl75b      1/1     Running   0          6m10s
```

### 9.2 验证coredns是否生效

**k8s-master任一节点操作**

```bash
# 下载service   yaml文件
wget ftp://192.168.166.21/nginx-service.yaml
# apply
kubectl apply -f nginx-service.yaml 
kubectl get service -o wide
```

```bash
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   SELECTOR
kubernetes      ClusterIP   10.68.0.1      <none>        443/TCP        64m   <none>
nginx-service   NodePort    10.68.27.177   <none>        80:32600/TCP   1s    app=nginx
```

```bash
# 测试是否返回nginx正常页面
curl 10.68.27.177
```

```bash
kubectl run test --rm -it --image=alpine /bin/sh
If you don't see a command prompt, try pressing enter.
/ # cat /etc/resolv.conf
nameserver 10.68.0.2
search default.svc.cluster.local. svc.cluster.local. cluster.local. openstacklocal
options ndots:5
# 测试集群内部服务解析
/ # nslookup nginx-service.default.svc.cluster.local
Server:        10.68.0.2
Address:    10.68.0.2:53


Name:    nginx-service.default.svc.cluster.local
Address: 10.68.27.177
/ # nslookup kubernetes.default.svc.cluster.local
Server:        10.68.0.2
Address:    10.68.0.2:53


Name:    kubernetes.default.svc.cluster.local
Address: 10.68.0.1
# 测试外部域名的解析，默认集成node的dns解析
/ # nslookup www.baidu.com
Server:        10.68.0.2
Address:    10.68.0.2:53

Non-authoritative answer:
www.baidu.com    canonical name = www.a.shifen.com
Name:    www.a.shifen.com
Address: 180.101.49.11
Name:    www.a.shifen.com
Address: 180.101.49.12

Non-authoritative answer:
www.baidu.com    canonical name = www.a.shifen.com
/ #
```

## 10 安装kuboard

### 10.1 安装

**k8s-master任一节点操作**

```bash
kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
kubectl apply -f https://addons.kuboard.cn/metrics-server/0.3.6/metrics-server.yaml
# 查看 Kuboard 运行状态：
kubectl get pods -l k8s.eip.work/name=kuboard -n kube-system
```

输出结果如下所示：

```bash
NAME                      READY   STATUS    RESTARTS   AGE
kuboard-8b8574658-6t8q6   1/1     Running   0          78s
```

### 10.2 获取Token

**k8s-master任一节点操作**

```bash
# 此Token拥有 ClusterAdmin 的权限，可以执行所有操作
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)
```

### 10.3 访问Kuboard

您可以通过NodePort、port-forward 两种方式当中的任意一种访问 Kuboard

* 通过NodePort访问
* 通过port-forward访问\]

Kuboard Service 使用了 NodePort 的方式暴露服务，NodePort 为 32567；您可以按如下方式访问 Kuboard。

```text
http://任意一个Worker节点的IP地址:32567/
```

输入前一步骤中获得的 token，可进入 **Kuboard 集群概览页**

