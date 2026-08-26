# Istio 1.30.3 installation

Run these commands on `k8s-master` with a working kubeconfig.

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm upgrade --install istio-base istio/base \
  --namespace istio-system \
  --create-namespace \
  --version 1.30.3 \
  --set defaultRevision=default \
  --wait

helm upgrade --install istiod istio/istiod \
  --namespace istio-system \
  --version 1.30.3 \
  --wait

helm upgrade --install istio-ingress istio/gateway \
  --namespace istio-ingress \
  --create-namespace \
  --version 1.30.3 \
  --values platform/istio/ingress-values.yaml \
  --wait
```

The values file defines the complete service port list, avoiding Helm sparse-list schema errors when assigning NodePort 30081.

Verify the installation:

```bash
helm list -n istio-system
helm list -n istio-ingress
kubectl get pods -n istio-system
kubectl get pods,svc -n istio-ingress
kubectl get pods -n istio-ingress -l istio=ingress
```
