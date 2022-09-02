# 第一階段 - 在 GCP 建立 Zabbix service

## Architecture
![image](https://github.com/microfusion-cloud/zabbix-on-gke/blob/main/phase1/architecture.png)

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

## 1.建立 GKE 叢集

## 2.部署 PostgreSQL

部署容器 [PosttgreSQL](https://hub.docker.com/_/postgres/) 
相關設置(TODO)

## 3.部署暴露 Zabbix 

## Step-by-step
1. Create server deployment
```
kubectl apply -f zabbix-server-deployment.yaml
```
2. Create server service
```
kubectl apply -f zabbix-server-service.yaml
```
3. Create web deployment
```
kubectl apply -f zabbix-frontend-deployment.yaml
```
4. Create web service

```
kubectl apply -f zabbix-frontend-service.yaml
```

5. Create web ingress

```
kubectl apply -f ingress.yaml
```