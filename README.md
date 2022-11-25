# Phase 3 - 使用 Standalone NEG 可以自行控制 Ingress 的行為 

ref: https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg

## Objective
* 不使用 Google Cloud 原生 Ingress (GCLB) 暴露服務
* 獨立建立 GCLB 並連接到 K8s Service


## Step
### 0. 前置步驟
- 打開 Cloud Shell
- 下載此範例程式並進入 phase3

```bash
git clone git@github.com:microfusion-cloud/zabbix-on-gke.git && cd zabbix-on-gke/phase3
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



(TODO:zabbix 部署)

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
   --rules=tcp:9376
```

建立 global IP
```bash
gcloud compute addresses create hostname-server-vip \
    --ip-version=IPV4 \
    --global

GCLB_IP=$(gcloud compute addresses describe hostname-server-vip --global --format="get(address)")
```

```
gcloud compute health-checks create http http-basic-check \
    --use-serving-port

gcloud compute backend-services create my-bes \
    --protocol HTTP \
    --health-checks http-basic-check \
    --global

gcloud compute url-maps create web-map \
    --default-service my-bes

gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map

gcloud compute forwarding-rules create http-forwarding-rule \
    --address=$GCLB_IP \
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80

```

將 LB 加入後端 NEG 服務
```
gcloud compute backend-services add-backend my-bes \
    --global \
    --network-endpoint-group=zabbix-frontend-neg \
    --network-endpoint-group-zone=asia-east1-b \
    --balancing-mode RATE --max-rate-per-endpoint 5

gcloud compute backend-services add-backend my-bes \
    --global \
    --network-endpoint-group=zabbix-frontend-neg \
    --network-endpoint-group-zone=asia-east1-c \
    --balancing-mode RATE --max-rate-per-endpoint 5

```