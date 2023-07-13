# Phase 2 - 使用代管式資料庫 CloudSQL 替代 PostgreSQL / 保護 Secret 

## Architecture
![image](https://github.com/microfusion-cloud/zabbix-on-gke/blob/main/assets/phase2.png)


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
git clone https://github.com/microfusion-cloud/zabbix-on-gke.git && cd zabbix-on-gke/phase2
```
### 1.建立 GKE 叢集

```bash
GOOGLE_CLOUD_PROJECT=gcpsa-sandbox
REGION=asia-east1
CLUSTER=zabbix-cluster

gcloud container clusters create-auto $CLUSTER --region $REGION --project $GOOGLE_CLOUD_PROJECT

gcloud container clusters get-credentials $CLUSTER --region $REGION --project $GOOGLE_CLOUD_PROJECT

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
  --network=projects/$GOOGLE_CLOUD_PROJECT/global/networks/default \
  --project=$GOOGLE_CLOUD_PROJECT
  
gcloud services vpc-peerings connect \
--service=servicenetworking.googleapis.com \
--ranges=google-managed-services-default \
--network=default \
--project=$GOOGLE_CLOUD_PROJECT
```

#### 使用 gcloud-cli 建立 Cloud SQL instance 

```bash
DB_INSTANCE=zabbix-instance

gcloud sql instances create $DB_INSTANCE \
  --database-version=POSTGRES_15 \
  --tier=db-g1-small \
  --storage-type=HDD \
  --storage-size=10 \
  --storage-auto-increase \
  --zone=asia-east1-a \
  --root-password=password123 \
  --no-assign-ip \
  --network=projects/$GOOGLE_CLOUD_PROJECT/global/networks/default \
  --project=$GOOGLE_CLOUD_PROJECT
```

#### 使用 gcloud-cli 建立 Database
```bash
DB_NAME=zabbix

gcloud sql databases create $DB_NAME \
  --instance=$DB_INSTANCE \
  --project=$GOOGLE_CLOUD_PROJECT
```

#### 使用 gcloud-cli 建立 User/PW
```bash
#自定義密碼
DB_PASS=YOUR_PW
DB_USER=zabbix

gcloud sql users create $DB_USER \
   --instance=$DB_INSTANCE \
   --password=${DB_PASS} \
   --project=$GOOGLE_CLOUD_PROJECT
```
#### 建立 Database secret

```bash
DB_SECRET=zabbix-secret

kubectl create secret generic $DB_SECRET \
  --from-literal=username=$DB_USER \
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


### 3. 監控系統 - Zabbix 

#### 建立硬碟
```bash
DISK_NAME=pvc-snmp

gcloud compute disks create $DISK_NAME --project=$GOOGLE_CLOUD_PROJECT \
  --type=pd-balanced --size=30GB --region=$REGION \
  --replica-zones=projects/$GOOGLE_CLOUD_PROJECT/zones/$REGION-c,projects/$GOOGLE_CLOUD_PROJECT/zones/$REGION-b
```

#### 使用 [yq](https://github.com/mikefarah/yq) 替換 pv 模板

```bash
yq -i '.spec.csi.volumeHandle="projects/'$GOOGLE_CLOUD_PROJECT'/regions/'$REGION'/disks/'$DISK_NAME'"' zabbix-server/zabbix-pv.yaml

```

#### 取得 postgreSQL connection name 
規則如下：
```bash
{Project_ID}:{Region}:{Instance_Name}

example:
gcpsa-sandbox:asia-east1:zabbix-instance
```

#### 修改 `zabbix-server/zabbix-deployment.yaml`

把 connection name 覆蓋 `zabbix-server-deployment.yaml` 的 `<INSTANCE_CONNECTION_NAME>`

Before:
```
# line 49:
- "<INSTANCE_CONNECTION_NAME>""
```
After:
```base
- "gcpsa-sandbox:asia-east1:zabbix-instance"
```
#### Deployment 使用 KSA
可以觀察到 `zabbix-server/zabbix-deployment.yaml` line 16:

```bash
serviceAccountName: ksa-cloudsqlproxy
```
可以指定 Deployment 使用服務帳戶運行

### 部署服務
部署 **zabbix-server** 資料夾裡所有檔案：
```bash
kubectl apply -f zabbix-server
```

## 4. Zabbix 管理介面

#### 修改 `frontend-deployment.yaml`

把先前取得的 *connection name* 覆蓋 `frontend/frontend-deployment.yaml` 的 `<INSTANCE_CONNECTION_NAME>`

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


## TODO: 補上 Agent 填入 Health check ##


## 清理全部資源 (Deployment,Service and Ingress)
```bash
kubectl delete -f frontend && \
kubectl delete -f zabbix-server && \
gcloud sql instances delete $DB_INSTANCE
```
