
## 需要安装的工具组件

- etcd
- flannel
- kube-apiserver
- kube-proxy
- kube-scheduler
- kube-controller-manager



## 安装步骤


### 一 、安装 etcd

**下载 etcd 并配置环境变量**

下载地址：

解压到指定目录

```
tar -xzvf etcd_xxxx.tar.gz -C /usr/local/xxx/etcd
```
建立软链接(或者直接配置环境变量到etcd可执行文件目录)

```
ln -s /usr/local/xxx/etcd/etcd /usr/local/bin/etcd
ln -s /usr/local/xxx/etcd/etcdctl /usr/local/bin/etcdctl
```

**创建 ssl 证书**

创建 `/etc/etcd/ssl`

```
cd /etc
mkdir -p /etcd/ssl
cd ./etcd/ssl
```
写入如下内容到 etcd.csr.json

```
cat << EOF > etcd-csr.json
{
    "CN": "etcd",
    "hosts": [
        "10.0.0.1",
        "127.0.0.1",
        "172.16.5.241",
        "172.17.0.1",
        "172.18.0.1",
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
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/opt/kube/bin/etcd \
  --name=etcd1 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://172.16.0.30:2380 \
  --listen-peer-urls=https://172.16.0.30:2380 \
  --listen-client-urls=https://172.16.0.30:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://172.16.0.30:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=etcd1=https://172.16.0.30:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```


#### kube-apiserver.service

```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook \
  --bind-address=172.16.0.30 \
  --insecure-bind-address=127.0.0.1 \
  --authorization-mode=Node,RBAC \
  --kubelet-https=true \
  --kubelet-client-certificate=/etc/kubernetes/ssl/admin.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/admin-key.pem \
  --anonymous-auth=false \
  --service-cluster-ip-range=10.68.0.0/16 \
  --service-node-port-range=20000-40000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://172.16.0.30:2379 \
  --enable-swagger-ui=true \
  --endpoint-reconciler-type=lease \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/lib/audit.log \
  --event-ttl=1h \
  --requestheader-client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --requestheader-allowed-names= \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --proxy-client-cert-file=/etc/kubernetes/ssl/aggregator-proxy.pem \
  --proxy-client-key-file=/etc/kubernetes/ssl/aggregator-proxy-key.pem \
  --enable-aggregator-routing=true \
  --runtime-config=batch/v2alpha1=true \
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```


#### kube-controller-manager.service

```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kube/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.68.0.0/16 \
  --cluster-cidr=172.20.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --node-cidr-mask-size=24 \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```


#### kubelet.service

```
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/memory/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/pids/system.slice/kubelet.service
ExecStart=/opt/kube/bin/kubelet \
  --address=172.16.0.30 \
  --allow-privileged=true \
  --anonymous-auth=false \
  --authentication-token-webhook \
  --authorization-mode=Webhook \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-dns=10.68.0.2 \
  --cluster-domain=cluster.local. \
  --cni-bin-dir=/opt/kube/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --fail-swap-on=false \
  --hairpin-mode hairpin-veth \
  --hostname-override=172.16.0.30 \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --max-pods=200 \
  --network-plugin=cni \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.1 \
  --register-node=true \
  --registry-qps=200 \
  --registry-burst=200 \
  --root-dir=/var/lib/kubelet \
  --tls-cert-file=/etc/kubernetes/ssl/kubelet.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubelet-key.pem \
  --cgroups-per-qos=true \
  --cgroup-driver=cgroupfs \
  --enforce-node-allocatable=pods,kube-reserved \
  --kube-reserved=cpu=200m,memory=500Mi,ephemeral-storage=1Gi \
  --kube-reserved-cgroup=/system.slice/kubelet.service \
  --eviction-hard=memory.available<200Mi,nodefs.available<10% \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```




#### kube-proxy.service

```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
# kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后
# kube-proxy 会对访问 Service IP 的请求做 SNAT，这个特性与calico 实现 network policy冲突，因此禁用
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kube/bin/kube-proxy \
  --bind-address=172.16.0.30 \
  --hostname-override=172.16.0.30 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --proxy-mode=iptables
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

#### kube-scheduler.service

```
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kube/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```


### ssl 证书

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
    "CN": "kubernetes",
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

**ca-apiserver-csr.json**

```
cat << EOF > ca-apiserver-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
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

**ca-etcd-csr.json**

```
cat << EOF > ca-etcd-csr.json
{
    "CN": "etcd",
    "hosts": [
        "10.0.0.1",
        "127.0.0.1",
        "172.16.5.241",
        "172.17.0.1",
        "172.18.0.1",
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

**admin-csr.json**
```
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
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```


生成CA证书（ca.pem）和密钥（ca-key.pem）

```shell
# 生成 ca.pem 和 ca-key.pem
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# 请求颁发证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes ca-apiserver-csr.json | cfssljson -bare apiserver

```


生成 admin 证书和私钥

```
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

