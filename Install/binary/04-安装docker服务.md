## 安装docker服务

### 安装docker

```bash
curl -sSL get.docker.com | bash -s -- --mirror Aliyun
```

### 清理 iptables

因为后续`calico`网络、`kube-proxy`等将大量使用 iptables规则，安装前清空所有`iptables`策略规则；常见发行版`Ubuntu`的 `ufw` 和 `CentOS`的 `firewalld`等基于`iptables`的防火墙最好直接卸载，避免不必要的冲突。

```
iptables -F && iptables -X \
        && iptables -F -t nat && iptables -X -t nat \
        && iptables -F -t raw && iptables -X -t raw \
        && iptables -F -t mangle && iptables -X -t mangle
```

- calico 网络支持 `network-policy`，使用的`calico-kube-controllers` 会使用到`iptables` 所有的四个表 `filter` `nat` `raw` `mangle`，所以一并清理

### 启动 docker

```
systemctl daemon-reload && systemctl enable docker && systemctl start docker
```

### 配置加速器

```bash
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://pclhthp0.mirror.aliyuncs.com"]
}
EOF
```

```
systemctl restart docker
```



### 可选-安装docker查询镜像 tag的小工具

docker官方目前没有提供在命令行直接查询某个镜像的tag信息的方式，网上找来一个脚本工具，使用很方便。

```
$ docker-tag library/ubuntu
"14.04"
"16.04"
"17.04"
"latest"
"trusty"
"trusty-20171117"
"xenial"
"xenial-20171114"
"zesty"
"zesty-20171114"
$ docker-tag mirrorgooglecontainers/kubernetes-dashboard-amd64
"v0.1.0"
"v1.0.0"
"v1.0.0-beta1"
"v1.0.1"
"v1.1.0-beta1"
"v1.1.0-beta2"
"v1.1.0-beta3"
"v1.7.0"
"v1.7.1"
"v1.8.0"
```

- 需要先apt安装轻量JSON处理程序 `jq`
- 然后下载脚本即可使用
- 脚本很简单，就一行命令如下

```
#!/bin/bash
curl -s -S "https://registry.hub.docker.com/v2/repositories/$@/tags/" | jq '."results"[]["name"]' |sort
```

- 对于 CentOS7 安装 `jq` 稍微费力一点，需要启用 `EPEL` 源

```
wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -ivh epel-release-latest-7.noarch.rpm
yum install jq
```

### 验证

运行`ansible-playbook 03.docker.yml` 成功后可以验证

```
systemctl status docker 	# 服务状态
journalctl -u docker 		# 运行日志
docker version
docker info
```

`iptables-save|grep FORWARD` 查看 iptables filter表 FORWARD链，最后要有一个 `-A FORWARD -j ACCEPT` 保底允许规则

```
iptables-save|grep FORWARD
:FORWARD ACCEPT [0:0]
:FORWARD DROP [0:0]
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A FORWARD -j ACCEPT
```

## 