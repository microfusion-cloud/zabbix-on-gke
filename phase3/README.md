# Phase 3 - Run Zabbix container on GKE

## Objective
* Replace Compute Engine to Container Platform - Google Kubernetes Engine
* Using Standalone NEG for sepreated 

## Components

### Infra
* HTTP(s) Cloud Load Balancing (GCLB)
* Cloud SQL for PostgreSQL
* **(New)** Google Kubernetes Engine (GKE)
  * TCP Load Balancing (TCPLB) as **Service**

### Container
* Zabbix container
