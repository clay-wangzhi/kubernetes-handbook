kubernetesV1.8版本后建议开启TLS双向认证及RBAC授权管理，以加强集群的安全管理。界内流行的开启TLS方法为基于一个“公钥基础设施（public key infrastructure，缩写为 PKI）”，使用了内部托管的认证中心（CA），常见PKI工具有CFSSL,OPENSSL等，下面详细介绍kubernetes使用CFSSL工具创建CA并开启TLS认证。

## **一、HTTP、HTTPS、SSL、TLS**

　　HTTP 是一个网络协议，是专门用来传输 Web 内容，大部分网站都是通过 HTTP 协议来传输 Web 页面、以及 Web 页面上包含的各种东东（图片、CSS 样式、JS 脚本）。　　

　　SSL 是“Secure Sockets Layer”的缩写，中文叫做“安全套接层”。它是在上世纪90年代中期，由网景公司设计的。（顺便插一句，网景公司不光发明了 SSL，还发明了很多 Web 的基础设施——比如“CSS 样式表”和“JS 脚本”），为啥要发明 SSL 这个协议捏？因为原先互联网上使用的 HTTP 协议是明文的，存在很多缺点——比如传输内容会被偷窥（嗅探）和篡改。发明 SSL 协议，就是为了解决这些问题。

　　TLS是 SSL的 标准化，SSL标准化之后的名称改为 TLS（是“Transport Layer Security”的缩写），中文叫做“传输层安全协议”。很多相关的文章都把这两者并列称呼（SSL/TLS），因为这两者可以视作同一个东西的不同阶段。

　　****HTTPS 协议，说白了就是“HTTP 协议”和“SSL/TLS 协议”的组合。你可以把 HTTPS 大致理解为——“HTTP over SSL”或“HTTP over TLS”。****

 

## **二、CA认证的原理**

　　通过下面介绍信的描述介绍CA的原理。

　　***\*◇\** 普通的介绍信**

　　想必大伙儿都听说过介绍信的例子吧？假设 A 公司的张三先生要到 B 公司去拜访，但是 B 公司的所有人都不认识他，他咋办捏？常用的办法是带公司开的一张介绍信，在信中说：兹有张三先生前往贵公司办理业务，请给予接洽......云云。然后在信上敲上A公司的公章。

　　张三先生到了 B 公司后，把介绍信递给 B 公司的前台李四小姐。李小姐一看介绍信上有 A 公司的公章，而且 A 公司是经常和 B 公司有业务往来的，这位李小姐就相信张先生不是歹人了。

这里，A公司就是CA证书

　　**◇ 引入中介机构的介绍信**

　　好，回到刚才的话题。如果和 B 公司有业务往来的公司很多，每个公司的公章都不同，那前台就要懂得分辨各种公章，非常滴麻烦。所以，有某个中介公司 C，发现了这个商机。C公司专门开设了一项“代理公章”的业务。

　　今后，A 公司的业务员去 B 公司，需要带2个介绍信：

　　介绍信1

　　含有 C 公司的公章及 A 公司的公章。并且特地注明：C 公司信任 A 公司。

　　介绍信2

　　仅含有 A 公司的公章，然后写上：兹有张三先生前往贵公司办理业务，请给予接洽......云云。

　　某些不开窍的同学会问了，这样不是增加麻烦了吗？有啥好处捏？

　　主要的好处在于，对于接待公司的前台，就不需要记住各个公司的公章分别是啥样子的；他/她只要记住中介公司 C 的公章即可。当他/她拿到两份介绍信之后，先对介绍信1的 C 公章，验明正身；确认无误之后，再比对介绍信1和介绍信2的两个 A 公章是否一致。如果是一样的，那就可以证明介绍信2是可以信任的了。

 

## **三、公钥基础设施(PKI)**

　　CA(Certification Authority)证书，指的是权威机构给我们颁发的证书。

　　密钥就是用来加解密用的文件或者字符串。密钥在非对称加密的领域里，指的是私钥和公钥，他们总是成对出现，其主要作用是加密和解密。常用的加密强度是2048bit。

　　RSA即非对称加密算法。非对称加密有两个不一样的密码，一个叫私钥，另一个叫公钥，用其中一个加密的数据只能用另一个密码解开，用自己的都解不了，也就是说用公钥加密的数据只能由私钥解开。

#### 证书的编码格式

　　PEM(Privacy Enhanced Mail)，通常用于数字证书认证机构（Certificate Authorities，CA），扩展名为`.pem`, `.crt`, `.cer`, 和 `.key`。内容为Base64编码的ASCII码文件，有类似`"-----BEGIN CERTIFICATE-----"` 和 `"-----END CERTIFICATE-----"`的头尾标记。服务器认证证书，中级认证证书和私钥都可以储存为PEM格式（认证证书其实就是公钥）。Apache和nginx等类似的服务器使用PEM格式证书。

　　DER(Distinguished Encoding Rules)，与PEM不同之处在于其使用二进制而不是Base64编码的ASCII。扩展名为`.der`，但也经常使用`.cer`用作扩展名，所有类型的认证证书和私钥都可以存储为DER格式。Java使其典型使用平台。

#### 证书签名请求CSR

　　CSR(Certificate Signing Request)，它是向CA机构申请数字×××书时使用的请求文件。在生成请求文件前，我们需要准备一对对称密钥。私钥信息自己保存，请求中会附上公钥信息以及国家，城市，域名，Email等信息，CSR中还会附上签名信息。当我们准备好CSR文件后就可以提交给CA机构，等待他们给我们签名，签好名后我们会收到crt文件，即证书。

　　注意：CSR并不是证书。而是向权威证书颁发机构获得签名证书的申请。

　　把CSR交给权威证书颁发机构,权威证书颁发机构对此进行签名,完成。保留好`CSR`,当权威证书颁发机构颁发的证书过期的时候,你还可以用同样的`CSR`来申请新的证书,key保持不变.

#### 数字签名

　　数字签名就是"非对称加密+摘要算法"，其目的不是为了加密，而是用来防止他人篡改数据。

　　其核心思想是：比如A要给B发送数据，A先用摘要算法得到数据的指纹，然后用A的私钥加密指纹，加密后的指纹就是A的签名，B收到数据和A的签名后，也用同样的摘要算法计算指纹，然后用A公开的公钥解密签名，比较两个指纹，如果相同，说明数据没有被篡改，确实是A发过来的数据。假设C想改A发给B的数据来欺骗B，因为篡改数据后指纹会变，要想跟A的签名里面的指纹一致，就得改签名，但由于没有A的私钥，所以改不了，如果C用自己的私钥生成一个新的签名，B收到数据后用A的公钥根本就解不开。

　　常用的摘要算法有MD5、SHA1、SHA256。

　　使用私钥对需要传输的文本的摘要进行加密，得到的密文即被称为该次传输过程的签名。

#### 数字证书和公钥

　　数字证书则是由证书认证机构（CA）对证书申请者真实身份验证之后，用CA的根证书对申请人的一些基本信息以及申请人的公钥进行签名（相当于加盖发证书机 构的公章）后形成的一个数字文件。实际上，数字证书就是经过CA认证过的公钥，除了公钥，还有其他的信息，比如Email，国家，城市，域名等。

## **四、CFSSL安装及基础知识**　

　　项目地址： https://github.com/cloudflare/cfssl

　　下载地址： https://pkg.cfssl.org/

　　CFSSL是CloudFlare开源的一款PKI/TLS工具。 CFSSL 包含一个命令行工具 和一个用于 签名，验证并且捆绑TLS证书的 HTTP API 服务。 使用Go语言编写。

CFSSL包括：

- 一组用于生成自定义 TLS PKI 的工具
- `cfssl`程序，是CFSSL的命令行工具
- `multirootca`程序是可以使用多个签名密钥的证书颁发机构服务器
- `mkbundle`程序用于构建证书池
- `cfssljson`程序，从`cfssl`和`multirootca`程序获取JSON输出，并将证书，密钥，CSR和bundle写入磁盘

PKI借助数字证书和公钥加密技术提供可信任的网络身份。通常，证书就是一个包含如下身份信息的文件：

- 证书所有组织的信息
- 公钥
- 证书颁发组织的信息
- 证书颁发组织授予的权限，如证书有效期、适用的主机名、用途等
- 使用证书颁发组织私钥创建的数字签名

**安装cfssl**

这里我们只用到`cfssl`工具和`cfssljson`工具：

```js
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

`cfssl`工具，子命令介绍：

- `bundle`: 创建包含客户端证书的证书包
- `genkey`: 生成一个key(私钥)和CSR(证书签名请求)
- `scan`: 扫描主机问题
- `revoke`: 吊销证书
- `certinfo`: 输出给定证书的证书信息， 跟cfssl-certinfo 工具作用一样
- `gencrl`: 生成新的证书吊销列表
- `selfsign`: 生成一个新的自签名密钥和 签名证书
- `print-defaults`: 打印默认配置，这个默认配置可以用作模板
- `serve`: 启动一个HTTP API服务
- `gencert`: 生成新的key(密钥)和签名证书
  - -ca：指明ca的证书
  - -ca-key：指明ca的私钥文件
  - -config：指明请求证书的json文件
  - -profile：与-config中的profile对应，是指根据config中的profile段来生成证书的相关信息
- `ocspdump`
- `ocspsign`
- `info`: 获取有关远程签名者的信息
- `sign`: 签名一个客户端证书，通过给定的CA和CA密钥，和主机名
- `ocsprefresh`
- `ocspserve`

 

**cfssl常用命令：**

- `cfssl gencert -initca ca-csr.json | cfssljson -bare ca ## 初始化ca`
- `cfssl gencert -initca -ca-key key.pem ca-csr.json | cfssljson -bare ca ## 使用现有私钥, 重新生成`
- `cfssl certinfo -cert ca.pem`
- `cfssl certinfo -csr ca.csr`

 

## **五、使用CFSSL创建CA认证步骤**

### 　　1、创建认证中心(CA)

　　CFSSL可以创建一个获取和操作证书的内部认证中心。运行认证中心需要一个CA证书和相应的CA私钥。任何知道私钥的人都可以充当CA颁发证书。因此，私钥的保护至关重要。

```java
cfssl print-defaults config > config.json # 默认证书生产策略配置模板
cfssl print-defaults csr > csr.json #默认csr请求模板
#按照需求修改上述生成的文件
cat ca-csr.json
{
　　"CN": "kubernetes", 
　　"key": {
　　　　"algo": "rsa",
　　　　"size": 2048
　　},
　　"names": [
　　　　{
　　　　　　"C": "CN",
　　　　　　"ST": "JiangSu",
　　　　　　"L": "NanJing",
　　　　　　"O": "k8s",
　　　　　　"OU": "System"
　　　　}
　　　],
　　　"ca": {
　　　　"expiry": "87600h"
　　}
} 
 
知识点：
"CN"：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)
"O"：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)
CN: Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。浏览器使用该字段验证网站是否合法
C: Country， 国家
L: Locality，地区，城市
O: Organization Name，组织名称，公司名称
OU: Organization Unit Name，组织单位名称，公司部门
ST: State，州，省
```

```
cat ca-config.json
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
 
知识点：
ca-config.json：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；此实例只有一个kubernetes模板。
signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；
server auth：表示client可以用该 CA 对server提供的证书进行验证；
client auth：表示server可以用该CA对client提供的证书进行验证；
注意标点符号，最后一个字段一般是没有都好的。
 
#初始化创建CA认证中心，将会生成 ca-key.pem（私钥）  ca.pem（公钥）
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```



### 　　2、*创建kubernetes证书*

```java
   cat kubernetes-csr.json
    {
        "CN": "kubernetes",
        "hosts": [
          "127.0.0.1",
          "10.192.44.129",
          "10.192.44.128",
          "10.192.44.126",
          "10.192.44.127",
          "10.254.0.1",
          "*.kubernetes.master",
          "localhost",
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
                "ST": "JiangSu",
                "L": "NanJing",
                "O": "k8s",
                "OU": "System"
            }
        ]
    }

 知识点：
    这个证书目前专属于 apiserver，加了一个 *.kubernetes.master 域名以便内部私有 DNS 解析使用(可删除)；至于很多人问过 kubernetes 这几个能不能删掉，答案是不可以的；因为当集群创建好后，default namespace 下会创建一个叫 kubenretes 的 svc，有一些组件会直接连接这个 svc 来跟 api 通讯的，证书如果不包含可能会出现无法连接的情况；其他几个 kubernetes 开头的域名作用相同
    hosts包含的是授权范围，不在此范围的的节点或者服务使用此证书就会报证书不匹配错误。
    10.254.0.1是指kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP。
    #hosts配置区域，即一个证书的网站可以是*.youku.com也是可以是*.google.com
#生成kubernetes证书和私钥
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes 
       
 知识点： -config 引用的是模板中的默认配置文件，-profiles是指定特定的使用场景，比如ca-config.json中的kubernetes区域　　
```

##### 3、创建admin证书 

```java
cat admin-csr.json   #证书请求文件
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
              "ST": "JiangSu",
              "L": "NanJing",
              "O": "system:masters",
              "OU": "System"
            }
          ]
        }
```

*#生成admin证书和私钥
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
*

*知识点：
这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group*

转载于:https://www.cnblogs.com/LangXian/p/11282986.html