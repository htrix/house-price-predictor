# Kubernetes Deployment Code

This folder contains Kubernetes manifests (and a `kustomization.yaml`) for deploying:
1. The Streamlit frontend (`streamlit-deploy.yaml`, `streamlit-svc.yaml`)
2. The model inference backend (`model-deploy.yaml`, `model-svc.yaml`)

## - Deploying to Kubernetes (Steps)

### 1. Setup Kubernetes Cluster
Setup a 3 node Kubernetes cluster following your course/lab instructions (Lab K801).

Validate your environment:
```bash
kubectl get nodes
```

You should see 3 nodes listed and in `Ready` status.

Optionally verify pods are running:
```bash
kubectl get pods -A
```

### 2. Deploy Streamlit App to Kubernetes
```bash
kubectl create deployment streamlit --image=htrix/streamlit:latest --port=8501
```

Validate:
```bash
kubectl get deploy
kubectl describe deploy
kubectl get pods
```

Try scaling:
```bash
kubectl scale deploy streamlit --replicas=3
kubectl get all
```

Scale back to one:
```bash
kubectl scale deploy streamlit --replicas=1
```

### 3. Expose the App with a Service
Create a NodePort service:
```bash
kubectl create service nodeport streamlit --tcp=8501 --node-port=30000
```

Validate:
```bash
kubectl get svc,ep,pods -o wide
kubectl describe svc streamlit
```

Access the app:
http://localhost:30000/

### 4. Setup Model Inference on Kubernetes
Create the model deployment and service:
```bash
kubectl create deployment model --image=<YOUR_DOCKERHUB_USERNAME>/house-price-model:latest --port=8000
kubectl create service nodeport model --tcp=8000 --node-port=30100
```

Validate:
```bash
kubectl get all
```

Access FastAPI docs:
http://localhost:30100/docs

### 5. Connecting Streamlit to Model Backend
Your Streamlit app is configured to connect to the model using hostname `model` (as used in your app code).

At this point, the frontend should be able to reach the backend via the Kubernetes service.

### 6. Auto Generating Kubernetes Manifests (YAML + Kustomize)
Instead of creating resources manually, generate YAML manifests and manage them with Kustomize.

```bash
cd house-price-predictor/deployment/kubernetes
```

Generate Streamlit manifests:
```bash
kubectl create deployment streamlit \
  --image=htrix/streamlit:latest \
  --port=8501 \
  -o yaml --dry-run=client > streamlit-deploy.yaml
```

```bash
kubectl create service nodeport streamlit \
  --tcp=8501 --node-port=30000 \
  -o yaml --dry-run=client > streamlit-svc.yaml
```

Generate model manifests (replace `xxxxxx` / `<YOUR_DOCKERHUB_USERNAME>`):
```bash
kubectl create deployment model \
  --image=<YOUR_DOCKERHUB_USERNAME>/house-price-model:latest \
  --port=8000 \
  -o yaml --dry-run=client > model-deploy.yaml
```

```bash
kubectl create service nodeport model \
  --tcp=8000 --node-port=30100 \
  -o yaml --dry-run=client > model-svc.yaml
```

Your `kustomization.yaml` references:
- `model-deploy.yaml`
- `model-svc.yaml`
- `streamlit-deploy.yaml`
- `streamlit-svc.yaml`

Note: in this repo, `model-deploy.yaml` currently contains a placeholder image
`xxxxxx/house-price-model:latest`. Replace it with your real Docker image before applying.

### 7. Delete Previous Deployments and Apply Manifests
Delete previous resources:
```bash
kubectl delete deploy,svc model streamlit
```

Apply with Kustomize:
```bash
cd house-price-predictor
kubectl apply -k deployment/kubernetes
```

## Lab 9 - Autoscaling Models

### Step 1: Install KEDA
```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda \
--namespace keda \
--create-namespace

Validate:

```bash
kubectl get all -n keda
```

Step 2: Add Resource Spec to the Pod
Edit deployment/kubernetes/model-deploy.yaml and add CPU/memory requests + limits to the model container:

```
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "100m"
    memory: "128Mi"

```

Then apply:

kubectl apply -k deployment/kubernetes
Step 3: Pick Custom Metric (Latency p95)
From the Grafana/Prometheus dashboard JSON used earlier, pick a p95 latency query, e.g.:

histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m]))
by (le))
Scale up if p95 latency > 0.5 seconds.

Step 4: Create a KEDA ScaledObject
Create fastapi-scaledobject.yaml:

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: fastapi-latency-autoscaler
  namespace: default
spec:
  scaleTargetRef:
    name: model
  minReplicaCount: 1
  maxReplicaCount: 5
  pollingInterval: 30
  cooldownPeriod: 300
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prom-kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: fastapi_latency_p95
      query: |
        histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le))
      threshold: "0.5"
Step 5: Apply the ScaledObject
```
kubectl apply -f fastapi-scaledobject.yaml
kubectl get scaledobject
kubectl get hpa
```
Step 6: Add Another Metric (Multi-Trigger Scaling)
Add request-rate scaling as a second trigger in fastapi-scaledobject.yaml:
```
  - type: prometheus
    metadata:
      serverAddress: http://prom-kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: request_rate
      query: sum(rate(http_requests_total[1m]))
      threshold: "1000"
```
Re-apply:
```
kubectl apply -f fastapi-scaledobject.yaml
kubectl get scaledobject,hpa,pods
```
Step 7: Run a Load Test
Install hey:

macOS:
```
brew install hey
```
Linux:
```
sudo snap install hey
```
or
```
go install github.com/rakyll/hey@latest
```
Create predict.json:
```
{
  "sqft": 4500,
  "bedrooms": 4,
  "bathrooms": 2,
  "year_built": 2014,
  "condition": "Good",
  "location": "Urban"
}
```
Try one prediction:
```
curl -X POST http://localhost:30100/predict \
-H "Content-Type: application/json" \
-d @predict.json
```
Run a longer load test:
```
hey -n 5000 -c 200 -m POST \
-H "Content-Type: application/json" \
-D predict.json \
http://localhost:30100/predict
```
Run for a duration:
```
hey -z 3m -c 200 -m POST \
-H "Content-Type: application/json" \
-D predict.json \
http://localhost:30100/predict
```
Troubleshooting: Separate Endpoint for FastAPI Metrics (9100)
If metrics need a separate endpoint, add a dedicated metrics server on port 9100.

Update src/api/main.py (somewhere between app = FastAPI( and endpoints):
from prometheus_client import start_http_server
import threading
# Start Prometheus metrics server on port 9100 in a background thread
```
def start_metrics_server():
    start_http_server(9100)
```
Update root Dockerfile to expose:

```
EXPOSE 8000 9100
```
Update deployment/kubernetes/model-svc.yaml to include a metrics port:
```
- name: "metrics"
  port: 9100
  targetPort: 9100
```
Update the Prometheus ServiceMonitor to scrape that endpoint (in deployment/monitoring/servicemonitor.yaml) and then re-apply manifests.
After a few minutes, check Prometheus targets: http://localhost:30300/targets

PART II: Adding CPU Based Scaling
Install metrics-server:
```
cd ~
git clone https://github.com/schoolofdevops/metrics-server.git
kubectl apply -k metrics-server/manifests/overlays/release
```
Wait for it to collect metrics, then validate:
```
kubectl top pods
kubectl top nodes
```
Update fastapi-scaledobject.yaml to add a CPU trigger (in addition to the Prometheus one):
```
- type: cpu
  metricType: Utilization
  metadata:
    value: "50"
```
Re-apply, then run the load test again to see scaling.

Using VerticalPodAutoscaler (VPA)
Deploy VPA:

```
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/
./hack/vpa-up.sh
```

Create model-vpa.yaml:

```
---
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: model
  labels:
    role: model
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: model
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 500m
        memory: 512Mi
      controlledResources: ["cpu", "memory"]
```

Apply and watch:

```bash
kubectl apply -f model-vpa.yaml
kubectl get vpa model --watch
```