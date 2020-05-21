## 架构图

![image-20200507161538839](images/image-20200507161538839.png)

## 节点规划

| 角色              | 数量 | 描述                                                     |
| ----------------- | ---- | -------------------------------------------------------- |
| chrony-server节点 | 1    | 其余主机同步时间源，证书存放地址                         |
| etcd节点          | 3    | 注意etcd集群需要1,3,5,7...奇数个节点，一般复用master节点 |
| master节点        | 3    | 高可用集群至少2个master节点                              |
| node节点          | 3    | 运行应用负载的节点，可根据需要提升机器配置/增加节点数    |

## 组件版本

| 组件       | 版本    |
| ---------- | ------- |
| kubernetes | v1.18.2 |
| etcd       | 3.4.3   |
| docker     | 19.03.9 |

## Service 和 Pods Ip 段划分

| 名称               | ip网段        | 备注                       |
| ------------------ | ------------- | -------------------------- |
| service-cluster-ip | 10.68.0.0/16  | 可用地址 65534             |
| pods-ip            | 172.20.0.0/16 | 可用地址 65534             |
| 集群dns            | 10.68.0.2     | 用于集群service域名解析    |
| k8s svc            | 10.68.0.1     | 集群 kubernetes svc 解析ip |



## ip对应主机名

| ip地址          | hostname      |
| --------------- | ------------- |
| 192.168.166.229 | chrony-server |
| 192.168.166.178 | k8s-master-1  |
| 192.168.166.217 | k8s-master-2  |
| 192.168.166.38  | k8s-master-3  |
| 192.168.166.99  | k8s-node-1    |
| 192.168.166.51  | k8s-node-2    |
| 192.168.166.205 | k8s-node-3    |
| 192.168.166.73  | etcd-1        |
| 192.168.166.111 | etcd-2        |
| 192.168.166.221 | etcd-3        |

## 软件对应地址

所有软件，我都已经提前下载好，并上传到了内网的ftp服务器上

* kubernetes-server-v1.18.2    下载链接

  https://storage.googleapis.com/kubernetes-release/release/v1.18.2/kubernetes-server-linux-amd64.tar.gz

* kubernetes-node-v1.18.2     下载链接

  https://storage.googleapis.com/kubernetes-release/release/v1.18.2/kubernetes-node-linux-amd64.tar.gz

* cfssl_linux-amd64        下载链接

  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64

* cfssljson_linux-amd64     下载链接

  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

* etcd-v3.4.3-linux-amd64.tar.gz

  https://github-production-release-asset-2e65be.s3.amazonaws.com/11225014/e3083e80-f583-11e9-91d0-084dffe50098?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20200506%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200506T082231Z&X-Amz-Expires=300&X-Amz-Signature=641deef2e7f732cc7bcea8ed85e169a8714dacc6b424e24cdfe85c10d31c0bf9&X-Amz-SignedHeaders=host&actor_id=34151437&repo_id=11225014&response-content-disposition=attachment%3B filename%3Detcd-v3.4.3-linux-amd64.tar.gz&response-content-type=application%2Foctet-stream

* cni-plugins-linux-amd64-v0.8.5.tgz 

  https://github-production-release-asset-2e65be.s3.amazonaws.com/84575398/0c3a9580-3d52-11ea-87ec-bcc2e12d89fc?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20200507%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200507T033451Z&X-Amz-Expires=300&X-Amz-Signature=07706adc5e447f4ac04053e63662669eb0d04d20ae5b3f280f3f5ebf035818cf&X-Amz-SignedHeaders=host&actor_id=34151437&repo_id=84575398&response-content-disposition=attachment%3B filename%3Dcni-plugins-linux-amd64-v0.8.5.tgz&response-content-type=application%2Foctet-stream

> 项目所需二进制软件包，一般在github官网或者工具的官网，都可以直接下载到，只不过需要翻墙。

| 软件包名称                           | 内网ftp下载地址                                           |
| ------------------------------------ | --------------------------------------------------------- |
| kubernetes-server-linux-amd64.tar.gz | ftp://192.168.166.21/kubernetes-server-linux-amd64.tar.gz |
| kubernetes-node-linux-amd64.tar.gz   | ftp://192.168.166.21:/kubernetes-node-linux-amd64.tar.gz  |
| cfssl_linux-amd64                    | ftp://192.168.166.21/cfssl_linux-amd64                    |
| cfssljson_linux-amd64                | ftp://192.168.166.21/cfssljson_linux-amd64                |
| etcd-v3.4.3-linux-amd64.tar.gz       | wget ftp://192.168.166.21/etcd-v3.4.3-linux-amd64.tar.gz  |
| cni-plugins-linux-amd64-v0.8.5.tgz   | ftp://192.168.166.21/cni-plugins-linux-amd64-v0.8.5.tgz   |

