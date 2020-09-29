# mo-gateway 配置示例





#### deployment配置

```yml
# 新建一个工作站
apiVersion: v1
kind: Namespace
metadata:
  name: ville-app

---
# 启动一个对外服务的端口
kind: Service
apiVersion: v1
metadata:
  name: mo-gateway
  namespace: ville-app
  labels:
    name: mo-gateway
spec:
#端口类型需要指定 NodePort
  type: NodePort
  ports:
  # service 也是一个类似容器的东西，这个端口是自己的端口
  - port: 10080 
  	# 目标端口
    targetPort: 10080
    # 对外开放的端口
    nodePort: 30080
  selector:
  # service 选择器要选择的服务，通过label来匹配的
    name: mo-gateway


---
# 创建一个Deployment类型的应用
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mo-gateway
  namespace: ville-app
spec:
# 指定启动的副本个数
  replicas: 1
  selector:
    matchLabels:
    # lable k-v 配置，可用于selector查询应用
      name: mo-gateway
  # 指定启动应用的模板
  template:
    metadata:
      labels:
        name: mo-gateway
    spec:
      containers:
        - name: mo-gataway
          image: mo-gateway
          # 获取镜像的方式，Never 表示不拉取(本地docker image存在就配置这个值)，Allway 总是拉取
          imagePullPolicy: Never
          ports: 
          - containerPort: 10080
            protocol: TCP
```





#### 账户角色配置

```yaml
# 创建一个账户，同时会创建一个 secret token
# 可使用 kubectl get secret -n <namespace> 查看账户对应的 token
kind: ServiceAccount
apiVersion: v1
metadata:
  name: ville-sa
  namespace: ville-app

---
# 创建一个角色
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ville-reader
  namespace: ville-app
rules:
# 角色权限创建的规则
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods","pods/log","services"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["autoscaling", "apps"]
  resources: ["deployments","replicasets"]
  verbs: ["get", "list", "watch"]

---
# 使用账户绑定角色
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ville-sa-reader
  namespace: ville-app
subjects:
# 指定账户
- kind: ServiceAccount
  name: ville-sa
  namespace: ville-app
roleRef:
#自定角色
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: ville-reader
```

