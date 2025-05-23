# Node.js-App-Monitoring-with-Prometheus-and-Grafana


## 🎯 Project Title: **Node.js App Monitoring on Kubernetes with Prometheus and Grafana**

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
├── app/                             # Node.js app exposing /metrics
├── k8s/
│   ├── namespace.yaml               # (Optional) Create a namespace
│   ├── app-deployment.yaml         # Deploy Node.js app
│   ├── app-service.yaml            # ClusterIP service for the app
│   ├── prometheus-deployment.yaml  # Deploy Prometheus with config & PVC
│   ├── prometheus-service.yaml     # NodePort to access Prometheus
│   ├── grafana-deployment.yaml     # Deploy Grafana with PVC
│   ├── grafana-service.yaml        # NodePort to access Grafana
│   ├── alertmanager-deployment.yaml
│   ├── alertmanager-service.yaml
│   ├── node-exporter-daemonset.yaml
│   ├── kube-state-metrics.yaml
│   ├── pvc-prometheus.yaml
│   ├── pvc-grafana.yaml
│   └── config/
│       ├── prometheus-config.yaml
│       └── alertmanager-config.yaml
```

---
## setup_monitoring.py

```python

import os

base_dir = "/home/lili/monitoring-project"
structure = {
    "app/index.js": """\
const express = require('express');
const app = express();
const client = require('prom-client');

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
""",
    "app/package.json": """\
{
  "name": "node-monitoring-app",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "express": "^4.18.2",
    "prom-client": "^14.1.1"
  }
}
""",
    "k8s/namespace.yaml": """\
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
""",
    "k8s/app-deployment.yaml": """\
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
      - name: node-app
        image: laly9999/node-monitoring-app:1
        ports:
        - containerPort: 3000
""",
    "k8s/app-service.yaml": """\
apiVersion: v1
kind: Service
metadata:
  name: node-app-service
  namespace: monitoring
spec:
  selector:
    app: node-app
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP
""",
    "k8s/prometheus-deployment.yaml": """\
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config-volume
              mountPath: /etc/prometheus/
            - name: data
              mountPath: /prometheus
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-config
        - name: data
          persistentVolumeClaim:
            claimName: prometheus-pvc
""",
    "k8s/prometheus-service.yaml": """\
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
    - port: 9090
      targetPort: 9090
      nodePort: 30090
  type: NodePort
""",
    "k8s/grafana-deployment.yaml": """\
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana
        ports:
        - containerPort: 3000
        volumeMounts:
          - name: grafana-storage
            mountPath: /var/lib/grafana
      volumes:
        - name: grafana-storage
          persistentVolumeClaim:
            claimName: grafana-pvc
""",
    "k8s/grafana-service.yaml": """\
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30300
  type: NodePort
""",
    "k8s/alertmanager-deployment.yaml": """\
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
        - name: alertmanager
          image: prom/alertmanager
          args:
            - "--config.file=/etc/alertmanager/config.yml"
          volumeMounts:
            - name: config-volume
              mountPath: /etc/alertmanager/
      volumes:
        - name: config-volume
          configMap:
            name: alertmanager-config
""",
    "k8s/alertmanager-service.yaml": """\
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-service
  namespace: monitoring
spec:
  selector:
    app: alertmanager
  ports:
    - port: 9093
      targetPort: 9093
      nodePort: 30903
  type: NodePort
""",
    "k8s/node-exporter-daemonset.yaml": """\
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter
          ports:
            - containerPort: 9100
              name: metrics
""",
    "k8s/kube-state-metrics.yaml": """\
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      containers:
        - name: kube-state-metrics
          image: k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.10.1
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: kube-state-metrics
""",
    "k8s/pvc-prometheus.yaml": """\
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
""",
    "k8s/pvc-grafana.yaml": """\
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
""",
    "k8s/config/prometheus-config.yaml": """\
global:
  scrape_interval: 15s
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager-service:9093']
scrape_configs:
  - job_name: 'node-app'
    static_configs:
      - targets: ['node-app-service:3000']
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics:8080']
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
  - job_name: 'alertmanager'
    static_configs:
      - targets: ['alertmanager-service:9093']
""",
    "k8s/config/alertmanager-config.yaml": """\
global:
  resolve_timeout: 5m

route:
  receiver: "default"

receivers:
  - name: "default"
    email_configs:
      - to: "your-email@example.com"
        from: "alertmanager@example.com"
        smarthost: "smtp.example.com:587"
        auth_username: "alertmanager@example.com"
        auth_password: "yourpassword"
"""
}

# Create directories and write files
for path, content in structure.items():
    file_path = os.path.join(base_dir, path)
    os.makedirs(os.path.dirname(file_path), exist_ok=True)
    with open(file_path, "w") as f:
        f.write(content)

"All files created successfully in /home/lili/monitoring-project."

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

