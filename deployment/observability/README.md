## Setting up Model Monitoring

This lab sets up Prometheus + Grafana (via Helm), instruments the FastAPI backend for `/metrics`, and configures a Prometheus `ServiceMonitor` to scrape those metrics.

### 1. Setup Prometheus and Grafana with HELM
#### Install Helm (Helm v3)
On Linux/MacOS you can install Helm v3 with:
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Verify installation:
```bash
helm --help
helm version
```

#### Deploy Prometheus Stack with HELM
Add/update the Helm repo:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install the chart (Prometheus + Grafana):
```bash
helm upgrade --install prom \
  -n monitoring \
  --create-namespace \
  prometheus-community/kube-prometheus-stack \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30200 \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30300
```

Validate:
```bash
helm list -A
kubectl get all -n monitoring
```

You should be able to access:
- Prometheus: `http://localhost:30300/`
- Grafana: `http://localhost:30200/`

Login to Grafana:
- Username: `admin`
- Password: `prom-operator`

### 2. Build Model Monitoring Dashboard (Instrumentation + Scraping)
#### Add Instrumentation for FastAPI
Edit `src/api/main.py` and add `prometheus_fastapi_instrumentator.Instrumentator`, then instrument your FastAPI app:

Add import:
```python
from prometheus_fastapi_instrumentator import Instrumentator  # Add this
```

Then instrument:
```python
# Initialize and instrument Prometheus metrics
Instrumentator().instrument(app).expose(app)  # Add this
```

#### Add dependency
Append this to `src/api/requirements.txt`:
```txt
prometheus-fastapi-instrumentator==6.1.0
```

Commit and push the changes (as described in the lab):
```bash
git add src/api/main.py src/api/requirements.txt
git commit -am "add fastapi instrumentation"
git push origin
```

After pushing, your CI pipeline should rebuild and republish your Docker image.

#### Deploy this change to Kubernetes
Restart the model deployment:
```bash
kubectl rollout restart deployment model
```

Verify the service exposes `/metrics` on the FastAPI deployment.

#### Create ServiceMonitor for Prometheus scraping
Create the ServiceMonitor manifest at:
`deployment/monitoring/servicemonitor.yaml`

Example `ServiceMonitor` (adjust if needed to match your namespace/labels):
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: house-price-api-monitor
  labels:
    release: prom   # Match the Helm release name used above
spec:
  selector:
    matchLabels:
      app: model
  namespaceSelector:
    matchNames:
      - default   # or your namespace
  endpoints:
    - port: "8000"   # match model service port
      path: /metrics
      interval: 15s
```

Apply it:
```bash
kubectl apply -f deployment/monitoring/servicemonitor.yaml
```

Check creation:
```bash
kubectl get servicemonitor
kubectl describe servicemonitor
```

Validate Prometheus is scraping targets:
`http://localhost:30300/targets`

Optionally run example Prometheus queries:
```text
http_requests_total
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le, handler))
rate(http_request_size_bytes_sum[1m])
```

If queries return results, Prometheus is collecting FastAPI data.

### 3. Visualize in Grafana
In Grafana:
1. Go to `Dashboards` -> `New` -> `Import`
2. In the box that reads `Import via dashboard JSON model`, paste the dashboard JSON from the lab (`enhanced_fastapi_ml_dashboard`)

You should then have a model-specific monitoring dashboard (latency, requests, error rates, etc.).

