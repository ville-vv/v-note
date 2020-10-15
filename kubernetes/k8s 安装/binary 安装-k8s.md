
## 需要安装的工具组件

- cfssl
- etcd
- flannel
- kube-apiserver
- kube-proxy
- kube-scheduler
- kube-controller-manager
- kubelet

## 修改系统设置

1、关闭selinux(contos需要操作)

```
sed -i "s/SELINUX\=.*/SELINUX=disabled/g" /etc/selinux/config
```

2、关闭防火墙

不同系统方法不同：略

3、关闭交换分区

```
swapoff -a
```





## 安装 `cfssl` go 版本的 ca证书生成工具

**下载`cfssl`工具** ：地址 https://pkg.cfssl.org/ 

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
sudo mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```



## 安装 etcd

**下载 etcd 并配置环境变量** ， https://github.com/etcd-io/etcd/releases  下载文档版本etcd，并解压。

```
tar -xzvf etcd_xxxx.tar.gz -C /usr/local/xxx/etcd
```
建立软链接(或者直接配置环境变量到etcd可执行文件目录)

```
ln -s /usr/local/xxx/etcd/etcd /usr/local/bin/etcd
ln -s /usr/local/xxx/etcd/etcdctl /usr/local/bin/etcdctl
```

**创建证书存放路径** 

```
mkdir -p /etc/etcd/ssl
cd /etc/etcd/ssl
```

**编辑 ca-config.json** 

```
cat << EOF > ca-config.json
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "876000h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
```

**编辑 ca-csr.json**

```
cat << EOF > ca-csr.json
{
    "CN": "etcd-01",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "GZ",
            "ST": "ShenZhen",
            "O": "k8s",
            "OU": "System"
        }
    ],
    "ca": {
        "expiry": "876000h"
    }
}
EOF
```

**生成ca.pem 和ca-key.pem**

```
# 生成 ca.pem 和 ca-key.pem
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

```

**创建 etcd.csr.json**

```
cat << EOF > /etc/etcd/ssl/etcd-csr.json

{
    "CN": "etcd-01",
    "hosts": [
        "10.0.0.1",
        "10.1.0.1",
        "10.254.0.1",
        "127.0.0.1",
        "172.16.5.241"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C":  "CN",
            "L":  "GZ",
            "O":  "k8s",
            "OU": "System",
            "ST": "ShenZhen"
        }
    ]
}
EOF

```

**颁发证书**
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

```

**配置 etcd.service 启动文件**

cd 到目录 `/usr/lib/systemd/system` 并创建文件 etcd.service 文件。

```
cd /usr/lib/systemd/system
```
写入内容到 `etcd.service` 文件
```
cat << EOF > etcd.service
[Unit]
Description=Etcd Server
Documentation=https://github.com/coreos
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd  --name=${NAME} --data-dir=${DATA_DIR} --cert-file=${CERT_FILE} --key-file=${KEY_FILE} --peer-cert-file=${PEER_CERT_FILE} --peer-key-file=${PEER_KEY_FILE} --trusted-ca-file=${TRUSTED_CA_FILE} --peer-trusted-ca-file=${PEER_TRUSTED_CA_FILE} --initial-advertise-peer-urls=${INITIAL_ADVERTISE_PEER_URLS} --listen-client-urls=${LISTEN_CLIENT_URLS} --listen-peer-urls=${LISTEN_PEER_URLS} --advertise-client-urls=${ADVERTISE_CLIENT_URLS} --initial-cluster-token=${INITIAL_CLUSTER_TOKEN} --initial-cluster-state=${INITIAL_CLUSTER_STATE} "
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

写etcd启动的配置文件
```
cat << EOF > /etc/etcd/etcd.conf
NAME=etcd_ville
DATA_DIR="/var/lib/etcd/etcd_ville.etcd"
LISTEN_PEER_URLS="https://172.16.5.241:2380"
LISTEN_CLIENT_URLS="https://172.16.5.241:2379"
INITIAL_ADVERTISE_PEER_URLS="https://172.16.5.241:2380"
ADVERTISE_CLIENT_URLS="https://172.16.5.241:2379"
INITIAL_CLUSTER_STATE="new"
INITIAL_CLUSTER_TOKEN="etcd-cluster1"
CERT_FILE="/etc/etcd/ssl/etcd.pem"
KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
EOF
```

**启动etcd**

```
systemctl daemon-reload
systemctl start etcd.service
systemctl status etcd.service


# 停止服务
systemctl stop etcd.service
systemctl restart etcd.service
```


验证节点状态

```
export ETCDCTL_API=3
etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://172.16.5.241:2379 endpoint health
```



## 创建K8s需要的证书文件

**1、创建 kubernetes 证书签名请求文件** 

```
cat << EOF > /etc/kubernetes/ssl/kubernetes-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "10.254.0.1",
      "127.0.0.1",
      "172.16.5.241",
      "172.17.0.1",
      "172.18.0.1",
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
            "L": "GZ",
            "ST": "ShenZhen",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```
hosts 中的内容可以为空，即是按照上面的配置，向集群中增加新节点也不需要重新生成证书；如果 hosts 不为空，则需要指定授权使用该证书的IP或域名列表，由于该证书后续被etcd集群和kubernetes master集群使用，所以上面分别指定了etcd集群，kubernetes master集群的主机IP和kuberunetes服务ip



生成kubernetes证书和私钥，ca.pem 和 ca-key.pem 与 etcd 中的相同

```
cd /etc/kubernetes/ssl/
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```



**2、创建admin证书签名请求文件** 

```
cat << EOF > admin-csr.json
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
            "L": "GZ",
            "ST": "ShenZhen",
            "O": "system:masters",
            "OU": "System"
        }
    ]
}
EOF
```
- kube-apiserver使用RBAC对客户端（如Kubelet,kube-proxy,Pod）请求进行授权。
- kube-apiserver预定义了一些RBAC使用的RoleBindings，如cluster-admin将Group System：masters与Role cluster-admin绑定，该Role授予kube-apiserver的所有API的权限；
- OU指定该证书的Group为system:masters，kubelet使用该证书访问kube-apiserver时，由于证书为CA签名，所以认证通过，同时由于证书用户组为经过预授权的system：masters，所以被授予访问所有API的权限


生成 admin证书和私钥
```shell
#cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes admin-csr.json | cfssljson -bare admin

```



**3、创建kube-proxy证书签名请求文件** 

```
cat > kube-proxy-csr.json << EOF
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
            "L": "GZ",
            "ST": "ShenZhen",
            "O": "system:kube-proxy",
            "OU": "System"
        }
    ]
}
EOF
```
- CN指定该证书的User为kube-proxy；
- kube-apiserver预定义的RoleBinding cluster-admin将User kube-proxy与Role System：node-proxies绑定，该Role授予了调用kube-apiserver Proxy相关API的权限；

生成kube-proxy客户端证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

**4、创建kubelet证书签名请求文件** 

```
cat > /etc/kubernetes/ssl/kubelet-csr.json << EOF
{
  "CN": "system:node:172.16.5.241",
  "hosts": [
    "127.0.0.1",
    "10.254.0.1",
    "172.16.5.241"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "GZ",
      "O": "system:nodes",
      "OU": "System"
    }
  ]
}
EOF
```

生成kubelet客户端证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kubelet-csr.json | cfssljson -bare kubelet
```



## 安装 Flannel

#### 下载 Flannel 

到 https://github.com/coreos/flannel/releases 下载 Flannel，并且解压。

```shell
tar -xzvf flannel-v0.12.0-linux-amd64.tar.gz
```

复制 `flanneld`  `mk-docker-opts.sh` 文件到 /usr/loacl/bin 

```shell
cp flanneld mk-docker-opts.sh /usr/loacl/bin
```



#### 向etcd中写入网段信息 

Falnnel要用etcd存储自身一个子网信息，所以要保证能成功连接Etcd，写入预定义子网段，etcd 集群只需要写在一台即可：
```
etcdctl --endpoints="https://172.16.5.241:2379"  --ca-file=/etc/etcd/ssl/ca.pem  --cert-file=/etc/etcd/ssl/etcd.pem  --key-file=/etc/etcd/ssl/etcd-key.pem  set /kubernetes/network/config '{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'
```

> 注意：这里有一个很大的坑， 我使用的 etcd 是 3.3.x 版本，etcdctl 切换到了 v3 的api, 写入的数据在flannel中读取会出错，所有在 使用 etcdctl 写入数据的时候必须保证etcdctl 使用的是 api v2版本。因为 flannel目前只支持 v2 格式的数据。
>
> etcdctl 默认是使用 v2

检验etcd的数据

```
etcdctl --endpoints="https://172.16.5.241:2379"  --ca-file=/etc/etcd/ssl/ca.pem  --cert-file=/etc/etcd/ssl/etcd.pem  --key-file=/etc/etcd/ssl/etcd-key.pem  get /kubernetes/network/config
```


 **创建启动文件** 

 ```
cat > /usr/lib/systemd/system/flanneld.service << EOF
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/local/bin/flanneld \
  -etcd-cafile=/etc/etcd/ssl/ca.pem \
  -etcd-certfile=/etc/etcd/ssl/etcd.pem \
  -etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  -etcd-endpoints=https://172.16.5.241:2379 \
  -etcd-prefix=/kubernetes/network
ExecStartPost=/usr/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
 ```
- `mk-docker-opts.sh`：将分配给 `flanneld` 的 `pod` 子网网段信息写入到/run/flannel/docker文件中，后续docker启动时使用这个文件中参数值设置docker0网桥
- `flanneld` 使用系统缺省路由所在的接口和其他节点通信，对于有多个网络接口的机器（如内网和公网），可使用-iface=enpxx选项值指定通信接口；

**修改docker配置** 


```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker

# 新增参数
EnvironmentFile=/run/flannel/docker

# 注释原来的启动配置
# ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# 修改为
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS


ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity

# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target

```

**启动Flannel** 

```
systemctl daemon-reload
systemctl start flanneld.service

journalctl -u flanneld -f
```



##  k8s Master 节点组件部署

### 下载 k8s 二进制组件执行程序

github下载地址：https://github.com/kubernetes/kubernetes/releases

解压 kubernetes-server-linux-amd64.tar.gz,  二进制文件在  /kubernetes/server/bin 中，把二进制文件 kube-apiserver, kube-controller-manager , kube-scheduler 复制到 /opt/kubernetes/bin/, 把 二进制文件 kubelet , kubectl, kube-proxy 复制到 /usr/local/bin 中

```
cd  ./kubernetes/server/bin
mkdir  /opt/kubernetes/bin/
cp  kube-apiserver, kube-controller-manager , kube-scheduler /opt/kubernetes/bin/
cp  kubelet , kubectl, kube-proxy  /usr/local/bin
```

### 安装 kube-apiserver

**配置 kube-apiserver 启动文件** 

```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=/etc/kubernetes/conf/apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver ${KUBE_APISERVER}
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

创建配置文件
```
cat << EOF > /etc/kubernetes/conf/apiserver.conf
# --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook 

#  --kubelet-certificate-authority=/etc/kubernetes/ssl/ca.pem \
#  --kubelet-client-certificate=/etc/kubernetes/ssl/admin.pem \
#  --kubelet-client-key=/etc/kubernetes/ssl/admin-key.pem \

KUBE_APISERVER_ARGS="--logtostderr=true \
  --allow-privileged=true \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \ 
  --bind-address=172.16.5.241 \
  --insecure-bind-address=127.0.0.1 \
  --authorization-mode=Node,RBAC \
  --kubelet-https=true \
  --anonymous-auth=false \
  --service-cluster-ip-range=10.254.0.0/16 \
  --service-node-port-range=20000-60000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --etcd-servers=https://172.16.5.241:2379 \
  --enable-swagger-ui=true \
  --enable-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/token.csv \
  --endpoint-reconciler-type=lease \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/lib/audit.log \
  --event-ttl=1h \
  --proxy-client-cert-file=/etc/kubernetes/ssl/kube-proxy.pem \
  --proxy-client-key-file=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --enable-aggregator-routing=true \
  --runtime-config=rbac.authorization.k8s.io/v1beta1 \
  --v=2"
EOF
```
具体参数解释：https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/

- --authorization-mode=RBAC 指定在安全端口使用RBAC模式，拒绝未通过授权的请求；
- kube-scheduler，kube-controller-manager一般和kube-apiserver部署在同一台机器上，他们使用非安全端口和kube-apiserver通信；
- kubelet，kube-proxy，kubectl部署在其他Node节点，如果通过安全端口访问kube-apiserver，则必须先通过TLS证书认证，再通过RBAC授权；
- kube-proxy，kubectl通过在使用的证书里指定相关的User，Group来达到通过RBAC授权的目的。
- Bootstartp：如果使用了kubelet TLS Bootstartp机制，则不能再指定 –kubelet-certificate-authority、–kubelet-client-certificate 和 –kubelet-client-key 选项，否则后续kube-apiserver校验kubelet证书时出现”x509: certificate signed by unknown authority“ 错误；
- --admission-control值必须包含ServiceAccount，否则部署集群插件时会失败；
- --bind-address不能为127.0.0.1；
- --runtime-config：配置rbac.authorization.k8s.io/v1beta1，表示运行时的apiVersion；
- service-cluster-ip-range：指定Service cluster ip段地址，该地址路由不可达；
- --service-node-port-range：指定NodePort的端口范围

```
systemctl daemon-reload
systemctl start kube-apiserver.service
systemctl status kube-apiserver.service


# 查看日志
journalctl -u kube-apiserver -f
```



### 部署 kube-controller-manager

```
cat << EOF > /usr/lib/systemd/system/kube-controller-manager.service

[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=-/etc/kubernetes/conf/controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

创建配置文件
```
cat << EOF > /etc/kubernetes/conf/controller-manager.conf
KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 \
  --logtostderr=true \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.254.0.0/16 \
  --cluster-cidr=172.30.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --node-cidr-mask-size=24 \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --leader-elect=true \
  --v=2"
EOF

```
- --address的值必须为127.0.0.1,应为当前kube-apiserver期望scheduler和conntroller-manaager在同一台机器上；
- --master=http://{master_ip}:8080：使用非安全的8080端口与kube-apiserver通信；
- --cluster-cidr指定Cluster中Pod的CIDR范围，该网段在各Node必须路由可达（flanneld保证）
- --service-cluster-ip-range参数指定Cluster中Service的CIDR范围，该网络在各Node间必须路由不可达，必须与kube-apiserver中的参数保持一致；
- --cluster-signing-*指定的证书和私钥文件用来签名TLS BootStrap创建的证书和私钥
- --root-ca-file用来对kube-apiserver证书进行校验，指定该参数后，才会在Pod容器的ServiceAccount中放置该CA证书文件
- --leader-elect=true部署多台master集群时选举产生一直处于工作状态的kube-controller-manager进程；



```
systemctl daemon-reload
systemctl start kube-controller-manager.service
systemctl status kube-controller-manager.service


# 查看日志
journalctl -u kube-controller-manager -f
```


### 部署 kube-scheduler

```
cat << EOF > /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=-/etc/kubernetes/conf/scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_ARGS
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

创建配置文件

```
cat << EOF > /etc/kubernetes/conf/scheduler.conf
KUBE_SCHEDULER_ARGS="--logtostderr=true \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2"
EOF
```
- --address必须为127.0.0.1，应为当前kube-apiserver期望scheduler和contorller-manager在同一主机；
- master=http://{MASTER_IP}:8080：使用非安全 8080 端口与 kube-apiserver 通信；
- –leader-elect=true 部署多台机器组成的 master 集群时选举产生一处于工作状态的 kube-controller-manager 进程；


```
systemctl daemon-reload
systemctl start kube-scheduler.service
systemctl status kube-scheduler.service


# 查看日志
journalctl -u kube-scheduler -f
```


### kubectl 工具

kubectl是kubernetes的集群管理工具，任何节点通过kubetcl都可以管理整个k8s集群。部署成功后会生成 $HOME/.kube/config文件，kubectl就是通过这个获取kube-apiserver地址，证书，用户名等信息

**1、创建 $HOME/.kube/config 文件**

```
cd /etc/kubernetes/
# 设置集群参数,--server指定Master节点ip
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://172.16.5.241:6443

# 设置客户端认证参数
kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem

# 设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin

# 设置默认上下文
kubectl config use-context kubernetes
```

- admin.pem：证书O字段值为system:masters，kube-apiserver预定义的RoleBinding cluster-admin将Group system:master与Role cluster-admin绑定，该Role 授予了调用Kube-apiserver相关的API权限

查看是否创建成功
```
cat $HOME/.kube/config
```


**2、创建bootstartp.kubeconfig文件**

```
#生成token 变量
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')

cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

mv token.csv /etc/kubernetes/

# 设置集群参数--server为master节点ip
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://172.16.5.241:6443 \
  --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

mv bootstrap.kubeconfig /etc/kubernetes/
```

**创建kube-proxy.kubeconfig**
```
# 设置集群参数 --server参数为master ip
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://172.16.5.241:6443 \
  --kubeconfig=kube-proxy.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
mv kube-proxy.kubeconfig /etc/kubernetes/
```
- 设置集群参数和客户端认证参数，--embed-certs都为true，这会将certificate-authority，client-cretificate和client-key指向的证书文件内容写入到生成的kube-proxy.kebuconfig文件中；
- kube-proxy.pem证书中CN为system:kube-proxy,kube-apiserver预定义的RoleBinding cluster-admin将User system:kube-proxy与Role system:node-proxy绑定，该Role授予了调用kube-apiserver Proxy相关的api权限；




### 验证服务

```
# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}     
```

以上是在部署 kubernetes 集群服务中，master 节点完成。



## k8s Node 节点组件部署

> 备注：我这里 node节点和master节点是在同一台机器上。



### 部署 kubelet

kubelet在启动时向kube-apiserver发送TLS bootstrapping请求，需要先将bootstrap token文件中的kubelet-bootstrap用户赋予system:node-bootstrapper角色，然后kubelet才有权限创建认证请求，授权,在master上运行一次即可。

```
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

创建kubelet工作目录

```
mkdir /var/lib/kubelet
```
配置kubelt

```
cat << EOF > /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

#--bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig
#--cert-dir=/etc/kubernetes/ssl/

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/conf/kubelet.conf
ExecStartPre=/usr/sbin/swapoff -a
ExecStart=/usr/local/bin/kubelet --address=172.16.5.241 \
  --hostname-override=172.16.5.241 \
  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cni-conf-dir=/etc/cni/net.d \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --authentication-token-webhook \
  --authorization-mode=Webhook \
  --cluster-dns=10.254.0.2 \
  --cluster-domain=cluster.local \
  --hairpin-mode=hairpin-veth \
  --register-node=true \
  --max-pods=200 \
  --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest  \
  --root-dir=/var/lib/kubelet \
  --v=2
Restart=on-failure
KillMode=process
RestartSec=5

[Install]
WantedBy=multi-user.target

```

写入配置

```
cat << EOF > /etc/kubernetes/conf/kubelet.conf
KUBELET_ARGS="--address=172.16.5.241 \
  --hostname-override=172.16.5.241 \
  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cni-conf-dir=/etc/cni/net.d \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --authentication-token-webhook \
  --authorization-mode=Webhook \
  --cluster-dns=10.254.0.2 \
  --cluster-domain=cluster.local \
  --hairpin-mode=hairpin-veth \
  --register-node=true \
  --max-pods=200 \
  --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest  \
  --root-dir=/var/lib/kubelet \
  --v=2"
EOF
```

- --address：本机IP，不能设置为127.0.0.1，否则后续Pods访问kubelet的API接口时会失败，因为 Pods 访问的 127.0.0.1 指向自己，而不是 kubelet;

- --hostname-overeide：本机IP；

- --cgroup-driver配置成cgroupfs（保持docker和kubelet中的cgroup driver配置一致即可）；

- --bootstrap-kubeconfig指向bootstrap.kubeconfig文件，kubelet使用该文件中的用户名和token向kube-apiserver发送TLS Bootstrapping请求；管理员通过了CSR请求后，kubelet自动在--cert-dir目录创建证书和私钥文件（kubelet-client.crt和kubelet-client.key）。

- --cluster-dns指定kubedns的Service ip(可以先分配，后续创建kubedns服务时指定该IP)，
- --cluster-domain指定域名后缀，这两个参数同时配置才会生效；

- --cluster-domain指定pod启动时/etc/resolve.conf文件中的search domain，起初我们将其配置成了 cluster.local.，这样在解析 service 的 DNS 名称时是正常的，可是在解析 headless service 中的 FQDN pod name 的时候却错误，因此我们将其修改为 cluster.local，去掉嘴后面的 ”点号“ 就可以解决该问题；

- --kubeconfig=/etc/kubernetes/kubelet.kubeconfig中指定的kubelet.kubeconfig文件在第一次启动kubelet之前并不存在，请看下文，当通过CSR请求后，会自动生成，如果你的节点节点上已经生成了~/.kube/config文件，你可以将该文件拷贝到该路径面，并命名为kubelet.kubeconfig文件，所有的节点可以共用同一个config文件，这样新添加节点时就不需要再创建CSR请求就能自动添加到kubernetes集群中，同样，在任意能够访问到kubernetes集群的主机上使用kubectl --kubeconfig命令操作集群，只要使用~/.kube/config文件就能通过权限认证，应为这个文件的认证信息为admin，对集群有所有权限。

**拷贝 $HOME/.kube/config**

```
cp $HOME/.kube/config /etc/kubernetes/kubelet.kubeconfig
```


**启动**
```
swapoff -a
systemctl daemon-reload
systemctl start kubelet.service
systemctl status kubelet.service

# 查看日志
journalctl -u kubelet -f
```

**查看node**

```
root@xxx:~# kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
172.16.5.241   Ready    <none>   10s   v1.18.9

```



### 部署 kube-proxy

创建工作目录
```
mkdir /var/lib/kube-proxy
```

创建启动文件

```
cat << EOF > /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
# kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后
# kube-proxy 会对访问 Service IP 的请求做 SNAT，这个特性与calico 实现 network policy冲突，因此禁用
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy --config=/etc/kubernetes/conf/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

创建配置文件
```
cat << EOF > /etc/kubernetes/conf/kube-proxy-config.yaml

apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 172.16.5.241
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 10.254.0.0/16
configSyncPeriod: 15m0s
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
detectLocalMode: ""
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: 172.16.5.241
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: ""
  strictARP: false
  syncPeriod: 30s
  tcpFinTimeout: 0s
  tcpTimeout: 0s
  udpTimeout: 0s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: ""
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
showHiddenMetricsForVersion: ""
udpIdleTimeout: 250ms
winkernel:
  enableDSR: false
  networkName: ""
  sourceVip: ""
EOF
```

加载服务

```
systemctl daemon-reload
systemctl start kube-proxy.service
systemctl status kube-proxy.service

# 查看日志
journalctl -u kube-proxy -f
```



## 部署网络插件

### 获取 coredns.yaml

将下载的 kubernetes-server-linux-amd64.tar.gz 解压后，再解压其中的 kubernetes-src.tar.gz 文件。coredns 对应的目录是：cluster/addons/dns

```
cd xxx/kubernetes/cluster/addons/dns/coredns
```

将coredns模板复制出来

```
cp coredns.yaml.base  coredns.yaml
```

修改coredns.yaml 参数
- 修改 `__PILLAR__DNS__DOMAIN__` 为 `cluster.local.`（自己部署k8s时设置的域名） 
```
#原来 
kubernetes __PILLAR__DNS__DOMAIN__ in-addr.arpa ip6.arpa

改为
kubernetes cluster.local. in-addr.arpa ip6.arpa

```

 - 修改 `image: k8s.gcr.io/coredns:1.6.5` 为 `coredns/coredns:1.6.5`

```
 原来
image: k8s.gcr.io/coredns:1.6.5

改为
image: coredns/coredns:1.6.5
```

- 修改 `__PILLAR__DNS__MEMORY__LIMIT__` 为 170Mi

```
原来
memory: __PILLAR__DNS__MEMORY__LIMIT__

改为
memory: 170Mi

```

- 修改 `__PILLAR__DNS__SERVER__` 为 `10.254.0.2`（ip是部署k8s时设置的集群ip,可以使用 kubectl get service 查看 ）
```
原来
clusterIP: __PILLAR__DNS__SERVER__

改为
clusterIP: 10.254.0.2
```

绑定一个cluster-admin的权限
```
kubectl create clusterrolebinding system:anonymous   --clusterrole=cluster-admin   --user=system:anonymous
```


创建插件

```shell
root@xxx:~# kubectl apply -f coredns.yaml 
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
```

查看情况 
```shell
root@xxx:~# kubectl get all -n kube-system
NAME                           READY   STATUS    RESTARTS   AGE
pod/coredns-55f9545c85-6t6v4   1/1     Running   0          43m

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.254.0.2   <none>        53/UDP,53/TCP,9153/TCP   43m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   1/1     1            1           43m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-55f9545c85   1         1         1       43m


root@xxx:~# kubectl  get pods -o wide -n=kube-system
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
coredns-55f9545c85-6t6v4   1/1     Running   0          45m   172.30.89.2   172.16.5.241   <none>           <none>


root@xxx:~# kubectl  get pods -o wide -n=kube-system
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
coredns-55f9545c85-6t6v4   1/1     Running   0          48m   172.30.89.2   172.16.5.241   <none>           <none>


root@xxx:~# kubectl  cluster-info
Kubernetes master is running at https://172.16.5.241:6443
CoreDNS is running at https://172.16.5.241:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

> 注意：coredns报错【[FATAL] plugin/loop: Loop (127.0.0.1:55710 -> :53) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 6229007223346367857.5322622110587761536."】
原因：是因为dns设置了包含127.0.x.x导致的。

临时解决方法，修改 vim /etc/resolv.conf 文件内容,把 nameserver 127.0.0.x 修改为 114.114.114.114
```
#nameserver 127.0.0.53
nameserver 114.114.114.114
```

可永久解决方法一
```
Install the resolvconf package.

sudo apt install resolvconf
Edit /etc/resolvconf/resolv.conf.d/head and add the following:

# Make edits to /etc/resolvconf/resolv.conf.d/head.
nameserver 8.8.8.8
nameserver 114.114.114.114
Restart the resolvconf service.

sudo service resolvconf restart
```

永久解决方法二

```
vim /etc/systemd/resolved.conf

# 写入内容
[Resolve]
DNS=114.114.114.114 8.8.8.8
```





## 部署 dashboard

**下载 dashboard.yaml文件**

```
wget  https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```
修改文件内容

```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  # 新增
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      # 新增
      nodePort: 28510
  selector:
    k8s-app: kubernetes-dashboard
```

创建 dashboard
```shell
ville:~$ kubectl apply -f recommended.yaml 
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

查看情况

```
ville:~$ kubectl get all -n=kubernetes-dashboard
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-6b4884c9d5-nj2hx   1/1     Running   0          28m
pod/kubernetes-dashboard-7b544877d5-b2r46        1/1     Running   0          28m

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP   10.254.153.12   <none>        8000/TCP        28m
service/kubernetes-dashboard        NodePort    10.254.56.135   <none>        443:28510/TCP   28m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dashboard-metrics-scraper   1/1     1            1           28m
deployment.apps/kubernetes-dashboard        1/1     1            1           28m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/dashboard-metrics-scraper-6b4884c9d5   1         1         1       28m
replicaset.apps/kubernetes-dashboard-7b544877d5        1         1         1       28m

```

查看 secret
```shell
ville:~$ kubectl get secret -n=kubernetes-dashboard
NAME                               TYPE                                  DATA   AGE
default-token-9kst5                kubernetes.io/service-account-token   3      11m
kubernetes-dashboard-certs         Opaque                                0      11m
kubernetes-dashboard-csrf          Opaque                                1      11m
kubernetes-dashboard-key-holder    Opaque                                2      11m
kubernetes-dashboard-token-tc2sl   kubernetes.io/service-account-token   3      11m

```


新增 dashboard 超级管理员用户

```
cat << EOF > dashboard-cluster-role-bind.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

创建角色绑定
```shell
ville:~$ kubectl apply -f dashboard-cluster-role-bind.yaml
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created

```

再次查看 secret 

```shell
ville:~$ kubectl get secret -n=kubernetes-dashboard
NAME                               TYPE                                  DATA   AGE
admin-user-token-89dnx             kubernetes.io/service-account-token   3      10s
default-token-9kst5                kubernetes.io/service-account-token   3      15m
kubernetes-dashboard-certs         Opaque                                0      15m
kubernetes-dashboard-csrf          Opaque                                1      15m
kubernetes-dashboard-key-holder    Opaque                                2      15m
kubernetes-dashboard-token-tc2sl   kubernetes.io/service-account-token   3      15m
```

获取管理员 secret token

```shell
ville:~$ kubectl describe secret admin-user-token-89dnx -n=kubernetes-dashboard
Name:         admin-user-token-89dnx
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 8deaeca1-1fed-433d-94b5-13a5af074870

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1350 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjJUXzdCdzdoVjB3Q0R0MmFycUpoUnpTc3FjX1o1bjJEN0NzWHRxNkxfSHMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTg5ZG54Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI4ZGVhZWNhMS0xZmVkLTQzM2QtOTRiNS0xM2E1YWYwNzQ4NzAiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.Uc9-C-7DIqPHDRyws-f7W3jCHztgpi0Lmpb-hTC5zODrWzPDHwayvEZ4pEQ1NDi6ODs6GsDl5uVFsT_EDFb585PlOjkcoDrOvEtTkusFA2p0UjbvzfPjlCnZYkCxZSiyqlRnoXpZLeJdKRz5nI-ouBvtPhRlg16KgRtU6qUIxxA7z7xZI5bi8tJj277O5BnF70lYzWyyVmyybcBMyN2E0KJdYVCUZsvXDfRjOZwfTNs8HuWPXgZELek_K1jmoGxjilhZL0iv2XBLkmPQ7-pOAaWNoPo_BAV0Sy-XUY28WeBNhTmlKhj6UxAmAojNkaxhjqlmEs1ba7kTU7MD40_BlA
```

获取登录dashboard端口地址
```shell
ville:~$ kubectl get service -n=kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.254.153.12   <none>        8000/TCP        35m
kubernetes-dashboard        NodePort    10.254.56.135   <none>        443:28510/TCP   35m
```
dashboard 登录地址为 https://您的本机IP:28510, 我这里是 https://172.16.5.241:28510 
登录 `dashboard` 使用上面方法获取到的 token 认证即可。



## 重启 k8s


```
systemctl daemon-reload

systemctl stop kube-scheduler.service
systemctl stop kube-controller-manager.service
systemctl stop kubelet.service
systemctl stop kube-proxy.service
systemctl stop kube-apiserver.service
systemctl stop etcd.service

systemctl start kube-apiserver.service
systemctl start kube-controller-manager.service
systemctl start kube-scheduler.service
systemctl start kubelet.service
systemctl start kube-proxy.service
```



## 清理 k8s 环境

```
rm -rf /var/lib/etcd
rm -rf /var/lib/kubelet/
rm -rf /var/lib/kube-proxy/
rm -rf $HOME/.kube/
rm -rf /etc/kubernetes/ssl/*.pem
rm -rf /etc/kubernetes/*.kubeconfig
rm -rf /etc/kubernetes/*.pem

```




