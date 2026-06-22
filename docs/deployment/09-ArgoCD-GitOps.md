# Argo CD 기반 GitOps 배포 구성

## 1. 개요 및 목표

현재 MaeSoonGan 배포 구조에서 GitHub Actions는 Docker 이미지를 생성하고 ECR에 Push합니다.

향후 Argo CD를 도입하면 Kubernetes Manifest를 기준으로 EKS의 실제 상태를 자동 동기화할 수 있습니다.

```text
GitHub Actions
    ↓
Docker Image Build
    ↓
Amazon ECR Push
    ↓
Manifest의 Image Tag 변경
    ↓
Git Repository Commit
    ↓
Argo CD 변경 감지
    ↓
Amazon EKS 동기화
```

---

## 2. GitOps란

GitOps는 Git Repository를 배포 상태의 기준으로 사용하는 운영 방식입니다.

```text
Git에 정의된 상태
    =
클러스터가 유지해야 하는 상태
```

운영자가 클러스터에 직접 명령을 반복 실행하기보다, Git의 Manifest를 수정하고 Argo CD가 이를 적용합니다.

---

## 3. GitHub Actions와 Argo CD 역할 분리

### GitHub Actions

```text
Test
Gradle Build
Docker Build
Image Scan
ECR Push
Manifest Image Tag Update
```

### Argo CD

```text
Git 변경 감지
Manifest 렌더링
EKS 상태 비교
Sync
Drift 감지
배포 상태 시각화
```

### Kubernetes

```text
Pod 실행
Replica 유지
Health Check
Service 연결
Self-healing
```

---

## 4. 목표 아키텍처

```text
Developer
    ↓
Application Repository
    ↓
GitHub Actions
    ├── Test
    ├── Gradle Build
    ├── Docker Build
    └── ECR Push
            ↓
     Manifest Repository
            ↓
         Argo CD
            ↓
      Argo Rollouts
            ↓
         Amazon EKS
            ↓
       Kubernetes Service
            ↓
      ALB Ingress / Client
```

---

## 5. Repository 구성 방식

### 방식 1. 애플리케이션과 Manifest를 같은 Repository에서 관리

장점:

- 초기 구성이 단순함
- 코드와 Manifest 변경 추적이 쉬움

단점:

- 애플리케이션과 운영 권한 분리 어려움
- CI Commit으로 Workflow가 재실행될 수 있음

### 방식 2. Manifest 전용 Repository 분리

장점:

- 애플리케이션과 배포 권한 분리
- 환경별 승인 및 변경 이력 관리
- Argo CD가 배포 Repository만 감시

단점:

- Repository가 추가됨
- Image Tag 업데이트 연동 필요

운영 환경에서는 Manifest Repository 분리를 권장합니다.

---

## 6. 환경별 디렉터리 구조

Kustomize Overlay 예시:

```text
deploy
├── base
│   ├── admin-service
│   ├── auth-service
│   ├── contest-service
│   ├── market-service
│   └── order-service
│
└── overlays
    ├── dev
    ├── staging
    └── prod
```

환경별로 다음 값을 다르게 관리할 수 있습니다.

- Replica 수
- Image Tag
- Resource
- Domain
- ConfigMap
- Autoscaling
- Ingress
- 배포 전략

---

## 7. Argo CD 설치 개념

전용 Namespace:

```text
argocd
```

설치 후 주요 구성 요소:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-dex-server
argocd-redis
```

운영 환경에서는 설치 Manifest Version을 고정하고, 접근 방식과 인증 정책을 별도로 정의합니다.

---

## 8. Argo CD Application 예시

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: maesoongan-api
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/MaeSoonGan/{MANIFEST_REPOSITORY}.git
    targetRevision: main
    path: overlays/prod

  destination:
    server: https://kubernetes.default.svc
    namespace: app

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

---

## 9. Sync Policy

### `prune: true`

Git에서 삭제된 리소스를 Cluster에서도 삭제합니다.

운영 환경에서는 삭제 영향 범위를 충분히 검토해야 합니다.

### `selfHeal: true`

Cluster에서 수동 변경된 값을 Git 상태로 되돌립니다.

### `CreateNamespace=true`

대상 Namespace가 없으면 생성합니다.

---

## 10. Image Tag 업데이트 방식

### 방식 1. GitHub Actions가 Manifest 직접 수정

```text
ECR Push
    ↓
kustomization.yaml의 Image Tag 변경
    ↓
Commit & Push
    ↓
Argo CD Sync
```

### 방식 2. Argo CD Image Updater

Image Registry의 새 태그를 감지하여 자동으로 Manifest 또는 Argo CD Parameter를 업데이트합니다.

### 권장 기준

초기에는 GitHub Actions가 Commit SHA를 Manifest에 명시적으로 기록하는 방식이 이해와 추적이 쉽습니다.

---

## 11. Kustomize Image 변경 예시

```yaml
images:
  - name: {ECR_REGISTRY}/maesoongan/auth-service
    newName: {ECR_REGISTRY}/maesoongan/auth-service
    newTag: "{GITHUB_SHA}"
```

GitHub Actions에서 변경:

```bash
cd deploy/overlays/dev

kustomize edit set image \
  {ECR_REGISTRY}/maesoongan/auth-service=\
{ECR_REGISTRY}/maesoongan/auth-service:${GITHUB_SHA}
```

실제 Shell에서는 줄바꿈 없이 작성하거나 안전하게 변수를 구성합니다.

---

## 12. Sync 전 검증

다음 단계를 CI에 추가할 수 있습니다.

```text
YAML 문법 검사
Kustomize Build
Kubernetes Schema 검증
Policy 검사
보안 검사
```

예시:

```bash
kubectl kustomize deploy/overlays/dev
```

---

## 13. Argo Rollouts 도입

기본 Deployment 대신 Rollout 리소스를 사용해 Canary 또는 Blue-Green 배포를 구성할 수 있습니다.

### Canary 개념

```text
기존 Version 90%
신규 Version 10%
    ↓
지표 확인
    ↓
신규 Version 50%
    ↓
지표 확인
    ↓
신규 Version 100%
```

### Blue-Green 개념

```text
Blue: 현재 운영 Version
Green: 신규 Version
    ↓
검증
    ↓
Service 전환
```

---

## 14. Canary Rollout 예시

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: auth-service
  namespace: app

spec:
  replicas: 4

  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause:
            duration: 2m
        - setWeight: 50
        - pause:
            duration: 5m
        - setWeight: 100
```

실제 운영에서는 트래픽 라우팅 도구와 Metric 분석을 함께 구성합니다.

---

## 15. Rollback 전략

### Git Revert

Manifest Repository에서 문제 Commit을 Revert합니다.

```text
Git Revert
    ↓
Argo CD 변경 감지
    ↓
이전 Image Tag로 Sync
```

### Argo Rollouts Abort

Canary 진행 중 지표가 악화되면 배포를 중단할 수 있습니다.

### 고정 Image Tag

Rollback을 위해 기존 Commit SHA 또는 Version 태그를 ECR에 보존해야 합니다.

---

## 16. Drift 관리

Cluster에서 직접 Manifest를 변경하면 Git과 실제 상태가 달라집니다.

Argo CD는 이를 `OutOfSync`로 표시합니다.

운영 원칙:

```text
kubectl edit 금지
kubectl set image 금지
Git 변경 후 Argo CD Sync
```

긴급 조치 후에는 반드시 Git에도 동일한 변경을 반영합니다.

---

## 17. Secret 관리

GitOps Repository에는 평문 Secret을 저장하지 않습니다.

가능한 방식:

- External Secrets Operator
- AWS Secrets Manager
- Sealed Secrets
- SOPS
- Secrets Store CSI Driver

AWS 환경에서는 AWS Secrets Manager와 External Secrets Operator 조합을 우선 검토할 수 있습니다.

---

## 18. 접근 제어

Argo CD에는 다음 권한 구분을 적용할 수 있습니다.

```text
Viewer
Developer
Operator
Administrator
```

운영 환경에서는 다음 원칙을 권장합니다.

- 개발자는 조회와 제한된 Sync 권한
- 운영 담당자는 배포 승인 권한
- 관리자 권한 최소화
- SSO 연동
- Audit Log 확인

---

## 19. 알림

Argo CD Notifications를 통해 다음 이벤트를 알릴 수 있습니다.

```text
Sync 성공
Sync 실패
Health Degraded
Application OutOfSync
Rollout 실패
```

연동 대상:

- Slack
- Email
- Webhook

---

## 20. 모니터링 연동

Canary 자동 승격 또는 Rollback을 위해 Metric을 활용할 수 있습니다.

예시 지표:

```text
HTTP 5xx 비율
응답시간
Pod Restart
CPU 사용률
Memory 사용률
주문 실패율
인증 실패율
```

Prometheus, Grafana, CloudWatch 등과 연계할 수 있습니다.

---

## 21. 구축 순서

```text
1. Argo CD 설치
2. Argo CD 접근 및 인증 구성
3. Manifest Repository 연결
4. Application 생성
5. Manual Sync 검증
6. Automated Sync 활성화
7. Self Heal과 Prune 검토
8. GitHub Actions Image Tag 업데이트 연동
9. 환경별 Overlay 분리
10. Argo Rollouts 설치
11. Canary 또는 Blue-Green 적용
12. Metric 기반 분석 및 알림 연동
```

---

## 22. 운영 체크리스트

- [ ] Git이 배포 상태의 기준인가
- [ ] 운영 Cluster 직접 수정이 제한되는가
- [ ] Manifest에 고정 Image Tag를 사용하는가
- [ ] Sync 전 Manifest 검증이 수행되는가
- [ ] Prune 영향 범위를 검토했는가
- [ ] Secret이 Git에 평문으로 저장되지 않는가
- [ ] Rollback 절차가 문서화되어 있는가
- [ ] ECR에 Rollback용 이미지가 보존되는가
- [ ] Argo CD 권한이 최소화되어 있는가
- [ ] Sync 실패 및 Health 악화 알림이 구성되어 있는가
