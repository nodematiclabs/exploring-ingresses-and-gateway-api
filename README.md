# Ingresses and Gateway API Demo (Across Nginx and Native-GCP)

## System Workloads Installation

Set environment variables (replace with your own GCP project)
```bash
export GOOGLE_PROJECT=example
```

Create a sandbox cluster
```bash
gcloud container clusters create comparing-ingress-controllers \
    --zone=us-central1-a \
    --gateway-api=standard \
    --addons ConfigConnector \
    --workload-pool=$GOOGLE_PROJECT.svc.id.goog \
    --scopes https://www.googleapis.com/auth/cloud-platform
gcloud container clusters get-credentials comparing-ingress-controllers --zone=us-central1-a
```

Create a service account with DNS Admin permissions for Cert Manager
```bash
gcloud iam service-accounts create cert-manager \
    --display-name="Cert Manager"
gcloud projects add-iam-policy-binding $GOOGLE_PROJECT \
    --member="serviceAccount:cert-manager@$GOOGLE_PROJECT.iam.gserviceaccount.com" \
    --role="roles/dns.admin"
gcloud projects add-iam-policy-binding $GOOGLE_PROJECT \
    --member="serviceAccount:$GOOGLE_PROJECT.svc.id.goog[default/cert-manager]" \
    --role="roles/iam.workloadIdentityUser"
```

Install Cert Manager
```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager --set installCRDs=true
kubectl annotate serviceaccount \
    cert-manager \
    "iam.gke.io/gcp-service-account=cert-manager@$GOOGLE_PROJECT.iam.gserviceaccount.com" \
    --overwrite
```

Modify the [cert-manager/cert-manager.yaml](cert-manager/cert-manager.yaml) to use your domain, then create Cert Manager certificates/configurations then
```bash
kubectl apply -f cert-manager/cert-manager.yaml
```

Create a service account with Editor permissions for Config Connector
```bash
gcloud iam service-accounts create config-connector \
    --display-name="Config Connector"
gcloud projects add-iam-policy-binding $GOOGLE_PROJECT \
    --member="serviceAccount:config-connector@$GOOGLE_PROJECT.iam.gserviceaccount.com" \
    --role="roles/editor"
gcloud projects add-iam-policy-binding $GOOGLE_PROJECT \
    --member="serviceAccount:$GOOGLE_PROJECT.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
    --role="roles/iam.workloadIdentityUser"
```

Modify the [config-connector/config.yaml](config-connector/config.yaml) to use your Config Connector service account then
```bash
kubectl apply -f config-connector/config.yaml
kubectl annotate namespace default cnrm.cloud.google.com/project-id=$GOOGLE_PROJECT
```

Install Nginx Ingress Controller
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx
```

## Nginx 

Apply test ingress configurations:
```bash
kubectl apply -f nginx/
```

## GCE Ingress

_The "HTTP Load Balancing" GKE feature (enabled by default) is required for these_

Apply test ingress configurations:
```bash
kubectl apply -f gce-ingress/
```

## Gateway API

Apply test ingress configurations:
```bash
kubectl apply -f gke-gateway/
```