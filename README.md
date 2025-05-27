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
│   ├── node-exporter-service.yaml               
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

# Set up **real-time application monitoring** for your Node.js app using **Prometheus and Grafana**, so you can track:

* 🟢 Number of incoming requests
* 🟠 CPU & memory usage
* 🔴 Signs of performance degradation

---

## ✅ Step-by-Step: Real-Time Node.js Application Monitoring

---

### 🔹 **Step 1: Expose Prometheus Metrics in Your Node.js App**

#### In `index.js`:

```js
const express = require('express');
const app = express();
const client = require('prom-client');

// Default metrics (CPU, memory, event loop lag, etc.)
client.collectDefaultMetrics();

// Custom counter: tracks HTTP requests
const httpRequestCounter = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
});

app.get('/', (req, res) => {
  httpRequestCounter.inc(); // Increment request count
  res.send('<h1>Hello from Real-Time Node.js Application Monitoring!</h1>');
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.listen(3000, () => {
  console.log('App running on port 3000');
});
```

#### In `package.json`:

```json
{
  "dependencies": {
    "express": "^4.18.2",
    "prom-client": "^14.1.1"
  }
}
```


#### In Dockerfile
```
# Use official Node.js LTS image
FROM node:18

# Set working directory
WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm install

# Copy the application source
COPY . .

# Expose application port
EXPOSE 3000

# Start the application
CMD ["node", "index.js"]


```

---

### 🔹 **Step 2: Build & Push Docker Image**

```bash
# Navigate to app/ directory
docker build -t laly9999/node-monitoring-app:1 .
docker run -d -p 3000:3000 laly9999/node-monitoring-app:1
docker push laly9999/node-monitoring-app:1
```
![image](https://github.com/user-attachments/assets/455959d1-6776-46b5-a996-8515f540891c)

---

### 🔹 **Step 3: Deploy to Kubernetes**

```bash
# Create Namespace
kubectl apply -f k8s/namespace.yaml
#  Deploy Node.js App                 
kubectl apply -f k8s/app-deployment.yaml
kubectl apply -f k8s/app-service.yaml
kubectl get all -n monitoring

#minikube service node-app-service -n monitoring
# In your VM:
kubectl port-forward svc/node-app-service 3000:3000 -n monitoring
# Then from your host machine, SSH with port forwarding:
ssh -L 3000:localhost:3000 lili@<your-vm-ip>
# Open in browser: http://localhost:3000

```
* This creates a Pod for your app and exposes port 3000 internally.

![image](https://github.com/user-attachments/assets/83162c17-c6d0-4140-9566-5ef6555ddb55)

![image](https://github.com/user-attachments/assets/afa9bdc8-5374-4a08-9c3e-831471fe74ab)

---

### Full File: prometheus.yaml
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']
      - job_name: 'node-app'
        static_configs:
          - targets: ['node-app-service:3000']
      - job_name: 'node-exporter'
        static_configs:
          - targets: ['node-exporter:9100']
      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics:8080']
---
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
          image: prom/prometheus:latest
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config-volume
              mountPath: /etc/prometheus/
              readOnly: true
            - name: prometheus-storage
              mountPath: /prometheus
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-config
        - name: prometheus-storage
          persistentVolumeClaim:
            claimName: prometheus-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
    - port: 9090
      targetPort: 9090
      nodePort: 30090


```


### 🔹 **Step 4: Update Prometheus to Scrape App Metrics**
In `prometheus-config.yaml`:

```yaml
scrape_configs:
  - job_name: 'node-app'
    static_configs:
      - targets: ['node-app-service:3000']
```


```
# Create PVC for Prometheus (persistent metrics)
kubectl apply -f k8s/pvc-prometheus.yaml

# Deploy Prometheus
kubectl apply -f k8s/prometheus.yaml


# Create PVC for Grafana (persistent dashboards)
kubectl apply -f k8s/pvc-grafana.yaml

# Deploy Grafana
kubectl apply -f k8s/grafana-deployment.yaml
kubectl apply -f k8s/grafana-service.yaml

# Verify Everything Is Running
kubectl get all -n monitoring

kubectl get pods -n monitoring
kubectl logs -n monitoring -l app=prometheus


```
#kubectl rollout restart deployment prometheus -n monitoring


---

### 🔹 **Step 5: Verify in Prometheus UI**

```bash
#minikube service prometheus-service -n monitoring
kubectl port-forward svc/prometheus 9090:9090 -n monitoring
ssh -L 9090:localhost:9090 lili@<your-vm-ip>
# Open in browser: http://localhost:9090
```
![image](https://github.com/user-attachments/assets/d92d23fc-e652-4dac-9244-b2bb3e98173b)

* Go to `Status → Targets`
* Confirm `node-app-service:3000` is `UP`
* Run queries in **Graph** tab:

  * 🔢 Requests: `http_requests_total`
    ![image](https://github.com/user-attachments/assets/1ebcb7c1-7195-44c8-aec8-afddfbe56282)

  * 🧠 Memory: `process_resident_memory_bytes`
    ![image](https://github.com/user-attachments/assets/0c5744b1-8658-42c6-8467-65895c5e65b6)

  * 🧮 CPU: `process_cpu_seconds_total`
     ![image](https://github.com/user-attachments/assets/cfc9de3c-28bc-48f0-837d-ca1a6e0c6224)

---

### 🔹 **Step 6: Create Grafana Dashboard**

```bash
#minikube service grafana-service -n monitoring
kubectl port-forward svc/grafana-service 3001:3000 -n monitoring
ssh -L 3001:localhost:3001 lili@10.0.0.146
# Open in browser: http://localhost:3001
```
![image](https://github.com/user-attachments/assets/8db2a5ec-ebde-4964-8b38-d74ea06b3b0e)

#### In Grafana UI:

1. Go to **Settings → Data Sources → Add Prometheus**

   * URL: `http://prometheus:9090`
  ![image](https://github.com/user-attachments/assets/f17d3d29-b7d5-4e51-a34a-e78f80e76f00)

     
2. Create a new **Dashboard → Panel**
3. Add the following PromQL queries:

| Metric Title  | PromQL Query                          |
| ------------- | ------------------------------------- |
| Request Count | `http_requests_total`                 |
| Memory Usage  | `process_resident_memory_bytes`       |
| CPU Usage     | `rate(process_cpu_seconds_total[1m])` |

4. Add thresholds to panels (e.g., turn red if memory > 150MB).
![image](https://github.com/user-attachments/assets/12c20347-d879-4b8e-b756-ab3b3246a00f)

---

### 🔹 **Step 7: Add Alerts (Optional)**

In Prometheus or Grafana, define alert rules like:

```yaml
- alert: HighMemoryUsage
  expr: process_resident_memory_bytes > 150000000
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "App memory usage is too high"
```

✅ This triggers an alert in Alertmanager, Slack, or email if configured.

---

## 🎯 Final Outcome

After following these steps, you will be able to:

* View **live request counts** to your app
* Monitor **memory and CPU trends** over time
* Catch **spikes in usage or degradation** before they impact users

---
---

# Set up **Infrastructure Visibility** in your Kubernetes cluster using **Node Exporter** and **Kube-State-Metrics** with Prometheus and Grafana.

This will give you deep insights into:

* ✅ Node-level metrics (CPU, memory, disk)
* ✅ Pod health and restarts
* ✅ Deployment status and replica counts

---

## ✅ Step-by-Step: Infrastructure Monitoring with Node Exporter & Kube-State-Metrics

---

### 🔹 **Step 1: Deploy Node Exporter**

```bash
kubectl apply -f k8s/node-exporter-daemonset.yaml
kubectl apply -f k8s/node-exporter-service.yaml
kubectl get svc -n monitoring | grep node-exporter

```

#### 🔍 What it does:

* Runs a **DaemonSet** that deploys `node-exporter` on **every node**
* Exposes host-level metrics on port `9100`

#### 🔧 What Prometheus will collect:

* `node_cpu_seconds_total`
* `node_memory_Active_bytes`
* `node_filesystem_avail_bytes`
* `node_load1`, `node_load5`, `node_load15`

---

### 🔹 **Step 2: Deploy Kube-State-Metrics**

```bash
kubectl apply -f k8s/kube-state-metrics.yaml
```

#### 🔍 What it does:

* Runs a Deployment and a Service exposing Kubernetes object metrics
* Exposes metrics on port `8080`

#### 🔧 What Prometheus will collect:

* `kube_pod_status_phase`
* `kube_pod_container_status_restarts_total`
* `kube_deployment_status_replicas`
* `kube_node_status_allocatable_cpu_cores`

---

### 🔹 **Step 3: Update Prometheus Scrape Config**

In `k8s/config/prometheus-config.yaml`:

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics:8080']
```

#### Then:

```bash
kubectl delete configmap prometheus-config -n monitoring
kubectl create configmap prometheus-config --from-file=k8s/config/prometheus-config.yaml -n monitoring
kubectl rollout restart deployment prometheus -n monitoring
```

---

### 🔹 **Step 4: Verify in Prometheus UI**

```bash
minikube service prometheus-service -n monitoring
```
![image](https://github.com/user-attachments/assets/52a928b1-069c-460d-8173-4ba764c1b452)

#### In Prometheus UI:

* Go to **Status → Targets**
* Confirm `node-exporter` and `kube-state-metrics` jobs are **UP**
* Run sample queries:

| Purpose             | PromQL                                           |
| ------------------- | ------------------------------------------------ |
| Node CPU usage      | `rate(node_cpu_seconds_total{mode!="idle"}[1m])` |
| Memory available    | `node_memory_MemAvailable_bytes`                 |
| Disk usage          | `node_filesystem_avail_bytes`                    |
| Pod restarts        | `kube_pod_container_status_restarts_total`       |
| Running pods        | `kube_pod_status_phase{phase="Running"}`         |
| Deployment replicas | `kube_deployment_status_replicas_available`      |

![image](https://github.com/user-attachments/assets/327ce353-3d6d-43da-9295-ce1961f83c1c)

---

### 🔹 **Step 5: Visualize in Grafana**

```bash
minikube service grafana-service -n monitoring
```

#### In Grafana UI:

1. **Add Prometheus** as data source if not done already
2. **Import prebuilt dashboards**:

| Dashboard              | ID              | Description                          |
| ---------------------- | --------------- | ------------------------------------ |
| Node Exporter Full     | `1860`          | System metrics: CPU, Memory, Disk    |
| K8s Cluster Monitoring | `315` or `6417` | Pods, Deployments, ReplicaSets, etc. |

![image](https://github.com/user-attachments/assets/9b39961d-7e86-4144-9219-92d543a4da4d)


3. Or **build your own panels** with queries like:

   * CPU Load: `rate(node_cpu_seconds_total{mode!="idle"}[5m])`
   * Memory Usage: `node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes`
   * Pod Restarts: `sum(kube_pod_container_status_restarts_total)`
![image](https://github.com/user-attachments/assets/afaba8e7-8821-4d7f-958b-7fde82954aac)

---

### 🔹 **Step 6: (Optional) Set Alerts in Prometheus or Grafana**

Example Prometheus rule for too many restarts:

```yaml
- alert: PodCrashLooping
  expr: increase(kube_pod_container_status_restarts_total[5m]) > 3
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Pod restart count is high"
```

---

## 🧠 What You Gain

| Layer          | Visibility                                     |
| -------------- | ---------------------------------------------- |
| **Node**       | CPU, memory, disk, load                        |
| **Pod**        | Status, restarts, container states             |
| **Deployment** | Desired vs available replicas, rollout health  |
| **Cluster**    | High-level health view (dashboards and alerts) |

---



## 🧼 Cleanup

```bash
kubectl delete -f k8s/
```

---

