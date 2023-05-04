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
GOOGLE_CLOUD_PROJECT=gcpsa-sandbox
REGION=asia-east1
CLUSTER=zabbix-cluster

gcloud container clusters create-auto $CLUSTER --region $REGION --project $GOOGLE_CLOUD_PROJECT

gcloud container clusters get-credentials $CLUSTER --region $REGION --project $GOOGLE_CLOUD_PROJECT

```

## 2. 建立 Database - PostgreSQL

```bash
tree database

database
├── postgres-deployment.yaml
├── postgres-pvc.yaml
└── postgres-service.yaml
```

部署容器 [PostgreSQL](https://hub.docker.com/_/postgres/) 

- postgres-pvc.yaml: 定義儲存空間 (PersistentVolumeClaim)，再動態產生 PV，由於儲存類別 `standard-rwo` 的綁定模式為 `WaitForFirstConsumer`，所以在任何 Pod 使用此 PV 前，不會真的建立 PV，將會處於 `Pending` 狀態。
- postgres-deployment.yaml: 部署資料庫 PostgreSQL 14，已經有環境變數 - `POSTGRES_USER`, `POSTGRES_PASSWORD` 與 `PGDATA`
- postgres-service.yaml: 暴露資料庫端口讓其他服務連線 (Port: 5432)

### 部署服務
部署 **database** 資料夾裡所有檔案：
```bash
kubectl apply -f database
```

## 3. 監控系統 - Zabbix 

```bash
tree zabbix-server 
zabbix-server
├── zabbix-deployment.yaml
├── zabbix-pv.yaml
├── zabbix-pvc.yaml
└── zabbix-service.yaml
```

### 自行管理硬碟

預設上會透過 PVC 動態建立硬碟，硬碟功能由 GCE 提供，刪除 PVC 也會同時刪除硬碟，
如果要保留硬碟資訊需要預先建立硬碟，再透過 PV 綁定。

接下來將使用 [StatefulSet 方式讀取預先存在硬碟](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/preexisting-pd#pv_to_statefulset)

#### 建立硬碟
```bash
DISK_NAME=pvc-snmp2

gcloud compute disks create $DISK_NAME --project=$GOOGLE_CLOUD_PROJECT \
  --type=pd-balanced --size=30GB --region=$REGION \
  --replica-zones=projects/$GOOGLE_CLOUD_PROJECT/zones/$REGION-c,projects/gcpsa-sandbox/zones/$REGION-b
```

#### 使用 [yq](https://github.com/mikefarah/yq) 替換 pv 模板

```bash
yq -i '.spec.csi.volumeHandle="projects/'$GOOGLE_CLOUD_PROJECT'/regions/'$REGION'/disks/'$DISK_NAME'"' zabbix-server/zabbix-pv.yaml

```

### 部署服務
部署 **zabbix-server** 資料夾裡所有檔案：
```bash
kubectl apply -f zabbix-server
```

## 4. Zabbix 管理介面

```bash
tree frontend 
frontend
├── frontend-deployment.yaml
├── frontend-service.yaml
└── ingress.yaml
```

### 部署前端服務 

部署 **frontend** 資料夾裡所有檔案：

```bash
kubectl apply -f frontend
```

### 取得管理介面 IP
```bash
kubectl get ingress zabbix-frontend-ingress
```

在瀏覽器輸入該 IP 並填入預設值
- Username : Admin
- Password : zabbix

## 清理全部資源 (Deployment,Service and Ingress)
```bash
kubectl delete -f frontend && \
kubectl delete -f zabbix-server && \
kubectl delete -f database
```