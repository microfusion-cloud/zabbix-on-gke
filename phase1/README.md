# 第一階段 - 在 GCP 建立 Zabbix service

## Architecture
![image](https://github.com/microfusion-cloud/zabbix-on-gke/blob/main/assets/phase1.png)

## 目標
* 在 GCP 建立簡單的 Zabbix 監控服務
* 快速熟悉 Docker 指令

## Components

### Infra
* Google Kubernetes Engine (GKE)
### Container
* Zabbix
* PostgreSQL 


---

## 0. 前置步驟
- 打開 Cloud Shell
- 下載此範例程式並進入 phase1

```bash
git clone https://github.com/microfusion-cloud/zabbix-on-gke.git && cd zabbix-on-gke/phase1
```


## 1.建立 GKE 叢集

```bash
gcloud container clusters create-auto "zabbix-cluster" \
   --region "asia-east1"

gcloud container clusters get-credentials zabbix-cluster --region asia-east1 --project [PROJECT_ID]

```

## 2.部署 PostgreSQL

部署容器 [PostgreSQL](https://hub.docker.com/_/postgres/) 

- postgres-pv.yaml: 定義儲存空間 (PersistentVolume)，由於儲存類別 `standard-rwo` 的綁定模式為 `WaitForFirstConsumer`，所以在任何 Pod 使用此 PV 前，不會真的建立 PV，將會處於 `Pending` 狀態。
- postgres-deployment.yaml: 部署資料庫 PostgreSQL 14，已經有環境變數 - `POSTGRES_USER`, `POSTGRES_PASSWORD` 與 `PGDATA`
- postgres-service.yaml: 暴露資料庫端口讓其他服務連線 (Port: 5432)

```bash
kubectl apply -f postgres-pv.yaml && \
kubectl apply -f postgres-deployment.yaml && \
kubectl apply -f postgres-service.yaml
```

## 3.部署暴露 Zabbix 

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
## 4.部署 Zabbix 管理介面

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

## TODO: 補上 Agent 填入 Health check ##

## 清理全部資源 (Deployment,Service and Ingress)
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