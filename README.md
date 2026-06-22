# 🚀 MaeSoonGan Deployment

## ✍️ 프로젝트 한 줄 소개

MaeSoonGan 백엔드 서비스를 안정적으로 빌드·배포·운영하기 위한 AWS EKS 기반 컨테이너 배포 환경과 CI/CD 파이프라인입니다.

---

## 🛫 레포지토리 개요

이 레포지토리는 MaeSoonGan 백엔드 서비스의 배포에 필요한 설정과 문서를 관리합니다.

개발자가 GitHub에 코드를 Push하면 GitHub Actions가 각 Spring Boot 서비스를 빌드하고 Docker 이미지를 생성합니다. 생성된 이미지는 Amazon ECR에 저장되며, Kubernetes Manifest를 통해 Amazon EKS에 배포됩니다.

현재는 GitHub Actions 기반 이미지 빌드와 ECR Push, Kubernetes Manifest 작성, EKS 연결까지 구성되어 있습니다. 향후 Argo CD와 Argo Rollouts를 연동하여 GitOps 기반 자동 배포와 무중단 배포 환경으로 확장할 예정입니다.

---

## 🎯 배포 목표

- 코드 Push부터 Docker 이미지 생성까지 자동화
- 서비스별 독립 빌드 및 배포
- Commit SHA 기반 이미지 버전 추적
- GitHub OIDC 기반 AWS 인증
- Kubernetes를 통한 실행 상태 관리
- 향후 Argo CD 기반 GitOps 자동 배포
- Argo Rollouts 기반 Canary 또는 Blue-Green 배포
- ALB, ACM, Route 53을 통한 외부 서비스 연결

---

## 🖥️ 배포 환경 및 서버 사양

| 구분 | 구성 |
|---|---|
| Cloud Provider | AWS |
| Container Registry | Amazon ECR |
| Container Orchestration | Amazon EKS |
| CI | GitHub Actions |
| CD | Kubernetes 수동 적용, Argo CD 예정 |
| Application | Spring Boot 3 |
| Build Tool | Gradle |
| Runtime | Java 17 |
| Container | Docker |
| Kubernetes Namespace | `app` |
| AWS Region | `ap-northeast-2` |
| Image Tag | `latest`, GitHub Commit SHA |

서버와 Node Group의 세부 사양은 실제 EKS 구성에 맞춰 별도 문서에 기록합니다.

---

## 🧩 구성 요소

### GitHub Actions

- Repository 코드 Checkout
- GitHub OIDC 기반 AWS 인증
- 서비스별 Docker 이미지 빌드
- Commit SHA 및 `latest` 태그 생성
- Amazon ECR 이미지 Push

### Amazon ECR

- 서비스별 Docker 이미지 저장
- 이미지 태그 및 배포 버전 관리
- EKS에서 사용할 이미지 제공

### Kubernetes

- Deployment와 Service 정의
- ConfigMap과 Secret 주입
- Readiness/Liveness Probe 설정
- NodeSelector 기반 Pod 배치
- Kustomize 기반 Manifest 관리

### Amazon EKS

- MaeSoonGan 백엔드 Pod 실행
- Replica 및 상태 관리
- Service 기반 내부 통신
- 향후 Ingress와 ALB 연결

### Argo CD 및 Argo Rollouts

- Git Repository와 EKS 상태 동기화
- Manifest 변경 자동 반영
- Canary 또는 Blue-Green 배포
- 배포 상태 및 Rollback 관리

---

## 🏗️ 전체 배포 아키텍처

```text
┌──────────────────────────────┐
│          Developer           │
│                              │
│     Code Commit & Push       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│           GitHub             │
│                              │
│      develop / main          │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       GitHub Actions         │
│                              │
│  OIDC Authentication         │
│  Gradle / Docker Build       │
│  Image Tagging               │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        Amazon ECR            │
│                              │
│  Service Docker Images       │
│  latest / Commit SHA         │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        Amazon EKS            │
│                              │
│  Deployment / Service        │
│  ConfigMap / Secret          │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      MaeSoonGan Services     │
│                              │
│  admin / auth / contest      │
│  market / order              │
└──────────────────────────────┘
```

향후 구조는 다음과 같이 확장합니다.

```text
GitHub Actions
    ↓
Amazon ECR
    ↓
Manifest Repository
    ↓
Argo CD
    ↓
Argo Rollouts
    ↓
Amazon EKS
    ↓
ALB Ingress
    ↓
Route 53 / Client
```

---

## 🔁 CI/CD 및 배포 흐름

```text
개발자 코드 Push
    ↓
GitHub Actions
    ↓
Gradle bootJar
    ↓
Docker Image Build
    ↓
AWS ECR Push
    ↓
Kubernetes Manifest 적용
    ↓
AWS EKS 배포
    ↓
Service / Ingress를 통한 요청 전달
```

현재 GitHub Actions는 Docker 이미지 생성과 ECR Push까지 담당합니다.

Kubernetes Manifest는 `kubectl apply -k`로 적용하며, 향후 Argo CD가 Git 변경사항을 감지하여 EKS 상태를 자동으로 동기화하도록 변경할 예정입니다.

---

## 📦 서비스별 배포 대상

| 서비스 | 역할 | ECR Repository |
|---|---|---|
| `admin-service` | 관리자 기능 | `maesoongan/admin-service` |
| `auth-service` | 사용자 인증 및 인가 | `maesoongan/auth-service` |
| `contest-service` | 대회 및 참가 관리 | `maesoongan/contest-service` |
| `market-service` | 시장 데이터 관리 | `maesoongan/market-service` |
| `order-service` | 주문 처리 | `maesoongan/order-service` |

서비스별 Dockerfile은 다음 위치에서 관리합니다.

```text
apps/api/admin-service/Dockerfile
apps/api/auth-service/Dockerfile
apps/api/contest-service/Dockerfile
apps/api/market-service/Dockerfile
apps/api/order-service/Dockerfile
```

---

## 🌐 네트워크 및 접근 구조

### Kubernetes 내부 통신

각 서비스는 Pod IP가 아니라 Kubernetes Service 이름을 통해 통신합니다.

```text
다른 서비스
    ↓
Kubernetes Service
    ↓
Ready 상태의 Pod
```

같은 Namespace에서는 다음 형식으로 접근합니다.

```text
http://{service-name}:{port}
```

### EKS API 접근

현재 EKS API Endpoint는 Private Access 중심으로 구성되어 있습니다.

```text
publicAccess: false
privateAccess: true
```

따라서 다음 방식 중 하나로 접근해야 합니다.

- 같은 VPC의 Bastion EC2
- VPN 연결 환경
- VPC 내부 관리 서버
- 제한된 IP만 허용한 Public Endpoint 임시 활성화

### 외부 트래픽

향후 다음 구조를 구성합니다.

```text
Client
    ↓
Route 53
    ↓
AWS ALB
    ↓
Ingress
    ↓
Kubernetes Service
    ↓
Application Pod
```

---

## 🔐 인증 및 보안 구성

### GitHub OIDC

GitHub Actions는 AWS Access Key를 장기 저장하지 않고 OIDC를 통해 IAM Role을 Assume합니다.

```text
GitHub Actions
    ↓
OIDC Token
    ↓
AWS STS
    ↓
IAM Trust Policy 검증
    ↓
임시 자격증명
    ↓
Amazon ECR 접근
```

### 접근 제한

IAM Trust Policy에서 다음 항목을 제한합니다.

- GitHub Owner
- Repository
- Branch
- OIDC Audience

허용 Branch:

```text
develop
main
```

OIDC Audience:

```text
sts.amazonaws.com
```

### Secret 관리

민감정보는 Repository에 직접 저장하지 않습니다.

```text
DB Password
JWT Secret
Redis Password
외부 API Key
```

실제 값은 Kubernetes Secret 또는 향후 AWS Secrets Manager를 통해 관리합니다.

---

## 📂 디렉터리 구조

```text
.
├── README.md
│
├── .github
│   └── workflows
│       └── build-and-push-ecr.yml
│
├── apps
│   └── api
│       ├── admin-service
│       │   └── Dockerfile
│       ├── auth-service
│       │   └── Dockerfile
│       ├── contest-service
│       │   └── Dockerfile
│       ├── market-service
│       │   └── Dockerfile
│       └── order-service
│           └── Dockerfile
│
├── infra
│   └── k8s
│       └── api
│           ├── namespace.yaml
│           ├── configmap.yaml
│           ├── secret.example.yaml
│           ├── admin-service.yaml
│           ├── auth-service.yaml
│           ├── contest-service.yaml
│           ├── market-service.yaml
│           ├── order-service.yaml
│           └── kustomization.yaml
│
└── docs
    └── deployment
        ├── 01-GitHub-Actions-CI.md
        ├── 02-Docker-Image.md
        ├── 03-AWS-ECR.md
        ├── 04-GitHub-OIDC-IAM.md
        ├── 05-Kubernetes-Application.md
        ├── 06-AWS-EKS-Deployment.md
        ├── 07-Secret-Management.md
        ├── 08-Troubleshooting.md
        └── 09-ArgoCD-GitOps.md
```

---

## 📋 배포 문서 목차

| 번호 | 문서 | 주요 내용 |
|---:|---|---|
| 01 | [GitHub Actions CI 파이프라인](./docs/deployment/01-GitHub-Actions-CI.md) | Gradle 빌드, Docker 이미지 생성, ECR Push |
| 02 | [서비스별 Docker 이미지 구성](./docs/deployment/02-Docker-Image.md) | Dockerfile, Multi-stage Build, 서비스별 설정 |
| 03 | [AWS ECR 이미지 저장소 구성](./docs/deployment/03-AWS-ECR.md) | Repository, 이미지 태그, 수명 주기 |
| 04 | [GitHub OIDC 및 IAM 구성](./docs/deployment/04-GitHub-OIDC-IAM.md) | GitHub Actions와 AWS 인증 |
| 05 | [Kubernetes 애플리케이션 배포](./docs/deployment/05-Kubernetes-Application.md) | Deployment, Service, ConfigMap, Secret |
| 06 | [AWS EKS 클러스터 연결 및 배포](./docs/deployment/06-AWS-EKS-Deployment.md) | kubeconfig, Node, 실제 배포 |
| 07 | [Secret 및 환경변수 관리](./docs/deployment/07-Secret-Management.md) | 환경변수와 민감정보 관리 |
| 08 | [배포 운영 및 트러블슈팅](./docs/deployment/08-Troubleshooting.md) | 배포 오류 분석과 해결 |
| 09 | [Argo CD 기반 GitOps](./docs/deployment/09-ArgoCD-GitOps.md) | 자동 배포와 Rollout 구성 |

---
