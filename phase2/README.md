# Phase 2 - Replace Storage Layer to Google Managed Service

## Architecture
![image](https://github.com/microfusion-cloud/zabbix-on-gke/blob/main/phase2/architecture.png)


## Objective
* Replace PostgreSQL on GCE to Cloud SQL for PostgreSQL  

## Components

### Infra
* Google Compute Engine (GCE)
* HTTP(s) Cloud Load Balancing (GCLB)
* TCP Load Balancing (TCPLB)
* **(New)** Cloud SQL for PostgreSQL

### Container
* Zabbix container
