# Phase 2 - 使用代管式資料庫 CloudSQL 替代 PostgreSQL / 保護 Secret 

## Architecture
![image](https://github.com/microfusion-cloud/zabbix-on-gke/blob/main/phase2/architecture.png)


## Objective
* Replace PostgreSQL on GCE to Cloud SQL for PostgreSQL
* 將需要隱藏的機敏資訊加密

## Components

### Infra
* Google Kubernetes Engine (GKE)
* HTTP(s) Cloud Load Balancing (GCLB)
* TCP Load Balancing (TCPLB)
* **(New)** Cloud SQL for PostgreSQL
* **(New)** Cloud Key Management Services

### Container
* Zabbix container

---

## Step
### 0. 前置步驟
- 打開 Cloud Shell
- 下載此範例程式並進入 phase2

```bash
git clone git@github.com:microfusion-cloud/zabbix-on-gke.git && cd zabbix-on-gke/phase2
```
### 1.建立 GKE 叢集

```bash
gcloud container clusters create-auto "zabbix-cluster" \
   --region "asia-east1"

gcloud container clusters get-credentials zabbix-cluster --region asia-east1 --project [PROJECT_ID]

```

### 2.部署 Cloud SQL for PostgreSQL

TODO: 建立 Cloud SQL / 建立 user/ password


### 3.部署暴露 Zabbix 

TODO: 密碼修改成使用 Secret 

取得 postgreSQL IP
```bash
kubectl get svc postgres -o jsonpath="{.spec.clusterIP}"
> 10.91.128.94
```
把此 IP 覆蓋 `zabbix-server-deployment.yaml` 的 `{DB_SERVER_HOST}`

Before:
```base
...
        env:
         - name: DB_SERVER_HOST
           value: ${DB_SERVER_HOST} # 覆蓋
...

```

After:
```base
...
        env:
         - name: DB_SERVER_HOST
           value: 10.91.128.94
...

```
部署 Zabbix Server
```bash
kubectl apply -f zabbix-server-deployment.yaml
```

暴露接收監控紀錄端口
```
kubectl apply -f zabbix-server-service.yaml
```
### 4.部署 Zabbix 管理介面

#### 把先前取得的 PostgreSQL IP 覆蓋 `zabbix-frontend-deployment.yaml` 的 `{DB_SERVER_HOST}`

部署前端服務 
```bash
kubectl apply -f zabbix-frontend-deployment.yaml && \
kubectl apply -f zabbix-frontend-service.yaml && \
kubectl apply -f ingress.yaml
```

取得管理介面 IP
```
kubectl get ingress zabbix-frontend-ingress
```

在瀏覽器輸入該 IP 並填入預設值
- Username : Admin
- Password : zabbix


### 清理全部資源 (Deployment,Service and Ingress)
```bash
kubectl delete -f postgres-deployment.yaml && \
kubectl delete -f postgres-pv.yaml && \
kubectl delete -f postgres-service.yaml && \
kubectl delete -f zabbix-server-deployment.yaml && \
kubectl delete -f zabbix-server-service.yaml && \
kubectl delete -f zabbix-frontend-deployment.yaml && \
kubectl delete -f zabbix-frontend-service.yaml && \
kubectl delete -f ingress.yaml
```