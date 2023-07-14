# Phase 3 - 自建 GCLB 取代 GKE Ingress 

## Architecture
![image](https://github.com/microfusion-cloud/zabbix-on-gke/blob/main/assets/phase3.png)

## Objective
* 不使用 GKE Ingress 暴露 L7 服務
* 改為建立 GCLB 並透過 Standalone NEG 連接到 K8s Service

更多資訊可以參考: https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg

## Step
### 0. 前置步驟
- 打開 Cloud Shell
- 下載此範例程式並進入 phase3

```bash
git clone https://github.com/microfusion-cloud/zabbix-on-gke.git && cd zabbix-on-gke/phase3
```
### 1.建立 GKE 叢集

注意這邊多了 network-tags, 以便後續設定防火牆

```bash
gcloud container clusters create-auto "zabbix-cluster" \
    --region "asia-east1" \
    --autoprovisioning-network-tags=gke-zabbix-node \
    --project $GOOGLE_CLOUD_PROJECT


gcloud container clusters get-credentials zabbix-cluster \
    --region asia-east1 \
    --project $GOOGLE_CLOUD_PROJECT

Fetching cluster endpoint and auth data.
kubeconfig entry generated for zabbix-cluster.

```

後續部署請參考 Phase 2. Step 2-4，請注意不執行 step 5.
ref: https://github.com/microfusion-cloud/zabbix-on-gke/tree/main/phase2#2%E5%BB%BA%E7%AB%8B-cloud-sql-for-postgresql


## 部署 Google Cloud Load Balancer

將建立以下資源：

![image](https://cloud.google.com/static/kubernetes-engine/images/sneg7.svg)

增加防火牆規則：

```bash
gcloud compute firewall-rules create fw-allow-health-check-and-proxy \
    --network=default \
    --action=allow \
    --direction=ingress \
    --target-tags=gke-zabbix-node \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --rules=tcp:8080 \
    --project $GOOGLE_CLOUD_PROJECT
```

建立 global IP
```bash
gcloud compute addresses create hostname-server-vip \
    --ip-version=IPV4 \
    --global \
    --project $GOOGLE_CLOUD_PROJECT

GCLB_IP=$(gcloud compute addresses describe hostname-server-vip --global --format="get(address)")
```
建立健康檢查與後端服務

```bash
gcloud compute health-checks create http http-basic-check \
    --use-serving-port \
    --project $GOOGLE_CLOUD_PROJECT

gcloud compute backend-services create my-bes \
    --protocol HTTP \
    --health-checks http-basic-check \
    --global \
    --project $GOOGLE_CLOUD_PROJECT

gcloud compute url-maps create web-map \
    --default-service my-bes \
    --project $GOOGLE_CLOUD_PROJECT

gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map \
    --project $GOOGLE_CLOUD_PROJECT

gcloud compute forwarding-rules create http-forwarding-rule \
    --address=$GCLB_IP \
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80 \
    --project $GOOGLE_CLOUD_PROJECT
```

將後端 NEG 服務 加入 LB
```bash
gcloud compute backend-services add-backend my-bes \
    --global \
    --network-endpoint-group=zabbix-frontend-neg \
    --network-endpoint-group-zone=asia-east1-b \
    --balancing-mode RATE --max-rate-per-endpoint 5 \
    --project $GOOGLE_CLOUD_PROJECT

gcloud compute backend-services add-backend my-bes \
    --global \
    --network-endpoint-group=zabbix-frontend-neg \
    --network-endpoint-group-zone=asia-east1-c \
    --balancing-mode RATE --max-rate-per-endpoint 5 \
    --project $GOOGLE_CLOUD_PROJECT
```

測試: 

```bash
echo $GCLB_IP
```

在瀏覽器輸入該 IP 並填入預設值
- Username : Admin
- Password : zabbix

## 清理全部資源 (Deployment,Service and Ingress)
```bash
kubectl delete -f frontend && \
kubectl delete -f zabbix-server && \
gcloud sql instances delete $DB_INSTANCE --project=$GOOGLE_CLOUD_PROJECT

(TODO: Delete Backend Service)

```