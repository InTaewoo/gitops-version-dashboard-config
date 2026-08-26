# Istio Sidecar and v1/v2 Canary Lab

## Deployment order

1. Install Istio 1.30.3 base, istiod, and the ingress gateway.
2. Confirm that the ingress gateway pod label matches `istio=ingress`.
3. Apply `argocd/application.yaml`, or patch the live Application, so it uses `master` and `PruneLast=true`.
4. Commit and push this configuration repository first.
5. Confirm that both dashboard pods are `2/2 Running` and Argo CD is `Synced/Healthy`.
6. Commit and push the application v2 change. The CI pipeline updates only the canary image tag and `app.env`.
7. Test traffic through ingress gateway NodePort 30081.

The `argocd/application.yaml` file is not part of the application Kustomize overlay. Before pushing the v1/v2 migration, update the live Application explicitly:

```bash
kubectl apply -f argocd/application.yaml
```

If the repository is not cloned on the master, use this equivalent patch:

```bash
kubectl patch application gitops-version-dashboard -n argocd --type merge \
  -p '{"spec":{"source":{"targetRevision":"master"},"syncPolicy":{"automated":{"prune":true,"selfHeal":true},"syncOptions":["CreateNamespace=true","PruneLast=true"]}}}'
```

## Verify the sidecars

```bash
kubectl get pods -n production
kubectl get pod -n production -l version=v1 -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'
kubectl get pod -n production -l version=v2 -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'
```

Each application pod must contain `dashboard` and `istio-proxy` containers.

## Verify Istio resources

```bash
kubectl get gateway,virtualservice,destinationrule -n production
kubectl get endpointslice -n production -l kubernetes.io/service-name=gitops-version-dashboard
```

## Open an SSM tunnel

```powershell
aws ssm start-session --target i-06453c2b5c05141e9 --document-name AWS-StartPortForwardingSession --parameters "portNumber=30081,localPortNumber=8081" --region ap-northeast-2
```

Open `http://localhost:8081`.

## Check the 90/10 distribution

```powershell
1..200 | ForEach-Object {
  (Invoke-RestMethod http://localhost:8081/api/info).imageTag
} | Group-Object | Select-Object Count, Name
```

## Promote the canary manually

Change the two weights in `overlays/prod/virtual-service.yaml`:

- `90/10`: initial canary
- `50/50`: expanded canary
- `0/100`: full promotion to v2

Commit and push each weight change. Argo CD updates routing without rebuilding the image.

## Remove the fallback NodePort

The application NodePort 30080 remains available during the first test. After gateway verification, remove `service-patch.yaml` from `overlays/prod/kustomization.yaml` and delete the file so that only the Istio gateway is externally reachable.
