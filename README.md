# ArgoCD Applications

GitOps 기반 Kubernetes 애플리케이션 배포 관리

## 구조

```
argocd-app/
├── root-apps/                    # Root Application (App of Apps)
│   └── devops/
│       ├── default/              # Default 워크스페이스
│       │   └── common-root.yaml
│       └── green/                # Green 워크스페이스
│           └── common-root.yaml
│
├── common-apps/                  # Application 정의
│   ├── default/                  # Default 워크스페이스
│   │   ├── istio-base.yaml      # Istio CRDs
│   │   ├── istiod.yaml          # Istio Control Plane
│   │   ├── istio-gateway.yaml   # Istio Ingress Gateway
│   │   ├── karpenter-config.yaml
│   │   ├── simple-web.yaml
│   │   └── values/              # 환경별 values
│   │       ├── istio-base.yaml
│   │       ├── istiod.yaml
│   │       ├── istio-gateway.yaml
│   │       ├── karpenter-config.yaml
│   │       └── simple-web.yaml
│   └── green/
│       ├── simple-web.yaml
│       └── values/
│           └── simple-web.yaml
│
└── charts/                       # 공통 Helm Charts
    ├── karpenter-config/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    │       ├── ec2nodeclass.yaml
    │       ├── nodepool-spot.yaml
    │       └── nodepool-ondemand.yaml
    └── simple-web/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── ingress.yaml
```

## 배포 방법

### 1. Root Application 배포 (App of Apps)

```bash
# Default 워크스페이스
kubectl apply -f root-apps/devops/default/common-root.yaml

# Green 워크스페이스
kubectl apply -f root-apps/devops/green/common-root.yaml
```

Root Application이 배포되면 ArgoCD가 자동으로 `common-apps/` 하위의 모든 애플리케이션을 배포합니다.

### 2. 개별 애플리케이션 배포

필요시 개별 애플리케이션만 배포 가능:

```bash
kubectl apply -f common-apps/default/istio-base.yaml
kubectl apply -f common-apps/default/istiod.yaml
kubectl apply -f common-apps/default/istio-gateway.yaml
kubectl apply -f common-apps/default/karpenter-config.yaml
kubectl apply -f common-apps/default/simple-web.yaml
```

## 배포된 애플리케이션

### Istio Service Mesh

**설치 순서:** istio-base → istiod → istio-gateway

#### 1. istio-base (CRDs)
- **용도**: Istio Custom Resource Definitions
- **Namespace**: istio-system
- **Sync Wave**: 1 (가장 먼저 배포)

#### 2. istiod (Control Plane)
- **용도**: Istio 컨트롤 플레인 (Pilot, Citadel, Galley)
- **Namespace**: istio-system
- **Sync Wave**: 2
- **주요 기능**:
  - 서비스 디스커버리
  - 트래픽 관리 및 라우팅
  - mTLS 인증서 관리
  - Sidecar 자동 주입

#### 3. istio-gateway (Ingress Gateway)
- **용도**: 외부 트래픽을 클러스터로 라우팅
- **Namespace**: istio-ingress
- **Sync Wave**: 3
- **Service Type**: LoadBalancer (AWS NLB)
- **Ports**: 80 (HTTP), 443 (HTTPS)

### Karpenter Configuration

- NodePool (Spot/OnDemand)
- EC2NodeClass

### Simple Web

샘플 웹 애플리케이션

## Istio 사용 방법

### 1. Namespace에 Sidecar 자동 주입 활성화

```bash
# Namespace 레이블 추가
kubectl label namespace default istio-injection=enabled

# 확인
kubectl get namespace -L istio-injection
```

### 2. Gateway 및 VirtualService 생성 예제

```yaml
# gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  selector:
    istio: gateway  # istio-gateway의 label
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "example.com"
---
# virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
  namespace: default
spec:
  hosts:
  - "example.com"
  gateways:
  - my-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: my-service
        port:
          number: 80
```

### 3. Istio Gateway LoadBalancer 주소 확인

```bash
kubectl get svc -n istio-ingress
```

## App of Apps 패턴

이 프로젝트는 **App of Apps** 패턴을 사용합니다:

1. **Root Application** (`common-root.yaml`)이 `common-apps/` 디렉토리를 모니터링
2. Root Application이 하위 모든 Application을 자동으로 배포
3. 각 Application은 독립적으로 관리되며 자동 sync/prune/selfHeal 가능

### 장점

- 중앙 집중식 관리
- 선언적 배포
- 환경별 분리 (default/green)
- GitOps 워크플로우

## Sync Wave를 통한 배포 순서 제어

Istio는 의존성이 있으므로 Sync Wave로 순서를 제어합니다:

```
Wave 1: istio-base (CRDs)
Wave 2: istiod (Control Plane)
Wave 3: istio-gateway (Data Plane)
```

## 모니터링

### ArgoCD UI에서 확인

```bash
# ArgoCD UI 접속 (포트포워딩)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 브라우저에서 접속
open https://localhost:8080
```

### CLI로 확인

```bash
# Application 목록
kubectl get applications -n argocd

# 특정 Application 상태 확인
kubectl get application istio-base -n argocd -o yaml
```

## 트러블슈팅

### Istio 설치 순서 문제

Istio는 반드시 순서대로 설치되어야 합니다. Sync Wave가 제대로 동작하지 않으면:

```bash
# 수동으로 순차 배포
kubectl apply -f common-apps/default/istio-base.yaml
# 설치 완료 대기
kubectl wait --for=condition=available --timeout=300s deployment/istiod -n istio-system

kubectl apply -f common-apps/default/istiod.yaml
# 설치 완료 대기
kubectl wait --for=condition=available --timeout=300s deployment/istiod -n istio-system

kubectl apply -f common-apps/default/istio-gateway.yaml
```

### Sidecar 주입 안됨

```bash
# Namespace 레이블 확인
kubectl get namespace default -o yaml | grep istio-injection

# Pod 재시작
kubectl rollout restart deployment/my-app -n default
```

### Gateway LoadBalancer 생성 안됨

```bash
# Service 상태 확인
kubectl describe svc istio-gateway -n istio-ingress

# AWS Load Balancer Controller 로그 확인
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

## 참고 자료

- [Istio Documentation](https://istio.io/latest/docs/)
- [ArgoCD App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [Istio on EKS Best Practices](https://aws.github.io/aws-eks-best-practices/networking/servicemesh/istio/)