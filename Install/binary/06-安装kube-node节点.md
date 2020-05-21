## 安装kube-node节点

`kube-node` 是集群中运行工作负载的节点，前置条件需要先部署好`kube-master`节点，它需要部署如下组件：

- docker：运行容器
- kubelet： kube-node上最主要的组件
- kube-proxy： 发布应用服务与负载均衡
- haproxy：用于请求转发到多个 apiserver，详见[HA-2x 架构](https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md#ha-architecture)
- calico： 配置容器网络 (或者其他网络组件)

### 变量配置文件

- 变量`KUBE_APISERVER`，根据不同的节点情况，它有三种取值方式
- 变量`MASTER_CHG`，变更 master 节点时会根据它来重新配置 haproxy

### 创建cni 基础网络插件配置文件

因为后续需要用 `DaemonSet Pod`方式运行k8s网络插件，所以kubelet.server服务必须开启cni相关参数，并且提供cni网络配置文件

### 创建 kubelet 的服务文件

- 根据官方建议独立使用 kubelet 配置文件，详见roles/kube-node/templates/kubelet-config.yaml.j2
- 必须先创建工作目录 `/var/lib/kubelet`

```bash
mkdir -p /var/lib/kubelet /var/lib/kube-proxy /etc/cni/net.d /opt/kube/bin /etc/kubernetes/ssl
wget ftp://192.168.166.21:/kubernetes-node-linux-amd64.tar.gz
tar -xvf kubernetes-node-linux-amd64.tar.gz
cp kubernetes/node/bin/* /opt/kube/bin/
cp kubernetes/node/bin/* /usr/local/bin/
```

```bash
yum -y install haproxy

```

```
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

```
vim /etc/haproxy/haproxy.cfg
```

```
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
        server 192.168.166.170 192.168.166.170:6443 check inter 10s fall 2 rise 2 weight 1
        server 192.168.166.156 192.168.166.156:6443 check inter 10s fall 2 rise 2 weight 1
        server 192.168.166.46 192.168.166.46:6443 check inter 10s fall 2 rise 2 weight 1
```

> server地址改为三个master地址

```
systemctl daemon-reload  && systemctl enable haproxy && systemctl start haproxy
```

```
# chrony-server操作
scp -r /root/.kube/ k8s-node-1:/root/
scp -r /root/.kube/ k8s-node-2:/root/
scp -r /root/.kube/ k8s-node-3:/root/
# node操作，修改config中的server地址，为127

```

```
cd /etc/kubernetes/ssl/

vim kubelet-csr.json

```

```
{
  "CN": "system:node:192.168.166.251",
  "hosts": [
    "127.0.0.1",
    "192.168.166.251"
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
```

```
curl -s -L -o /usr/local/bin/cfssl ftp://192.168.166.21/cfssl_linux-amd64
curl -s -L -o /usr/local/bin/cfssljson ftp://192.168.166.21/cfssljson_linux-amd64
chmod +x /usr/local/bin/{cfssl,cfssljson}
# chron-server 操作
 scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json} k8s-node-1:/etc/kubernetes/ssl
 scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json} k8s-node-2:/etc/kubernetes/ssl
 scp /root/ssl/{ca.pem,ca-key.pem,ca-config.json} k8s-node-3:/etc/kubernetes/ssl

```

```
cfssl gencert         -ca=ca.pem         -ca-key=ca-key.pem         -config=ca-config.json         -profile=kubernetes kubelet-csr.json | cfssljson -bare kubelet
kubectl config set-cluster kubernetes         --certificate-authority=ca.pem         --embed-certs=true         --server=https://127.0.0.1:6443    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config set-credentials system:node:192.168.166.251         --client-certificate=kubelet.pem         --embed-certs=true         --client-key=kubelet-key.pem    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config set-context default         --cluster=kubernetes    --user=system:node:192.168.166.251    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
kubectl config use-context default    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
```

```
vim /var/lib/kubelet/config.yaml
```

```
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.166.40
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
healthzBindAddress: 192.168.166.40
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

```
vim /usr/lib/systemd/system/kubelet.service
```



```
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
  --hostname-override=192.168.166.251 \
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

- --pod-infra-container-image 指定`基础容器`（负责创建Pod 内部共享的网络、文件系统等）镜像，**K8S每一个运行的 POD里面必然包含这个基础容器**，如果它没有运行起来那么你的POD 肯定创建不了，kubelet日志里面会看到类似 `FailedCreatePodSandBox` 错误，可用`docker images` 查看节点是否已经下载到该镜像
- --cluster-dns 指定 kubedns 的 Service IP(可以先分配，后续创建 kubedns 服务时指定该 IP)，--cluster-domain 指定域名后缀，这两个参数同时指定后才会生效；
- --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir={{ bin_dir }} 为使用cni 网络，并调用calico管理网络所需的配置
- --fail-swap-on=false K8S 1.8+需显示禁用这个，否则服务不能启动
- --client-ca-file={{ ca_dir }}/ca.pem 和 --anonymous-auth=false 关闭kubelet的匿名访问，详见[匿名访问漏洞说明](https://github.com/easzlab/kubeasz/blob/master/docs/setup/mixes/01.fix_kubelet_annoymous_access.md)
- --ExecStartPre=/bin/mkdir -p xxx 对于某些系统（centos7）cpuset和hugetlb 是默认没有初始化system.slice 的，需要手动创建，否则在启用--kube-reserved-cgroup 时会报错Failed to start ContainerManager Failed to enforce System Reserved Cgroup Limits
- 关于kubelet资源预留相关配置请参考 https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/

```
systemctl daemon-reload && systemctl enable kubelet && systemctl start kubelet.service 
systemctl status kubelet.service

```



### 创建 kube-proxy kubeconfig 文件

该步骤已经在 deploy节点完成，

- 生成的kube-proxy.kubeconfig 配置文件需要移动到/etc/kubernetes/目录，后续kube-proxy服务启动参数里面需要指定

```
# chrony-server操作
scp /root/ssl/kube-proxy.kubeconfig k8s-node-1:/etc/kubernetes
scp /root/ssl/kube-proxy.kubeconfig k8s-node-2:/etc/kubernetes
scp /root/ssl/kube-proxy.kubeconfig k8s-node-3:/etc/kubernetes
# node操作，修改kube-proxy.kubeconfig中的server
```



### 创建 kube-proxy服务文件

```
vim /usr/lib/systemd/system/kube-proxy.service

```



```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
# kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后，kube-proxy 会对访问 Service IP 的请求做 SNAT
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kube/bin/kube-proxy \
  --bind-address=192.168.166.251 \
  --cluster-cidr=172.20.0.0/16 \
  --hostname-override=192.168.166.251 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --proxy-mode=ipvs
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

- --hostname-override 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则
- 特别注意：kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；

```
systemctl daemon-reload 
systemctl enable kube-proxy.service 
systemctl start kube-proxy.service 
systemctl status kube-proxy.service

```



### 验证 node 状态

```
systemctl status kubelet	# 查看状态
systemctl status kube-proxy
journalctl -u kubelet		# 查看日志
journalctl -u kube-proxy 
```

```
cat <<EOF >> /root/.bashrc
source <(kubectl completion bash)
EOF
```

```
source /root/.bashrc
```



运行 `kubectl get node` 可以看到类似

```
NAME           STATUS    ROLES     AGE       VERSION
192.168.1.42   Ready     <none>    2d        v1.9.0
192.168.1.43   Ready     <none>    2d        v1.9.0
192.168.1.44   Ready     <none>    2d        v1.9.0
```

```
kubectl label node 192.168.166.251 kubernetes.io/role=node --overwrite
```

### 安装cni

> 建议略过此步骤，直接安装网络插件，flannel等

```
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
 ## 注意修改里面的cluster_ip的CIDR
 ## 在任意节点，apply一次即可
 kubectl apply -f kube-flannel.yml 
```

```
systemctl restart kubelet
kubectl get nodes
```

> 如果是master节点，要多两步操作
>
> ```
> kubectl cordon 192.168.166.95
> kubectl label node 192.168.166.95 kubernetes.io/role=master --overwrite
> 
> ```