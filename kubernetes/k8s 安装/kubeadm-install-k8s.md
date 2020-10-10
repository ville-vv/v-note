# kubeadm 安装 k8s

kubeadm command 

```shell
--apiserver-advertise-address string    #API Server 将要监听的监听地址
--apiserver-bind-port int32 #API Server 绑定的端口,默认为 6443,
--apiserver-cert-extra-sans stringSlice #可选的证书额外信息，用于指定 API Server 的服务器证书。可以是 IP 地址也可以是 DNS 名称。
--cert-dir string #证书的存储路径，缺省路径为 /etc/kubernetes/pki --config string #kubeadm 配置文件的路径

--ignore-preflight-errors strings #可以忽略检查过程 中出现的错误信息，比如忽略swap，如果为 all 就忽略所有
--image-repository string #设置一个镜像仓库，默认为 k8s.gcr.io --kubernetes-version string #选择 k8s 版本，默认为 stable-1 --node-name string #指定 node 名称
--pod-network-cidr #设置 pod ip 地址范围
--service-cidr #设置 service 网络地址范围
--service-dns-domain string #设置域名，默认为 cluster.local
--skip-certificate-key-print    #不打印用于加密的 key 信息
--skip-phases strings #要跳过哪些阶段
--skip-token-print  #跳过打印 token 信息
--token #指定 token
--token-ttl #指定 token 过期时间，默认为 24 小时，0 为永不过期
--upload-certs  #更新证书

#全局选项
--log-file string #日志路径
--log-file-max-size uint #设置日志文件的最大大小，单位为兆，默认为 1800，0 为没有限制 --rootfs #宿主机的根路径，也就是使用绝对路径
--skip-headers  #为 true，在 log 日志里面不显示消息的头部信息
--skip-log-headers  #为 true 在日志文件里面不记录头部信息
```



#### kubeadm init

```shell
sudo kubeadm init --image-repository registry.aliyuncs.com/google_containers   --ignore-preflight-errors=all  --pod-network-cidr=192.168.0.0/16
```

这里指定 --pod-network-cidr=192.168.0.0/16 是为下面的 pod 网络插件做准备

输出以下类似内容

```shell
[init] Using Kubernetes version: vX.Y.Z
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
.......
from directory "/etc/kubernetes/manifests". This can take up to 4m0s
.......
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

配置 kubelet 访问相关内容

```shell
#先删除内容
rm -rf $HOME/.kube

# 再执行下面操作
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 节点隔离

默认情况下，不会在主节点调用度 pod,如果你是单机部署，pod运行在主节点上，会需要调度pod,请运行下面命令：

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

官方的说明

> This will remove the `node-role.kubernetes.io/master` taint from any nodes that have it, including the control-plane node, meaning that the scheduler will then be able to schedule Pods everywhere.



#### k8s 网络插件

官方文档中提供多种网络插件其中包括 [Calico](https://docs.projectcalico.org/latest/introduction/) ，[Cilium](https://docs.cilium.io/en/stable/kubernetes/), [Contiv-VPP](https://contivpp.io/)  等等。这里因为在`kubeadm init` 中指定了 --pod-network-cidr=192.168.0.0/16 参数，Calico 也默认使用 192.168.0.0/16 所以这里就安装 [Calico](https://docs.projectcalico.org/getting-started/kubernetes/quickstart) 的网络插件

```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

> 如果想使用 192.168.0.0/16 在你的局域网中存冲突，可以修改为其他地址，但是 calico.yaml 文件中的地址也要做相应的修改。

查看网络插件启动状态(这里要多等待一会儿)  

```shell
[root@xxx]# kubectl get pods -n kube-system
calico-kube-controllers-6fcbbfb6fb-skxdp   1/1     Running   0          4h
calico-node-zj6fd                          1/1     Running   0          4h
coredns-7ff77c879f-cfqgr                   1/1     Running   0          4h2m
coredns-7ff77c879f-tgx26                   1/1     Running   0          4h2m
etcd-vil                                   1/1     Running   1          4h2m
kube-apiserver-vil                         1/1     Running   1          4h2m
kube-controller-manager-vil                1/1     Running   2          4h2m
kube-proxy-c95rt                           1/1     Running   1          4h2m
kube-scheduler-vil                         1/1     Running   3          4h2m

```



## 安装 kubernetes-dashboard

下载 dashboard.yaml 文档 

```shell
wget http://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

因为使用默认的 yaml 文档创建的 dashboard 是没有暴露端口给外部访问。可以使用以下三种方法让外部访问：

-  port-forward 创建端口映射，
-  kubectl proxy 。
- NodePort

这里使用 NodePort 方法。修改刚刚下载的 recommended.yaml 文档,找到 kubernetes-dashboard 的 Service 内容，示例如下：

```yaml
......
# 修改前内容
kind: Service
 apiVersion: v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 spec:
   ports:
     - port: 443
       targetPort: 8443
   selector:
     k8s-app: kubernetes-dashboard
......
```

 添加  type: NodePort 和 nodePort: 32134， 32134 是你要暴露的端口

```yaml
......
#修改后内容
kind: Service
 apiVersion: v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 spec:
   type: NodePort
   ports:
     - port: 443
       targetPort: 8443
       nodePort: 32134
   selector:
     k8s-app: kubernetes-dashboard
......
```

创建 dashboard

```
kubectl apply -f  recommended.yaml
```

查看 dashboard 是否启动, 如果启动  STATUS 为 Running

```shell
[root@xxx ]# kubectl -n kubernetes-dashboard get pods 
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-6b4884c9d5-zmkh6   1/1     Running   0          92m
kubernetes-dashboard-7b544877d5-fxbbm        1/1     Running   0          50m
```

尝试使用浏览器访问 https://ip:32134, 如果是谷歌浏览器会提示不能访问。是因为 dashboard https 的证书的原因。



## 创建证书

#### 使用 OpenSSL 创建证书

- 生成证书秘钥

```
openssl genrsa -des3 -passout pass:over4chars -out dashboard.pass.key 2048
```

- 生成私钥

```
openssl rsa -passin pass:over4chars -in dashboard.pass.key -out dashboard.key
# Writing RSA key
```

- 生成根证书

```
openssl req -new -key dashboard.key -out dashboard.csr
......
Country Name (2 letter code) [AU]:ZH
.....
内容自己随便填
```

- 证书签名

```
openssl x509 -req -sha256 -days 3650 -in dashboard.csr -signkey dashboard.key -out dashboard.crt
```

最后得到四个文件  `dashboard.crt`,  `dashboard.csr` , `dashboard.key` ,  `dashboard.pass.key` 我们后面需要用的只有 `dashboard.crt`  和 `dashboard.key` 



#### 重新创建 kubernetes-dashboard-certs

- 查看 kubernetes-dashboard-certs

```shell
[root@xxx]# kubectl -n kubernetes-dashboard get secret
default-token-7qvwn                kubernetes.io/service-account-token   3      108m
kubernetes-dashboard-certs         Opaque                                2      69m
kubernetes-dashboard-csrf          Opaque                                1      108m
kubernetes-dashboard-key-holder    Opaque                                2      108m
kubernetes-dashboard-token-l2zg6   kubernetes.io/service-account-token   3      108m
```

- 删除 kubernetes-dashboard-certs 

```
kubectl -n kubernetes-dashboard delete secret kubernetes-dashboard-certs
```

- 重新创建 dashboard 的 kubernetes-dashboard-certs

```shell
kubectl create secret generic kubernetes-dashboard-certs --from-file= $youpath/dashboard.key  --from-file=$youpath/dashboard.crt  -n kubernetes-dashboard
```

- 重启 kubernetes-dashboard , 删除pod会自动重启

```
kubectl -n kubernetes-dashboard delete pod kubernetes-dashboard
```

再次使用谷歌浏览器访问 dashboard  https://ip:port 会出现 token认证页面



## Dashboard login token

登陆 Dashboard 需要token, dashboard 创建的时候有一个默认的  token 项 可以使用下面命令获取

```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep "default-token" | awk '{print $1}')
```

将输出 的 token 复制到login页面即可，示例如下：

```
Name:         default-token-7qvwn
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 4e149142-87f4-4fe9-af38-579adee697f4

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InJpcWFjMEI4S2ZmVjdZYTRzLS1vT1k0cGg0aGwzbWNPUVhBMlpLSjNhNjQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVz5pby9zZXJ2aWNlYWNjb3VumVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLTdxdnduIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI0ZTE0OTE0Mi04N2Y0LTRmZTktYWYzOC01NzlhZGVlNjk3ZjQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6ZGVmYXVsdCJ9.rxdD_kuwKHFRaMyzJWMqZi8ZKSoLNor3yMelPgw7knZreovMXEEYKoHAzOXriC3rGYcBZbTH7iN-4d-2zUdMU5pby9zZXJ2aWNlYWNjb3VuNmp91nJKdM9kpZb5q7L2wgEZFys-3jfWfcRQv__kx0tVQ5nvmrt6Qt46rXEO2QJf2CkvNKYnyEFz9Qk4eDt0S4SxFBQQaxlXesdeU13KJxCmx6qHHWactXr5GHs5QjQnowSzmdSRKOhqOC2X4PnNyfhbXApdgvrShUA2QA1xy3qysWsT3Ffqet6slStqCtcNPoQXEp80yQyt42WYtRxdD-3DUovEWQLOytsBUwp5SrxChA

```

> 使用默认token 登录 dashboard 是不能做更改操作。 如果需要更改操作，可以创建 admin-user 

#### 创建 amdin-user

-  创建文件 dashboard-adminuser.yaml 输入以下内容 保存。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

- 创建 service account

```shell
[root@xxx ]# kubectl apply -f  dashboard-adminuser.yaml
```

- 创建文件 dashboard-cluster-role-bind.yaml 写入以下内容：

``` yaml
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
```

- 创建 ClusterRoleBinding

```shell
[root@xxx ]# kubectl apply -f  dashboard-cluster-role-bind.yaml
```

#### 使用 admin-user 登录

- 获取 admin token

```shell
[root@xxx ]# kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep "admin-user" | awk '{print $1}')
Name:         admin-user-token-mwjf4
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: af449090-3b7e-4baa-b80e-474bca319c38

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InJpcWFjMEI4S2ZmVjdZYTRzLS1vT1k0cGg0aGwzbWNPUVhBMlpLSjNhNjQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2Fy0NzRiY2EzMTljMzgiLCJzdWIiOiJzeX2VydmljZWFjY291bnQvljMzgiLCJzdWIiOiJzeXN0OiJhZG1pbi11c2VyLXRva2VuLW13amY0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJhZjQ0OTA5MC0zYjdlLTRiYWEtYjgwZS00NzRiY2EzMTljMzgiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.OLwnzOAgW2ozvpp9K9tdM48Xl0XINLYRr22urLakfLdFZjCbEgx6VlOKnjkenJr7ahu-EnuKgAS4m8k7NgCsUErj88JW0YZPA6Av5kYYNfP9ZZYX8KqY6FIuIIWfxNG12dfd-n0gevaYg7nGuh-_XokblfFHtLkrJajitSFYpatkFdXINLYRr22urLakfLdFZjCbEgx6VlOeN9KOjjku_IUrZyB15ArHNALb9teHMkJMBJneObKUNSSZKc7Dd4reUCaJb3M8MpbPRY8NQk5UpTmtjzuZHXs9aX7UVPUFGGNr-xQL_nj1cihxVDsg1BxZ2sbfKXWNT_IUshvq0AXySFWD1-g
```

使用这个admin 用户的token登录即可。



## 问题

出现 `Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")` 需要删除 ，解决方法如下：

```shell
rm -rf $HOME/.kube

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- 