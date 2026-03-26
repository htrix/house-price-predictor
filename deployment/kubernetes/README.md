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

