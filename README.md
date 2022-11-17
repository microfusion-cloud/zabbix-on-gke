# Phase 3 - 使用 Standalone NEG 可以自行控制 Ingress 的行為 

ref: https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg

## Objective
* 不使用 Google Cloud 原生 Ingress (GCLB) 暴露服務
* 獨立建立 GCLB 並連接到 K8s Service
