# 第一階段 - 在 GCP 建立 Zabbix service

## 目標
* 在 GCP 建立簡單的 Zabbix 監控服務
* 快速熟悉 Docker 指令

## Components

### Infra
* Google Compute Engine (GCE)
* HTTP(s) Cloud Load Balancing (GCLB)
* TCP Load Balancing (TCPLB)

### Container
* Zabbix
* PostgreSQL 

---

## 準備 PostgreSQL

### 建立 VM
```
gcloud compute instances create postgresql --zone=asia-east1-c \  
   --machine-type=e2-medium --network-interface=private-network-ip=10.140.0.100 \
   --image-family=ubuntu-1804-lts --image-project=ubuntu-os-cloud   
```

### 安裝 PostgreSQL

1. SSH 進入機器後，安裝 PostgreSQL
```
sudo apt update
sudo apt install postgresql postgresql-contrib

sudo systemctl start postgresql.service

```
2. 切換為 postgres 身份
```
sudo -i -u postgres
```
1. 進入資料庫
```
psql
```
4. 新增一個名稱為 *zabbix*，密碼為 *zabbixpw* 的帳號
```
CREATE USER "zabbix" WITH PASSWORD 'zabbixpw';
```
5. 建立 **zabbixdb** 資料庫並宣告 **zabbix** 為資料庫擁有者
```
CREATE DATABASE zabbixdb ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER "zabbix";
```
6. 離開資料庫並離開 postgres 身份
```
postgres=# \q
postgres@postgresql:~$ exit
logout
```
## 建立 Zabbix Server
```
gcloud compute instances create zabbix --zone=asia-east1-c \
   --machine-type=e2-medium --network-interface=private-network-ip=10.140.0.101 \
   --image-family=cos-97-lts --image-project=cos-cloud
```

SSH 進入機器後，安裝 [Zabbix server](https://hub.docker.com/r/zabbix/zabbix-server-pgsql)

```bash
docker run --name some-zabbix-server-pgsql \
    --network=host \ 
    -e DB_SERVER_HOST="10.140.0.100" \
    -e POSTGRES_USER="zabbix" \
    -e POSTGRES_PASSWORD="zabbixpw" \
    -e POSTGRES_DB="zabbixdb" \
    -e TZ=Asia/Taipei \
    -p 10051:10051 \
    --restart unless-stopped \
    -d zabbix/zabbix-server-pgsql:latest
```

安裝管理介面
```bash
docker run --name zabbix-web-nginx-pgsql -t \
    -e ZBX_SERVER_HOST="localhost" \
    -e DB_SERVER_HOST="10.140.0.100" \
    -e POSTGRES_USER="postgres" \
    -e POSTGRES_PASSWORD="zabbixpw" \
    -e POSTGRES_DB="zabbixdb" \
    -e PHP_TZ="Asia/Taipei" \
    -p 443:8443 \
    -p 80:8080 \
    --restart unless-stopped \
    -d zabbix/zabbix-web-nginx-pgsql:alpine-6.0-latest
```