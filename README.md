# RuoYi (Vue) Deployment in k8s
Ruoyi is a Java EE enterprise-level rapid deployment platform built on classic technology combinations (backend - Spring Boot, frontend - Vue), with built-in modules including department management, role users, menu and button authorization, data permissions, system parameters, log management, code generation, etc. Online timing tasks configuration, and support clusters, support multiple data sources, support distributed transactions. 

## Source code & project
- Document download: <https://doc.ruoyi.vip/ruoyi-vue>
- Source code download: <https://gitee.com/y_project/RuoYi-Vue>
- Deployment guideline: <https://doc.ruoyi.vip/ruoyi-vue/document/hjbs.html>
Note that the documentation is only available in Chinese

## Preparation
```
JDK >= 1.8 (recommend version 1.8)
Mysql >= 5.7.0 (recommend version 5.7)
Redis >= 3.0
Maven >= 3.0
Node >= 12
```

## Deployment steps
- Deploy Redis
- Deploy MySQL
- Build backend image
- Set up private repository
- Deploy backend
- Build frontend image
- Deploy frontend

## Redis
Ruoyi uses Redis as a cache, a single node installation might satisfy the needs, and data persistence is unnecessary. We choose Bitnami package for redis. ([Redis Chart](https://artifacthub.io/packages/helm/bitnami/redis))
Add repository
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
Install chart
```
helm install redis \
             --set architecture=standalone \
             --set-string auth.password=123456 \
             --set master.persistence.enabled=false \
             --set master.persistence.medium=Memory \
             --set master.persistence.sizeLimit=1Gi \
             bitnami/redis \
             --kubeconfig=/etc/rancher/k8s/k8s.yaml

```
After installation, copy the redis's DNS for applications connecting to redis 
```
redis-master.default.svc.cluster.local
```
View redis
```
kubectl get pod
```
## MySQL
We need to set up a database `ry-vue`, and import initial data. ([MySQL Chart](https://artifacthub.io/packages/helm/bitnami/mysql))

Upload database initical scripts (`quartz.sql` and `ry_20220822.sql`) to `/home/app/sql`

Create configMap `kubectl create cm ruoyi-init-sql --from-file=/home/app/sql`

Install database `db` by configFile `ruoyi-mysql.yaml`
```
helm install db -f ruoyi-mysql.yaml \
                   bitnami/mysql \
                   --kubeconfig=/etc/rancher/k8s/k8s.yaml
```
After installation, copy the DNS addresses for applications connecting to mysql
```
db-mysql-primary.default.svc.cluster.local:3306
db-mysql-secondary.default.svc.cluster.local:3306
```
Test MySQL connection
```
kubectl get pod
kubectl port-forward pods/db-mysql-primary-0 --address=localhost:3306
```
## Backend image
Fix backend source code
- `ruoyi-admin/src/resources/application.yml`, fix profile to `/home/ruoyi/uploadPath`
- `ruoyi-admin/src/resources/application.yml`, fix redis's password to `123456`
- `ruoyi-admin/src/resources/application-druid.yml`, fix mysql's password to `123456`

Under project's /root, create project's Dockerfile and build image
```
docker build -t  ruoyi-admin:v3.8 .
docker images
```

## Frontend image
Frontend code is in the /ruoyi-ui, we compile frontend code in the container.
```
cd /app/ruoyi-ui
npm install 
npm install --registry=https://registry.npmmirror.com
npm run build:prod
```
Create Dockerfile under /ruoyi-ui/src/, and build image
```
docker build -t ruoyi-ui:v3.8 .
docker images
```

## Private registry
We use registry to set up private registry, and tag our private registry address as the image's address
```
docker run -d -p 5000:5000 --restart always --name registry registry:2
docker tag ruoyi-ui:v3.8 10.150.36.72:5000/ruoyi-ui:v3.8
```
Then, push the image to private registry
```
docker push 10.150.36.72:5000/ruoyi-ui:v3.8
```

## Deploy backend application
To introduce addresses in k8s, we create `applicaiton-k8s.yaml`, replace the redis host address and MySQL host address.
```
kubectl create cm ruoyi-admin-config --from-file=/home/app/application-k8s.yaml
```
Then, we mount configMap to the container by volume, see `svc-ruoyi-admin.yaml`

Create the backend deployment and service
```
kubectl apply -f svc-ruoyi-admin.yaml
kubectl get pod
```
