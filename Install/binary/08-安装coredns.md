从kubernetes  github上下载coredns的yaml文件

修改以下字段

我根据yaml_base文件来改

```
__PILLAR__DNS__DOMAIN__ 变更为cluster.local.
__PILLAR__DNS__MEMORY__LIMIT__ 变更为170Mi
__PILLAR__DNS__SERVER__ 变更为10.68.0.2
```

```
wget ftp://192.168.166.21/coredns.yaml


## chrony server 操作
scp coredns-1.6.6.tar k8s-node-1:/root/
scp coredns-1.6.6.tar k8s-node-2:/root/
scp coredns-1.6.6.tar k8s-node-3:/root/

## node操作
docker load -i coredns-1.6.6.tar 
```

### 检验测试，网络的连通性

```
wget ftp://192.168.166.21/nginx-deployment.yaml
kubectl apply -f nginx-deployment.yaml 
```

```bash
kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE              NOMINATED NODE   READINESS GATES
nginx-deployment-6dd8bc586b-k9lz5   1/1     Running   0          4m35s   172.20.0.2   192.168.166.251   <none>           <none>
nginx-deployment-6dd8bc586b-xcsx9   1/1     Running   0          4m35s   172.20.1.2   192.168.166.53    <none>           <none>
nginx-deployment-6dd8bc586b-zsr82   1/1     Running   0          4m35s   172.20.1.3   192.168.166.53    <none>           <none>

```

> 在任意主机平  pod内的ip，如果能ping通，代表网络连通性正常