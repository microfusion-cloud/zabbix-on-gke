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
   --region "asia-east1" \
   --project $GOOGLE_CLOUD_PROJECT

gcloud container clusters get-credentials zabbix-cluster \
  --region asia-east1 \
  --project $GOOGLE_CLOUD_PROJECT

Fetching cluster endpoint and auth data.
kubeconfig entry generated for zabbix-cluster.

```

### 2.建立 Cloud SQL for PostgreSQL
#### Instance specs
- DB Version : PostgreSQL 14
- Instance Name: zabbix-instance
- Machine Type : db-g1-small (1 vCore , 1.7 GB Ram)
- Storage : 10GB , HDD
- 啟動自動增加硬碟上限
- 單區域可用，無 Highly available


#### 設定 Cloud SQL 私人連線
ref: https://cloud.google.com/sql/docs/mysql/configure-private-services-access#configure-access

```bash
gcloud compute addresses create google-managed-services-default \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=16 \
  --network=projects/$GOOGLE_CLOUD_PROJECT/global/networks/default
  
gcloud services vpc-peerings connect \
--service=servicenetworking.googleapis.com \
--ranges=google-managed-services-default \
--network=default \
--project=$GOOGLE_CLOUD_PROJECT
```

#### 使用 gcloud-cli 建立 instance

```bash
gcloud sql instances create zabbix-instance \
  --database-version=POSTGRES_14 \
  --tier=db-g1-small \
  --storage-type=HDD \
  --storage-size=10 \
  --storage-auto-increase \
  --zone=asia-east1-a \
  --root-password=password123 \
  --no-assign-ip \
  --network=projects/$GOOGLE_CLOUD_PROJECT/global/networks/default
```

#### 使用 gcloud-cli 建立 Database
```bash
gcloud sql databases create zabbix \
  --instance=zabbix-instance
```

#### 使用 gcloud-cli 建立 User/PW
```bash
#自定義密碼
DB_PASS=YOUR_PW

gcloud sql users create zabbix \
   --instance=zabbix-instance \
   --password=${DB_PASS}
```
#### 建立 Database secret

```bash
kubectl create secret generic zabbix-secret \
  --from-literal=username=zabbix \
  --from-literal=password=${DB_PASS} \
  --from-literal=database=zabbix
```

### 設定 Workload Identity

Workload Identity 讓 Google Service Account (GSA) 與 Kubernetes Service Account (KSA) 綁定在一起，
KSA 將會模擬 GSA 所擁有的 IAM 權限與 Google Cloud 溝通，好處是可以避免產生 GSA Key 就可以使用 Google Cloud 服務。

ref: https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#authenticating_to

![image](https://github.com/microfusion-cloud/zabbix-on-gke/blob/main/assets/workload_identity.png)


建立 KSA
```bash
kubectl create serviceaccount ksa-cloudsqlproxy
```

建立 GSA 並給予 Cloud SQL Client 角色
```bash
gcloud iam service-accounts create cloudsqlproxy \
    --project=$GOOGLE_CLOUD_PROJECT

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member serviceAccount:cloudsqlproxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
    --role "roles/cloudsql.client"
```

建立 GSA 與 KSA 綁定
```bash
gcloud iam service-accounts add-iam-policy-binding cloudsqlproxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member serviceAccount:$GOOGLE_CLOUD_PROJECT.svc.id.goog[default/ksa-cloudsqlproxy]
```

建立註解
```bash
kubectl annotate serviceaccount ksa-cloudsqlproxy \
    iam.gke.io/gcp-service-account=cloudsqlproxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```


### 3.部署暴露 Zabbix server

#### 取得 postgreSQL connection name 
規則如下：
```bash
{Project_ID}:{Region}:{Instance_Name}

example:
calm-photon-320710:asia-east1:zabbix-instance
```

#### 修改 `zabbix-server-deployment.yaml`

把 connection name 覆蓋 `zabbix-server-deployment.yaml` 的 `<INSTANCE_CONNECTION_NAME>`

Before:
```
# line 58:
- "-instances=<INSTANCE_CONNECTION_NAME>=tcp:5432"
```
After:
```base
- "-instances=calm-photon-320710:asia-east1:zabbix-instance>=tcp:5432"
```
#### Deployment 使用 KSA
可以觀察到 `zabbix-server-deployment.yaml` line 15:

```bash
serviceAccountName: ksa-cloudsqlproxy
```
可以指定 Deployment 使用服務帳戶運行


#### 部署 Zabbix Server
```bash
kubectl apply -f zabbix-server-deployment.yaml
```

#### 暴露接收監控紀錄端口
```
kubectl apply -f zabbix-server-service.yaml
```
### 4.部署 Zabbix 管理介面

#### 修改 `zabbix-frontend-deployment.yaml`

把先前取得的 connection name 覆蓋 `zabbix-frontend-deployment.yaml` 的 `<INSTANCE_CONNECTION_NAME>`

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
kubectl delete -f postgres-service.yaml && \
kubectl delete -f zabbix-server-deployment.yaml && \
kubectl delete -f zabbix-server-service.yaml && \
kubectl delete -f zabbix-frontend-deployment.yaml && \
kubectl delete -f zabbix-frontend-service.yaml && \
kubectl delete -f ingress.yaml
```