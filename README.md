# Node.js-App-Monitoring-with-Prometheus-and-Grafana


## 🎯 Project Title: **Node.js App Monitoring on Kubernetes with Prometheus and Grafana**

---
> As part of our infrastructure monitoring initiative, we needed a scalable solution to visualize the health and performance of our microservices running in Kubernetes. This project demonstrates how we implemented a full observability stack using Prometheus for metric collection, Grafana for real-time dashboards, Alertmanager for proactive notifications, and supporting tools like Node Exporter and Kube-State-Metrics for deep system and cluster visibility. We containerized a Node.js application with custom metrics and deployed the entire stack on Minikube, integrating persistent storage and alerting to simulate a production-grade environment. This setup helped our DevOps team rapidly identify performance bottlenecks, track application trends, and ensure high system availability through timely alerts.
---

### ✅ Objective

Set up **Prometheus** to scrape metrics from a Node.js application and **Grafana** to visualize these metrics using a dashboard on a local **Minikube** Kubernetes cluster.

---

## 🛠 Tools & Concepts

| Tool               | Purpose                                                       |
| ------------------ | ------------------------------------------------------------- |
| Prometheus         | Collect and store time-series metrics                         |
| Node Exporter      | Expose system metrics to Prometheus (CPU, Memory, etc.)       |
| Grafana            | Create dashboards and visualizations using Prometheus metrics |
| Kube-State-Metrics | Exposes cluster state info like pods, nodes, deployments      |
| Minikube           | Run Kubernetes locally for development and testing            |
| Node.js App        | Sample app with Prometheus metrics exposed on `/metrics`      |

---

## 📁 Project Structure

```
monitoring-project/
├── app/                                           # Node.js app exposing Prometheus metrics
│   ├── index.js                                   # Express app serving '/' and '/metrics' using prom-client
│   ├── package.json                               # Defines dependencies like express and prom-client
│   └── Dockerfile                                 # Dockerfile to build the Node.js metrics app image
│
├── k8s/                                           # Kubernetes manifests for app + monitoring stack
│   ├── namespace.yaml                             # (Optional) Namespace to group all monitoring resources
│
│   ├── app-deployment.yaml                        # Deploys the Node.js app with 1 replica
│   ├── app-service.yaml                           # ClusterIP service exposing app on port 3000
│
│   ├── prometheus-deployment.yaml                 # Deploys Prometheus with PVC and config volume
│   ├── prometheus-service.yaml                    # NodePort service exposing Prometheus on port 9090
│
│   ├── grafana-deployment.yaml                    # Deploys Grafana with PVC for dashboard persistence
│   ├── grafana-service.yaml                       # NodePort service exposing Grafana on port 3000
│
│   ├── alertmanager-deployment.yaml               # Deploys Alertmanager with config volume
│   ├── alertmanager-service.yaml                  # NodePort service exposing Alertmanager on port 9093
│
│   ├── node-exporter-daemonset.yaml               # DaemonSet that runs node-exporter on each node for system metrics
│   ├── kube-state-metrics.yaml                    # Deployment + service to expose Kubernetes object metrics (pods, nodes, etc.)
│
│   ├── pvc-prometheus.yaml                        # PersistentVolumeClaim for Prometheus data storage
│   ├── pvc-grafana.yaml                           # PersistentVolumeClaim for Grafana data and dashboards
│
│   └── config/                                    # Configuration files for Prometheus and Alertmanager
│       ├── prometheus-config.yaml                 # Defines scrape targets: app, node-exporter, kube-state-metrics, alertmanager
│       └── alertmanager-config.yaml               # Alertmanager routes + email/SMS/Slack notification setup

```

---

## 🔧 Step-by-Step Breakdown

---

### **Step 1: Create a Node.js App with Prometheus Metrics**

```js
// app/index.js
const express = require('express');
const app = express();
const client = require('prom-client');

// Metrics
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics();

app.get('/', (req, res) => {
  res.send('Hello from Node.js!');
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.listen(3000, () => {
  console.log('App running on port 3000');
});
```

---

### **Step 2: Dockerize the App**

```Dockerfile
# Dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

Build & push the image:

```bash
# From inside the app/ directory
docker build -t laly9999/node-monitoring-app:1 .
docker push laly9999/node-monitoring-app:1
```

---

## 🧱 Step-by-Step Setup with Explanation

---

### 🔹 Step 1: Deploy the Node.js App

**Purpose**: This is the app we're monitoring.

```bash
kubectl apply -f k8s/app-deployment.yaml
kubectl apply -f k8s/app-service.yaml
```

* App exposes `/metrics` using `prom-client`
* Prometheus will scrape this endpoint to collect metrics like request counts, response times

🔍 **Prometheus Analysis**:
Visit Prometheus UI → **Targets** → Ensure `node-app-service:3000` is UP
Query: `http_requests_total` or `process_cpu_user_seconds_total`

---

### 🔹 Step 2: Configure Persistent Volumes (PVCs)

```bash
kubectl apply -f k8s/pvc-prometheus.yaml
kubectl apply -f k8s/pvc-grafana.yaml
```

* Ensures Prometheus and Grafana retain data across pod restarts

---

### 🔹 Step 3: Deploy Prometheus with Custom Config

```bash
kubectl create configmap prometheus-config --from-file=k8s/config/prometheus-config.yaml
kubectl apply -f k8s/prometheus-deployment.yaml
kubectl apply -f k8s/prometheus-service.yaml
```

* Scrapes metrics from:

  * Node.js app
  * Node Exporter
  * Kube-State-Metrics
  * Alertmanager
* Mounts volume at `/prometheus`

🔍 **Prometheus Analysis**:
Visit: `http://<MinikubeIP>:30090`
Try queries like:

* `up` → shows all healthy targets
* `rate(http_requests_total[1m])` → requests per second
* `process_resident_memory_bytes` → memory used by Node.js

---

### 🔹 Step 4: Deploy Grafana with PVC

```bash
kubectl apply -f k8s/grafana-deployment.yaml
kubectl apply -f k8s/grafana-service.yaml
```

Visit: `http://<MinikubeIP>:30300`

* Login: `admin/admin`
* Add data source: Prometheus URL → `http://prometheus-service:9090`

📊 **Grafana Analysis**:

1. Import Dashboard ID `11074` for Node.js
2. Build your own:

   * Panel: CPU Usage → Query: `process_cpu_seconds_total`
   * Pod restarts: `kube_pod_container_status_restarts_total`

---

### 🔹 Step 5: Add Kube-State-Metrics

```bash
kubectl apply -f k8s/kube-state-metrics.yaml
```

* Adds metrics like:

  * `kube_pod_info`
  * `kube_deployment_status_replicas_available`
  * `kube_node_status_capacity_cpu_cores`

🔍 Use these for dashboards on cluster health and pod status.

---

### 🔹 Step 6: Add Node Exporter (DaemonSet)

```bash
kubectl apply -f k8s/node-exporter-daemonset.yaml
```

* Runs on each node, collects host-level metrics
* Metrics include:

  * `node_cpu_seconds_total`
  * `node_memory_Active_bytes`
  * `node_filesystem_size_bytes`

📊 **Grafana Node Exporter Dashboard**: ID `1860`

---

### 🔹 Step 7: Set Up Alertmanager

```bash
kubectl create configmap alertmanager-config --from-file=k8s/config/alertmanager-config.yaml
kubectl apply -f k8s/alertmanager-deployment.yaml
kubectl apply -f k8s/alertmanager-service.yaml
```

Update Prometheus config to include:

```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager-service:9093
```

Add sample alert rule in Prometheus config:

```yaml
groups:
- name: example
  rules:
  - alert: HighMemoryUsage
    expr: process_resident_memory_bytes > 100000000
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Memory usage high"
```

🔔 Alertmanager sends alerts to email/Slack/SMS

---

### 🔹 Step 8: Access Dashboards

```bash
minikube service prometheus-service
minikube service grafana-service
```

---

## 🔍 How to Analyze in Prometheus

### 1. Navigate to `http://<minikube-ip>:30090`

* **Status → Targets**: Ensure all endpoints are UP
* **Graph Tab**:

  * Type: `rate(http_requests_total[1m])`
  * View trends over time

---

## 📊 How to Analyze in Grafana

### 1. Navigate to `http://<minikube-ip>:30300`

* Add Prometheus data source
* Import Dashboards:

  * Node.js → ID `11074`
  * Node Exporter → ID `1860`
  * K8s Cluster → ID `6417` or `315`

### 2. Build Panels:

* Memory usage: `process_resident_memory_bytes`
* App latency: `http_request_duration_seconds`
* Pod restarts: `kube_pod_container_status_restarts_total`
* Disk space: `node_filesystem_free_bytes`

---

## 🧼 Cleanup

```bash
kubectl delete -f k8s/
kubectl delete configmap prometheus-config alertmanager-config
```

---

## ✅ Summary of Key Concepts

| Concept               | How it Works                                        |
| --------------------- | --------------------------------------------------- |
| `/metrics` endpoint   | Your app exposes this for Prometheus to scrape      |
| Prometheus scrape job | Collects data at intervals (default 15s)            |
| Alertmanager          | Triggers actions based on defined conditions        |
| Node Exporter         | Gives raw node stats like CPU & memory              |
| Kube-State-Metrics    | Exposes Kubernetes API metrics in Prometheus format |
| PVC                   | Ensures metric data is not lost after restart       |
| Grafana               | Powerful frontend to visualize Prometheus metrics   |

---


## 🧼 Cleanup

```bash
kubectl delete -f k8s/
```

---

