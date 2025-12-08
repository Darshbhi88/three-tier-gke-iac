***

# Three‑Tier GKE IaC – React, Node.js, MongoDB, Prometheus & Grafana

This project deploys a production‑style **three‑tier web application** on **Google Kubernetes Engine (GKE)**:

- **Frontend**: React, built into a static bundle and served by Nginx  
- **Backend**: Node.js / Express REST API  
- **Database**: MongoDB  
- **Observability**: Prometheus + Grafana via `kube-prometheus-stack` Helm chart

Container images are stored in **Artifact Registry** and everything runs in a dedicated Kubernetes namespace.

***

## 1. Prerequisites

- Google Cloud project with billing enabled  
- gcloud SDK, kubectl, and Helm 3 installed  
- Docker available locally or via Cloud Shell  

Set your project:

```bash
gcloud config set project my-project-0011-473604
```

***

## 2. Create Artifact Registry

```bash
gcloud artifacts repositories create three-tier-repo \
  --repository-format=docker \
  --location=asia-south1 \
  --description="Three-tier app images"
```

Log Docker into Artifact Registry:

```bash
gcloud auth configure-docker asia-south1-docker.pkg.dev
```

Repository URL:

```text
asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo
```

***

## 3. Build and Push Images

### 3.1 Backend

From `app/backend`:

```bash
docker build -t asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/backend:v1 .
docker push asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/backend:v1
```

Folder layout:

```text
backend/
  index.js
  db.js
  models/task.js
  routes/tasks.js
  Dockerfile
```

Ensure imports like `require('../models/task')` match filenames exactly (case‑sensitive).

### 3.2 Frontend

Dockerfile uses Node 16 to build and Nginx to serve:

```dockerfile
FROM node:16-alpine AS build
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /usr/src/app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build and push from `app/frontend`:

```bash
docker build -t asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/frontend:v1 .
docker push asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/frontend:v1
```

Frontend uses `REACT_APP_BACKEND_URL=http://api:8080` to talk to the backend.

### 3.3 MongoDB (optional mirror)

```bash
docker pull mongo:4.4.6
docker tag mongo:4.4.6 asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/mongo:4.4.6
docker push asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/mongo:4.4.6
```

***

## 4. Create GKE Cluster & Namespace

Create a Standard GKE cluster (e.g. via console) with:

- Region: `asia-south1`  
- Node pool: `e2-standard-2`, 2–3 nodes  

Connect and create namespace:

```bash
gcloud container clusters get-credentials three-tier-cluster \
  --region asia-south1

kubectl create namespace workshop
kubectl config set-context --current --namespace=workshop
```

***

## 5. Kubernetes Manifests

### 5.1 MongoDB

`k8s_manifests/mongo-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
meta
  name: mongodb
  namespace: workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    meta
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/mongo:4.4.6
          args: ["--bind_ip_all"]
          ports:
            - containerPort: 27017
```

`k8s_manifests/mongo-service.yaml`:

```yaml
apiVersion: v1
kind: Service
meta
  name: mongodb-svc
  namespace: workshop
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
  type: ClusterIP
```

### 5.2 Backend

`k8s_manifests/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
meta
  name: api
  namespace: workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    meta
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/backend:v1
          ports:
            - containerPort: 8080
          env:
            - name: MONGO_URL
              value: "mongodb://mongodb-svc:27017/mydb"
```

`k8s_manifests/backend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
meta
  name: api
  namespace: workshop
spec:
  selector:
    app: api
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

### 5.3 Frontend

`k8s_manifests/frontend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
meta
  name: frontend
  namespace: workshop
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    meta
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: asia-south1-docker.pkg.dev/my-project-0011-473604/three-tier-repo/frontend:v1
          env:
            - name: REACT_APP_BACKEND_URL
              value: "http://api:8080"
          ports:
            - containerPort: 80
```

`k8s_manifests/frontend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
meta
  name: frontend
  namespace: workshop
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

***

## 6. Deploy the App

From `k8s_manifests`:

```bash
kubectl apply -f secrets.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```

Check resources:

```bash
kubectl get pods -n workshop
kubectl get svc -n workshop
```

When the `frontend` Service shows an `EXTERNAL-IP`, open:

```text
http://<EXTERNAL-IP>/
```

***

## 7. Monitoring with Prometheus & Grafana

This project uses the **kube‑prometheus‑stack** Helm chart for monitoring.

### 7.1 Install Helm repo & namespace

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring
```

### 7.2 Install kube‑prometheus‑stack

```bash
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
kubectl get pods -n monitoring
```

Wait until all pods in `monitoring` are `Running`.

### 7.3 Access Grafana

Port‑forward Grafana:

```bash
kubectl expose deployment kube-prometheus-stack-grafana --port=3000 --target-port=3000 --name=grafana --type=LoadBalancer -n monitor
```

Get the admin password:

```bash
kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

Open `http://<LoadBalancer IP>:3000` (Cloud Shell Web Preview or SSH tunnel) and log in with `admin` / `<password>`. [1][4]

Use built‑in Kubernetes dashboards and filter by namespace `workshop` to see metrics for:

- `frontend`
- `api`
- `mongodb`

***

## 8. Cleanup

To avoid ongoing costs:

```bash
# Delete app
kubectl delete namespace workshop

# Delete monitoring stack
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring

# Optionally delete GKE cluster
gcloud container clusters delete three-tier-cluster \
  --region asia-south1

# Optionally delete Artifact Registry repo
gcloud artifacts repositories delete three-tier-repo \
  --location=asia-south1
```

Deleting the whole GCP project will also remove all resources.

***