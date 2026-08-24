# GitOps Version Dashboard Config

`gitops-version-dashboard`를 Kubernetes에 배포하기 위한 GitOps 설정 저장소입니다.

## 구성

```text
gitops-version-dashboard-config/
├── argocd/
│   └── application.yaml
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    └── prod/
        ├── app.env
        ├── service-patch.yaml
        └── kustomization.yaml
```

## 사용 전 변경할 값

다음 두 파일의 `REPLACE_WITH_AWS_ACCOUNT_ID`를 실제 12자리 AWS Account ID로 변경합니다.

- `overlays/prod/app.env`
- `overlays/prod/kustomization.yaml`

ECR Repository 이름은 `gitops-version-dashboard`, AWS Region은 `ap-northeast-2`를 기준으로 작성했습니다.

## Kustomize 결과 확인

```bash
kubectl kustomize overlays/prod
```

## Argo CD Application 등록

```bash
kubectl apply -f argocd/application.yaml
kubectl get applications -n argocd
```

## 배포 확인

```bash
kubectl get pods,svc -n production -o wide
curl http://<WORKER_NODE_IP>:30080/health
```

