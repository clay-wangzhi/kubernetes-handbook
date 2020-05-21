# 准备工作

## 修改主机名

每台主机修改为环境说明中的hostname，并解析到每台的`/etc/hosts`文件中

* 设置主机名

```bash
hostnamectl set-hostname <hostname>
```

* 添加hosts

```bash
cat >> /etc/hosts << EOF
192.168.166.52 chrony-server
192.168.166.170 k8s-master-1
192.168.166.156 k8s-master-2
192.168.166.46 k8s-master-3
192.168.166.251 k8s-node-1
192.168.166.53 k8s-node-2
192.168.166.117 k8s-node-3
192.168.166.192 etcd-1
192.168.166.15 etcd-2
192.168.166.82 etcd-3
EOF
```

## 升级内核版本（所有主机）

k8s,docker,cilium等很多功能、特性需要较新的linux内核支持，所以有必要在集群部署前对内核进行升级。

* 载入公钥

```bash
rpm --import  https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

* 安装ELRepo

```bash
yum -y install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
```

* 安装新版本的kernel

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel -y install kernel-ml kernel-ml-devel
```

* 删除旧版本工具包

```bash
yum -y remove kernel-tools-libs.x86_64 kernel-tools.x86_64
```

* 安装新版本工具包

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel -y install kernel-ml-tools
```

* 选择默认启动内核

默认启动的顺序是从0开始，新内核是从头插入，所以需要选择0。

```bash
grub2-set-default 0
```

* 重启并检查

```bash
reboot
uname -r
```

## chrony 时间同步

在安装k8s集群前需确保各节点时间同步；`chrony` 是一个优秀的 `NTP` 实现，性能比ntp好，且配置管理方便；它既可作时间服务器服务端，也可作客户端。

### chrony-server安装配置

1）卸载原有的`ntp`服务

```bash
yum -y remove ntp ntpdate
```

2）安装`chrony`服务

```bash
yum -y install chrony
```

3）配置 `chrony server`

在`/etc/chrony.conf` 配置以下几项，其他项默认值即可

1. 配置时间源

   ```bash
   server ntp1.aliyun.com iburst
   server ntp2.aliyun.com iburst
   server time1.cloud.tencent.com iburst
   server time2.cloud.tencent.com iburst
   server 0.cn.pool.ntp.org iburst
   ```

2. 配置允许同步的客户端网段

   ```bash
   allow 0.0.0.0/0
   ```

   > 我这里允许所有，可以结合实际情况，具体配置

3. 配置离线也能作为源服务器

   ```bash
   local stratum 10
   ```

4）启动`chrony`服务

```bash
systemctl enable chronyd && systemctl start chronyd
```

5）验证配置

* 在 chrony server 检查时间源信息，默认配置为`ntp1.aliyun.com`的地址：

  ```bash
  chronyc sources -v
  ```

* 在 chrony server 检查时间源同步状态

  ```bash
  chronyc sourcestats -v
  ```

### chrony-client安装配置

1）卸载原有的`ntp`服务

```bash
yum -y remove ntp ntpdate
```

2）安装`chrony`服务

```bash
yum -y install chrony
```

3）配置`chrony client`

在`/etc/chrony.conf` 配置

清除所有其他时间源，只配置一个本地`chrony server`节点作为源

```text
server chrony-server iburst
```

4）启动`chrony`服务

```bash
systemctl enable chronyd && systemctl start chronyd
```

5）验证配置

* 在 chrony client 检查，可以看到时间源只有一个（groups.chrony\[0\] 节点地址）

  ```bash
  chronyc sources
  ```

### 验证时间同步状态完成

chrony 服务启动后，chrony server 会与配置的公网参考时间服务器进行同步；server 同步完成后，chrony client 会与 server 进行时间同步；一般来说整个集群达到时间同步需要几十分钟。可以用如下命令检查，初始时 **NTP synchronized: no**，同步完成后 **NTP synchronized: yes**

```bash
timedatectl
```

## 设置基础操作系统软件和系统参数

1）删除centos默认安装

```bash
yum -y remove firewalld python-firewall firewalld-filesystem
```

2）安装基础软件包

```bash
yum -y install bash-completion conntrack-tools ipset ipvsadm libseccomp nfs-utils psmisc rsync socat
```

3）关闭selinux：

```bash
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时
```

4）优化设置 journal 日志相关，避免日志重复搜集，浪费系统资源

编辑`/etc/rsyslog.conf`文件，注释掉下面内容

```text
ModLoad imjournal
IMJournalStateFile
```

重启rsyslog服务

```text
systemctl restart rsyslog
```

5）关闭swap：

```bash
swapoff -a  # 临时
vim /etc/fstab  # 永久
```

6）加载内核模块

```bash
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack
```

7）启用systemd自动加载模块服务

```bash
systemctl enabled systemd-modules-load
```

8）增加内核模块开机加载配置

```bash
# cat /etc/modules-load.d/10-k8s-modules.conf
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
```

9）设置系统参数

```bash
# cat /etc/sysctl.d/95-k8s-sysctl.conf 
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_tw_reuse = 0
net.netfilter.nf_conntrack_max=1000000
vm.swappiness = 0
vm.max_map_count=655360
fs.file-max=6553600
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
# sysctl -p /etc/sysctl.d/95-k8s-sysctl.conf
```

10 ）设置系统 ulimits

```bash
# cat /etc/systemd/system.conf.d/30-k8s-ulimits.conf 
[Manager]
DefaultLimitCORE=infinity
DefaultLimitNOFILE=100000
DefaultLimitNPROC=100000
```

11） 把SCTP列入内核模块黑名单

```text
# cat /etc/modprobe.d/sctp.conf 
# put sctp into blacklist
install sctp /bin/true
```

