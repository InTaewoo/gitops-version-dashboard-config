# GitOps 배포 설정 저장소

`gitops-version-dashboard` 애플리케이션을 Kubernetes에 배포하기 위한 선언형 설정 저장소입니다. 애플리케이션 소스와 배포 상태를 분리하고, Argo CD가 이 저장소를 단일 진실 공급원으로 사용하도록 구성했습니다.

Kustomize로 공통 리소스와 운영 환경 설정을 조합하고, Istio로 v1/v2 카나리 트래픽 분배와 `STRICT mTLS`를 적용합니다.

## 전체 프로젝트에서의 역할

| 저장소 | 역할 |
| --- | --- |
| [aws-terraform-cicd-project](https://github.com/InTaewoo/aws-terraform-cicd-project) | AWS VPC, EC2, ECR, IAM 및 GitHub OIDC 구성 |
| [gitops-version-dashboard](https://github.com/InTaewoo/gitops-version-dashboard) | 애플리케이션 코드, 테스트, 이미지 빌드 및 GitOps 설정 자동 갱신 |
| [gitops-version-dashboard-config](https://github.com/InTaewoo/gitops-version-dashboard-config) | 이 저장소. Kubernetes의 원하는 상태와 Istio 정책 관리 |

## 구현한 배포 구조

```text
GitHub Actions
  → v2 이미지 태그와 app.env 변경
  → GitOps 저장소 master 브랜치 Push
  → Argo CD가 overlays/prod 감지
  → Kustomize Render
  → production Namespace 동기화
  → Deployment v1 + Deployment v2
  → 하나의 공통 Service
  → DestinationRule의 v1/v2 Subset
  → VirtualService가 v1 90%, v2 10%로 분배
```

외부 요청 흐름:

```text
사용자
  → Istio Ingress Gateway NodePort 30081
  → Gateway dashboard-gateway
  → VirtualService
      ├── 90% → subset v1 → Stable Pod
      └── 10% → subset v2 → Canary Pod
  → STRICT mTLS
  → 애플리케이션 컨테이너 8080
```

## 저장소 구조

```text
.
├── argocd
│   └── application.yaml
├── base
│   ├── deployment-v1.yaml
│   ├── deployment-v2.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays
│   └── prod
│       ├── app.env
│       ├── stable.env
│       ├── namespace.yaml
│       ├── gateway.yaml
│       ├── destination-rule.yaml
│       ├── virtual-service.yaml
│       ├── peer-authentication.yaml
│       ├── service-patch.yaml
│       └── kustomization.yaml
├── platform
│   └── istio
│       ├── ingress-values.yaml
│       └── README.md
├── ISTIO_CANARY.md
└── README.md
```

## Kubernetes 리소스 설계

### v1 Stable Deployment

`base/deployment-v1.yaml`은 검증된 안정 버전을 고정합니다.

- Deployment: `gitops-version-dashboard-v1`
- Replica: 1
- 라벨: `version: v1`
- 이미지: 검증된 ECR Commit SHA 태그
- ConfigMap: `dashboard-config-stable`
- Rolling Update: `maxUnavailable: 0`, `maxSurge: 1`

새 애플리케이션 Commit이 Push되어도 CI는 v1 이미지와 `stable.env`를 변경하지 않습니다. 이 방식으로 카나리 검증 중 안정 버전을 유지합니다.

### v2 Canary Deployment

`base/deployment-v2.yaml`은 새 버전 검증을 위한 카나리 워크로드입니다.

- Deployment: `gitops-version-dashboard-v2`
- Replica: 1
- 라벨: `version: v2`
- 이미지 별칭: `gitops-version-dashboard-canary`
- 실제 ECR Repository와 태그는 Prod Overlay의 `images`에서 치환
- ConfigMap: `dashboard-config-canary`
- GitHub Actions가 새 Commit마다 v2 태그와 `app.env`를 자동 갱신

### 공통 컨테이너 정책

- 컨테이너 포트 `8080`
- `/health` Readiness Probe
- `/health` Liveness Probe
- CPU Request/Limit: `50m` / `200m`
- Memory Request/Limit: `64Mi` / `128Mi`
- 권한 상승 금지
- 읽기 전용 Root Filesystem
- 모든 Linux Capability 제거
- Pod, Namespace 및 Node 정보를 Downward API로 주입

### 공통 Service

Service를 v1/v2 두 개로 나누지 않고 하나만 사용합니다.

```text
Service selector: app.kubernetes.io/part-of=gitops-version-dashboard
```

이 Selector는 두 버전의 Pod를 모두 Endpoint로 등록합니다. Istio `DestinationRule`이 Endpoint의 `version` 라벨을 사용해 v1과 v2 Subset으로 다시 구분합니다.

```text
Service:80 → targetPort:http → Container:8080
```

## Kustomize 구성

`overlays/prod/kustomization.yaml`은 다음 리소스를 조합합니다.

- `production` Namespace
- Base의 v1/v2 Deployment와 공통 Service
- Istio Gateway
- DestinationRule
- VirtualService
- PeerAuthentication

환경 파일에서 ConfigMap을 생성합니다.

| 파일 | 생성되는 ConfigMap | 용도 |
| --- | --- | --- |
| `stable.env` | `dashboard-config-stable` | v1 안정 버전 메타데이터 |
| `app.env` | `dashboard-config-canary` | v2 카나리 버전 메타데이터 |

`configMapGenerator`가 내용 Hash를 이름에 포함하므로 환경 파일이 변경되면 Deployment가 새 ConfigMap을 참조하도록 갱신됩니다.

v2 이미지는 다음 Kustomize 이미지 변환으로 적용합니다.

```yaml
images:
  - name: gitops-version-dashboard-canary
    newName: <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/gitops-version-dashboard
    newTag: <COMMIT_SHA_8>
```

## Argo CD Application

`argocd/application.yaml`은 다음 정책으로 이 저장소를 감시합니다.

- Repository: `gitops-version-dashboard-config`
- Branch: `master`
- Path: `overlays/prod`
- Destination Namespace: `production`
- Automated Sync 활성화
- `prune: true`: Git에서 제거된 리소스를 클러스터에서도 제거
- `selfHeal: true`: 수동 변경으로 발생한 Drift를 Git 상태로 복원
- `CreateNamespace=true`: Namespace 자동 생성 지원
- `PruneLast=true`: 다른 리소스 동기화 후 Prune 수행
- Revision History 10개 유지

Application 등록:

```bash
kubectl apply -f argocd/application.yaml
kubectl get application gitops-version-dashboard -n argocd
```

상태 확인:

```bash
kubectl get application gitops-version-dashboard -n argocd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
```

정상 상태는 `Synced`, `Healthy`입니다.

## Istio 설치

현재 매니페스트는 Istio `1.30.3`의 `networking.istio.io/v1`, `security.istio.io/v1` API를 사용합니다.

Master 노드에서 Helm으로 설치합니다.

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

설치 확인:

```bash
helm list -n istio-system
helm list -n istio-ingress
kubectl get pods -n istio-system
kubectl get pods,svc -n istio-ingress
```

## Istio 리소스

### Sidecar 자동 주입

`production` Namespace에 다음 라벨을 적용합니다.

```yaml
istio-injection: enabled
```

새로 생성되는 애플리케이션 Pod에는 `istio-proxy` 컨테이너가 자동으로 추가됩니다.

```bash
kubectl get pod -n production \
  -o custom-columns='NAME:.metadata.name,CONTAINERS:.spec.containers[*].name,READY:.status.containerStatuses[*].ready'
```

정상 Pod에는 `dashboard`, `istio-proxy` 두 컨테이너가 있습니다.

### Gateway

- 이름: `dashboard-gateway`
- Ingress Gateway Selector: `istio: ingress`
- HTTP Port: `80`
- Host: `*`
- Istio Ingress HTTP NodePort: `30081`
- 상태 확인 NodePort: Kubernetes가 자동 할당
- HTTPS `443` NodePort: Kubernetes가 자동 할당

현재 Gateway는 HTTP 실습 구성입니다. 실제 운영에서는 구체적인 Domain과 TLS 인증서를 적용해야 합니다.

### DestinationRule

하나의 Service FQDN을 Pod 라벨로 구분합니다.

```yaml
subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### VirtualService

초기 카나리 비율:

| 대상 | Weight |
| --- | ---: |
| v1 Stable | 90 |
| v2 Canary | 10 |

Weight는 Replica 개수와 무관하게 요청 단위 라우팅 비율을 의미합니다. 요청 수가 적으면 실제 관측값에 통계적 편차가 발생할 수 있습니다.

### STRICT mTLS

`overlays/prod/peer-authentication.yaml`이 `production` Namespace 전체에 다음 정책을 적용합니다.

```yaml
spec:
  mtls:
    mode: STRICT
```

- Sidecar가 없는 클라이언트의 평문 직접 요청 차단
- Istio Ingress Gateway에서 애플리케이션 Sidecar까지 mTLS 사용
- Istio가 발급하고 순환하는 Workload 인증서 사용
- 애플리케이션 코드에서 인증서를 직접 관리하지 않음

현재 DestinationRule에는 TLS Mode를 명시하지 않고 Istio Auto mTLS를 사용합니다. `STRICT`는 수신 측에서 mTLS를 강제하며, Mesh에 참여한 Gateway와 Sidecar는 자동으로 Istio 인증서를 사용합니다.

## 포트 구성

| 포트 | 계층 | 용도 |
| ---: | --- | --- |
| `8080` | 컨테이너 | Node.js 애플리케이션 |
| `80` | Dashboard Service | v1/v2 공통 내부 Service |
| `30080` | Dashboard NodePort | Istio 도입 전 Fallback 직접 접근 경로 |
| `80` | Istio Gateway Service | 외부 HTTP 수신 |
| `30081` | Istio Gateway NodePort | 정상 Dashboard 카나리 진입 경로 |
| `443` | Istio Gateway Service | 향후 HTTPS 수신 |
| `15021` | Istio Gateway | 상태 및 헬스체크 |

`STRICT mTLS` 상태에서 `30080`으로 평문 접근하면 연결이 초기화되는 것이 정상입니다. Dashboard는 `30081`을 통해 접근해야 합니다.

## 배포 순서

1. Terraform 저장소에서 AWS 인프라 생성
2. kubeadm으로 Master/Worker 클러스터 구성
3. Calico 설치 및 Node `Ready` 확인
4. Argo CD 설치
5. Istio Base, istiod, Ingress Gateway 설치
6. `production` Namespace와 Sidecar 주입 정책 확인
7. `argocd/application.yaml` 적용
8. Argo CD `Synced/Healthy` 확인
9. v1/v2 Pod가 각각 `2/2 Running`인지 확인
10. `30081`을 통해 90/10 카나리 라우팅 검증
11. `STRICT mTLS`로 `30080` 평문 직접 접근이 차단되는지 확인

## Kustomize 결과 검증

```bash
kubectl kustomize overlays/prod
```

클러스터 적용 전 가능하다면 다음 검사도 수행합니다.

```bash
istioctl analyze -n production
```

## 배포 상태 확인

```bash
kubectl get namespace production --show-labels
kubectl get deployment,pod,svc -n production -o wide
kubectl get gateway.networking.istio.io -n production
kubectl get virtualservice.networking.istio.io -n production
kubectl get destinationrule.networking.istio.io -n production
kubectl get peerauthentication.security.istio.io -n production
```

Endpoint가 두 버전으로 구성됐는지 확인:

```bash
kubectl get endpointslice -n production \
  -l kubernetes.io/service-name=gitops-version-dashboard \
  -o wide
```

## 카나리 트래픽 검증

Master 노드에서:

```bash
curl http://127.0.0.1:30081/api/info
```

100회 요청 결과를 이미지 태그별로 집계합니다.

```bash
for i in $(seq 1 100); do
  curl -s http://127.0.0.1:30081/api/info | jq -r '.imageTag'
done | sort | uniq -c
```

예상 결과는 대략 v1 90회, v2 10회입니다. 정확한 횟수는 매 실행마다 달라질 수 있습니다.

## 로컬 PC에서 Dashboard 접속

SSM으로 EC2의 Istio NodePort `30081`을 로컬 `8081`에 연결합니다.

```powershell
aws ssm start-session `
  --target <MASTER_INSTANCE_ID> `
  --document-name AWS-StartPortForwardingSession `
  --parameters "portNumber=30081,localPortNumber=8081" `
  --region ap-northeast-2
```

세션을 유지한 상태에서 다음 주소로 접속합니다.

- Dashboard: `http://localhost:8081`
- API: `http://localhost:8081/api/info`

## mTLS 검증

Gateway를 통한 요청은 성공해야 합니다.

```bash
curl -v http://127.0.0.1:30081/api/info
```

응답 Header의 `server: istio-envoy`, `x-envoy-upstream-service-time`으로 Envoy 경유 여부를 확인할 수 있습니다.

애플리케이션 NodePort로 직접 보내는 평문 요청은 실패해야 합니다.

```bash
curl -v http://127.0.0.1:30080/api/info
```

`Connection reset by peer`가 발생하면 `STRICT mTLS`가 평문 직접 요청을 거부하고 있는 것입니다.

## 카나리 승격과 Rollback

`overlays/prod/virtual-service.yaml`의 Weight를 Git으로 변경합니다.

| 단계 | v1 | v2 | 의미 |
| --- | ---: | ---: | --- |
| 초기 카나리 | 90 | 10 | 새 버전 소량 노출 |
| 확대 검증 | 50 | 50 | 트래픽 확대 |
| 전체 승격 | 0 | 100 | v2 전체 전환 |
| 긴급 Rollback | 100 | 0 | 안정 버전으로 즉시 복귀 |

```bash
git add overlays/prod/virtual-service.yaml
git commit -m "deploy: adjust canary traffic weight"
git push origin master
```

Argo CD는 이미지 재빌드 없이 VirtualService만 갱신합니다. 애플리케이션 오류가 발생하면 v2 Deployment를 삭제하기보다 Weight를 `100/0`으로 변경하여 빠르게 트래픽을 차단할 수 있습니다.

현재 승격은 수동입니다. Prometheus와 Argo Rollouts를 연동하면 오류율, 응답 시간 및 사용자 정의 지표에 따른 자동 승격과 Rollback을 구현할 수 있습니다.

## GitHub Actions와의 계약

애플리케이션 저장소의 CI는 이 저장소에서 다음 파일만 자동 수정합니다.

- `overlays/prod/kustomization.yaml`: v2 이미지 `newTag`
- `overlays/prod/app.env`: v2 애플리케이션과 Build 메타데이터

수정 후 GitHub Actions Bot이 다음 형식으로 Commit합니다.

```text
deploy: dashboard <IMAGE_TAG>
```

v1의 `deployment-v1.yaml`과 `stable.env`는 자동 변경 대상이 아닙니다.

## Fallback NodePort 제거

`service-patch.yaml`은 Istio 전환 과정에서 기존 접근 경로를 유지하기 위해 남겨둔 임시 설정입니다. Istio Gateway와 mTLS 검증이 완료되면 다음 작업을 권장합니다.

1. `overlays/prod/kustomization.yaml`의 `service-patch.yaml` Patch 제거
2. `overlays/prod/service-patch.yaml` 삭제
3. Terraform Security Group의 TCP `30080` 공개 규칙 제거
4. Argo CD Sync 후 Dashboard Service가 `ClusterIP`로 변경됐는지 확인

```bash
kubectl get svc gitops-version-dashboard -n production
```

외부 진입점은 Istio Ingress Gateway 하나로 제한하는 것이 구조와 보안 측면에서 명확합니다.

## 현재 범위와 개선 방향

- Gateway Host가 `*`이고 HTTP만 사용하므로 Domain과 HTTPS 구성이 필요합니다.
- 상시 외부 접속이 필요하면 NodePort 대신 AWS Load Balancer와 Route 53을 연동하는 것이 적절합니다.
- `30080` Fallback NodePort와 관련 Security Group 규칙을 제거해야 합니다.
- v1/v2라는 고정 이름보다 장기적으로 Stable/Canary 역할 기반 명명도 고려할 수 있습니다.
- 현재 Replica가 버전별 1개이므로 고가용성, HPA 및 PodDisruptionBudget을 추가할 수 있습니다.
- 네트워크 인증인 mTLS와 별도로 `AuthorizationPolicy`를 추가하면 Namespace와 Service Account 단위 접근 제어가 가능합니다.
- Prometheus, Grafana, Kiali 및 Argo Rollouts를 연동하면 카나리 관찰성과 자동화를 강화할 수 있습니다.
