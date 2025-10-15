# Production MongoDB on Kubernetes with Percona Operator

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.24+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![MongoDB](https://img.shields.io/badge/MongoDB-7.0-47A248?logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![Percona](https://img.shields.io/badge/Percona-Operator-FF6600)](https://www.percona.com/)

> Deploy a production-ready MongoDB cluster on Kubernetes with automated backups, Prometheus monitoring, Discord alerts, and TLS security.

## 📋 Overview

This repository contains configuration files for deploying MongoDB on Kubernetes using the **Percona Operator**. It includes:

- ✅ 3-node MongoDB replica set (High Availability)
- ✅ Automated backups to AWS S3
- ✅ Prometheus + Grafana monitoring
- ✅ Discord webhook alerts
- ✅ TLS encryption support
- ✅ Auto-scaling persistent storage

## 🚀 Quick Start

### Prerequisites

- Kubernetes cluster (v1.24+)
- `kubectl` configured
- Helm 3 installed
- AWS account (for S3 backups)
- Discord webhook URL (for alerts)

### 1. Create Namespaces

```bash
kubectl create namespace mongodb
kubectl create namespace monitoring
```

### 2. Setup Storage Class

```bash
kubectl apply -f storageClass.yaml
```

### 3. Create Secrets

⚠️ **Important**: Never commit secrets to Git!

#### MongoDB Users Secret

```bash

kubectl apply -f users.yaml
```

#### AWS Backup Credentials

```bash
# Create AWS IAM user with S3 permissions first
# Then encode credentials:
echo -n "YOUR_AWS_ACCESS_KEY_ID" | base64
echo -n "YOUR_AWS_SECRET_ACCESS_KEY" | base64

# Edit aws_backup_iam_user.yaml with your credentials
kubectl apply -f aws_backup_iam_user.yaml
```

### 4. Install Percona Operator

```bash
# Add Percona Helm repository
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

# Install the operator
helm install percona-operator percona/psmdb-operator \
  --namespace mongodb
```

### 5. Deploy MongoDB Cluster

```bash
# Update values.yaml with your configuration:
# - Change bucket name in backup section
# - Update nameOverride if needed
# - Adjust storage size as required

helm install my-db percona/psmdb-db \
  --namespace mongodb \
  --values values.yaml
```

### 6. Verify Deployment

```bash
# Watch pods starting
kubectl get pods -n mongodb -w

# Expected output (2/2 = MongoDB + metrics exporter):
# NAME        READY   STATUS    RESTARTS   AGE
# my-db-rs0-0   2/2     Running   0          5m
# my-db-rs0-1   2/2     Running   0          4m
# my-db-rs0-2   2/2     Running   0          3m
```

## 📊 Monitoring Setup

### Install Prometheus Stack

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --values monitoring/values.yaml
```

### Apply Metrics Service

```bash
kubectl apply -f metric-service.yaml
```

### Setup Discord Alerts

```bash
# Create Discord webhook secret
kubectl create secret generic discord-webhook-url \
  --from-literal=url='<your-discord-webhook>/slack' \
  -n monitoring

# Apply alert rules
kubectl apply -f monitoring/alert-rules.yaml

# Apply alert manager configuration
kubectl apply -f monitoring/alert-manager-configuration.yaml
```

### Access Grafana

```bash
# Get Grafana admin password
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode

# Port forward to Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# Open http://localhost:3000
# Username: admin
# Password: <from above command>
```

### Import MongoDB Dashboard

1. Go to Grafana UI
2. Click **+** → **Import**
3. click **Upload dashboard JSON file** and select `monitoring/mongodb-dashboard.json`

## 💾 Backup & Restore

### Verify Automated Backups

```bash
# Check backup schedule (from values.yaml):
# - Daily: 0 0 * * * (midnight UTC)
# - Weekly: 0 1 * * 0 (Sunday 1 AM UTC)

# View backup status
kubectl get psmdb-backup -n mongodb

# Check backup logs
kubectl logs -l app.kubernetes.io/name=percona-server-mongodb-backup -n mongodb
```

### Manual Backup

```bash
# Trigger manual backup
kubectl apply -f - <<EOF
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBBackup
metadata:
  name: manual-backup-$(date +%Y%m%d-%H%M%S)
  namespace: mongodb
spec:
  clusterName: my-db
  storageName: s3-north-1
EOF

# Monitor backup progress
kubectl get psmdb-backup manual-backup-test -n mongodb -w
```

### Restore from Backup

```bash
# Edit mongo-restore.yaml with backup name
# Restore to test cluster
kubectl apply -f - <<EOF
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBRestore
metadata:
  name: restore-test-$(date +%Y%m%d)
  namespace: mongodb
spec:
  clusterName: <db-name>
  backupName: manual-backup-$(date +%Y%m%d-%H%M%S)
EOF

# Monitor restore
kubectl get psmdb-restore -n mongodb -w
```


# full tutorial: 
https://medium.com/@abdelrahmanelshahat00/how-to-deploy-production-mongodb-on-kubernetes-with-percona-operator-backups-monitoring-and-c4af1ebd21a4