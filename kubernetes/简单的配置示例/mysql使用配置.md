MySQL yaml 配置



## 配置 service 给内部使用

```yml
kind: Service
apiVersion: v1
metadata:
  name: mysql-service
  namespace: ville-app
  labes:
    name: mysql-service
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
```





## 配置 endPoint 给 service 转发流量

```yaml

kind: Endpoints
apiVersion: v1
metadata: 
  name: mysql-service
  namespace: ville-app
subsets:
  - address:
    - ip: 10.135.117.219
    ports:
      - port: 3306
```

