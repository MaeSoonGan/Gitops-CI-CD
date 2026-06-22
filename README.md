# Gitops-CI-CD

## 1. 개요 및 프로젝트 내 역할

이 레포지토리는 MaeSoonGan 서비스의 GitOps 및 CI/CD 배포 흐름을 관리하기 위한 인프라 레포지토리입니다.

Backend 서비스에서 코드가 변경되면 GitHub Actions가 서비스별 Docker 이미지를 빌드하고 AWS ECR에 Push합니다. 이후 Kubernetes 매니페스트는 해당 이미지를 참조하여 AWS EKS 환경에 서비스를 배포합니다. 최종적으로는 Argo CD가 Git 변경사항을 감지하고 클러스터 상태를 자동으로 동기화하는 GitOps 구조를 목표로 합니다.

```text
개발자 코드 push
        ↓
GitHub Actions 실행
        ↓
서비스별 bootJar 빌드
        ↓
Docker 이미지 빌드
        ↓
AWS ECR 이미지 push
        ↓
Kubernetes 매니페스트에서 이미지 참조
        ↓
AWS EKS에 Pod 배포
        ↓
Argo CD가 Git 변경사항 감지 후 자동 동기화
```

이 레포지토리는 애플리케이션 코드가 아니라 배포 파이프라인, Kubernetes 매니페스트, Argo CD 연동 설정을 관리하는 역할을 합니다.

<br />

## 2. 구성 환경 및 사양

| 구분 | 구성 |
| --- | --- |
| CI | GitHub Actions |
| 인증 방식 | GitHub OIDC 기반 AWS IAM Role Assume |
| Container Registry | AWS ECR |
| Container Runtime | Docker |
| Orchestration | Kubernetes |
| Cluster | AWS EKS |
| GitOps | Argo CD |
| 배포 대상 Namespace | `app` |
| AWS Region | `ap-northeast-2` |
| AWS Account ID | `295234319852` |
| EKS Cluster | `maesoongan-cluster` |

### ECR Repository

```text
maesoongan/admin-service
maesoongan/auth-service
maesoongan/contest-service
maesoongan/market-service
maesoongan/order-service
```

서비스별 ECR Repository를 분리해 서비스 단위로 이미지 버전 추적, 재배포, 롤백을 할 수 있도록 구성했습니다.

### Backend 배포 대상 서비스

```text
admin-service
auth-service
contest-service
market-service
order-service
```

### 이미지 태그 전략

```text
latest
github.sha
```

예시:

```text
295234319852.dkr.ecr.ap-northeast-2.amazonaws.com/maesoongan/auth-service:latest
295234319852.dkr.ecr.ap-northeast-2.amazonaws.com/maesoongan/auth-service:{commit-sha}
```

`latest`는 최신 이미지 확인용으로 사용하고, `github.sha`는 어떤 커밋으로 빌드된 이미지인지 추적하거나 특정 커밋 이미지로 롤백하기 위해 사용합니다.

<br />

## 3. 선택 이유 및 대안 비교

### GitHub Actions

GitHub Repository와 바로 연동할 수 있고, push 이벤트 기반으로 빌드 및 배포 파이프라인을 자동화할 수 있어 선택했습니다.

대안으로 Jenkins를 사용할 수 있지만, Jenkins는 별도 서버 운영, 플러그인 관리, 인증 관리가 필요합니다. 현재 프로젝트에서는 GitHub Actions가 관리 부담이 적고 GitHub 브랜치 전략과 자연스럽게 연결됩니다.

| 대안 | 장점 | 단점 |
| --- | --- | --- |
| GitHub Actions | GitHub와 직접 연동, 설정 간단, OIDC 연동 쉬움 | GitHub 의존성 |
| Jenkins | 자유도 높음, 복잡한 파이프라인 구성 가능 | 별도 서버 운영과 관리 필요 |

### AWS ECR

EKS에서 사용할 Docker 이미지를 AWS 내부 Registry에 저장하기 위해 ECR을 사용했습니다.

```text
GitHub Repository → 코드 저장소
AWS ECR           → Docker 이미지 저장소
AWS EKS           → ECR 이미지를 가져와 Pod 실행
```

Docker Hub도 사용할 수 있지만, EKS와 같은 AWS 환경에서는 ECR이 IAM 권한 연동, 네트워크 구성, 이미지 접근 제어 측면에서 더 자연스럽습니다.

### OIDC 기반 IAM Role

AWS Access Key를 GitHub Secrets에 직접 저장하지 않기 위해 OIDC 방식을 사용했습니다.

| 방식 | 특징 |
| --- | --- |
| Access Key 방식 | 설정은 단순하지만 장기 키 유출 위험이 있음 |
| OIDC Role Assume 방식 | 임시 자격증명을 사용하므로 보안상 더 안전함 |

OIDC 방식은 GitHub Actions 실행 시 GitHub가 OIDC 토큰을 발급하고, AWS STS가 IAM Role Trust Policy를 검증한 뒤 임시 권한을 발급하는 구조입니다.

### Kubernetes + Argo CD

Kubernetes는 EKS에서 서비스별 Pod, Service, ConfigMap, Secret을 선언적으로 관리하기 위해 사용합니다.

Argo CD는 Git 저장소의 매니페스트 상태와 실제 클러스터 상태를 비교하고 자동 동기화하기 위해 사용합니다. 이를 통해 수동 `kubectl apply` 중심의 배포에서 Git 변경 기반의 GitOps 배포 방식으로 확장할 수 있습니다.

<br />

## 4. 설치 및 주요 설정

### GitHub Actions Workflow

Workflow 파일 위치:

```text
.github/workflows/build-and-push-ecr.yml
```

Workflow 역할:

```text
1. Repository 코드 checkout
2. GitHub OIDC로 AWS IAM Role assume
3. Amazon ECR 로그인
4. 서비스별 Docker 이미지 빌드
5. latest, github.sha 태그 부여
6. ECR에 이미지 push
```

실행 조건:

```yaml
on:
  push:
    branches:
      - develop
      - main
  workflow_dispatch:
```

빌드 대상 서비스 matrix:

```yaml
strategy:
  matrix:
    service:
      - admin-service
      - auth-service
      - contest-service
      - market-service
      - order-service
```

Docker build 구조:

```bash
docker build \
  -f apps/api/${{ matrix.service }}/Dockerfile \
  -t $ECR_REGISTRY/maesoongan/${{ matrix.service }}:${{ github.sha }} \
  -t $ECR_REGISTRY/maesoongan/${{ matrix.service }}:latest \
  .
```

Docker build context는 Repository root를 사용합니다.

```text
gradlew
gradle/
settings.gradle
build.gradle
apps/
```

### Dockerfile 구성

서비스별 Dockerfile 위치:

```text
apps/api/admin-service/Dockerfile
apps/api/auth-service/Dockerfile
apps/api/contest-service/Dockerfile
apps/api/market-service/Dockerfile
apps/api/order-service/Dockerfile
```

기본 구조:

```text
1단계: Gradle로 bootJar 생성
2단계: JRE 이미지에서 jar 실행
```

예시:

```dockerfile
FROM eclipse-temurin:17-jdk AS build

WORKDIR /app

COPY gradlew .
COPY gradle ./gradle
COPY settings.gradle .
COPY build.gradle .
COPY apps ./apps

RUN chmod +x gradlew
RUN ./gradlew :apps:api:auth-service:bootJar --no-daemon

FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=build /app/apps/api/auth-service/build/libs/*.jar app.jar

ENV AUTH_SERVICE_PORT=8081

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
```

서비스별로 달라지는 값:

```text
Gradle task 경로
서비스 포트
빌드 결과 jar 경로
```

예시:

```text
Gradle task: :apps:api:auth-service:bootJar
Port: 8081

Gradle task: :apps:api:order-service:bootJar
Port: 8084
```

### GitHub Secrets

GitHub Repository의 `Settings → Secrets and variables → Actions`에 등록합니다.

```text
AWS_REGION=ap-northeast-2
AWS_ROLE_ARN=arn:aws:iam::295234319852:role/MaesoonganGitHubActionsEcrRole
ECR_REGISTRY=295234319852.dkr.ecr.ap-northeast-2.amazonaws.com
```

주의사항:

```text
ECR_REGISTRY에는 repository 경로를 포함하지 않는다.

올바른 값:
295234319852.dkr.ecr.ap-northeast-2.amazonaws.com

잘못된 값:
295234319852.dkr.ecr.ap-northeast-2.amazonaws.com/maesoongan/auth-service
```

### IAM OIDC Provider

AWS Console에서 다음 값으로 OIDC Provider를 생성합니다.

```text
Provider type: OpenID Connect
Provider URL: https://token.actions.githubusercontent.com
Audience: sts.amazonaws.com
```

OIDC Audience는 반드시 `sts.amazonaws.com`이어야 합니다.

### IAM Role

Role 이름:

```text
MaesoonganGitHubActionsEcrRole
```

연결 정책:

```text
MaesoonganGitHubActionsEcrPushPolicy
```

Trust Policy 조건:

```text
GitHub Owner: MaeSoonGan
GitHub Repo: Back-End
허용 브랜치: develop, main
Audience: sts.amazonaws.com
```

허용 대상 예시:

```text
repo:MaeSoonGan/Back-End:ref:refs/heads/develop
repo:MaeSoonGan/Back-End:ref:refs/heads/main
```

### ECR Push용 IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EcrAuth",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "EcrPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeRepositories"
      ],
      "Resource": "arn:aws:ecr:ap-northeast-2:295234319852:repository/maesoongan/*"
    }
  ]
}
```

IAM Policy 작성 시 주의사항:

```text
JSON에는 주석을 넣지 않는다.
마지막 항목 뒤에 쉼표를 넣지 않는다.
작은따옴표가 아니라 큰따옴표를 사용한다.
{AWS_ACCOUNT_ID} 같은 placeholder는 실제 계정 ID로 변경한다.
```

### Kubernetes Manifest

API 매니페스트 위치:

```text
infra/k8s/api
```

주요 파일:

```text
namespace.yaml
configmap.yaml
secret.example.yaml
admin-service.yaml
auth-service.yaml
contest-service.yaml
market-service.yaml
order-service.yaml
kustomization.yaml
README.md
```

Namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
```

Kubernetes namespace와 ECR repository prefix는 서로 다릅니다.

```text
Kubernetes namespace: app
ECR repository prefix: maesoongan
```

이미지 예시:

```text
295234319852.dkr.ecr.ap-northeast-2.amazonaws.com/maesoongan/auth-service:latest
```

### ConfigMap / Secret 분리

ConfigMap에는 비밀이 아닌 설정값을 둡니다.

```text
서버 포트
Spring profile
Redis host 이름
공통 URL
```

Secret에는 외부에 노출되면 안 되는 값을 둡니다.

```text
DB username
DB password
JWT secret
Redis password
외부 API key
```

Secret 실제 값은 Git에 올리지 않고, 예시 파일만 관리합니다.

```text
secret.example.yaml
```

### Kustomize 적용

```bash
kubectl kustomize infra/k8s/api
kubectl apply -k infra/k8s/api
```

확인 명령어:

```bash
kubectl get pods -n app
kubectl get svc -n app
kubectl get pods -n app -o wide
```

<br />

## 5. 동작 흐름 및 트러블슈팅

### 최종 배포 흐름

```text
1. 개발자가 develop 또는 main 브랜치에 코드 push
2. GitHub Actions Workflow 실행
3. GitHub OIDC 토큰 발급
4. AWS STS가 IAM Role Trust Policy 검증
5. GitHub Actions가 임시 AWS 자격증명 획득
6. Amazon ECR 로그인
7. 서비스별 Docker 이미지 빌드
8. latest, github.sha 태그로 ECR push
9. Kubernetes 매니페스트에서 ECR 이미지 참조
10. EKS app namespace에 Pod 생성
11. Service가 Pod로 트래픽 전달
12. Argo CD가 Git 변경사항을 감지하여 클러스터와 동기화
```

### GitHub Actions 실행 방법

자동 실행:

```bash
git push origin develop
```

수동 실행:

```text
GitHub Repository
→ Actions
→ Build and Push Docker Images to ECR
→ Run workflow
```

실행 결과 확인:

```text
Checkout
Configure AWS credentials
Login to Amazon ECR
Build Docker image
Push Docker image
```

ECR 이미지 확인:

```text
AWS Console
→ ECR
→ Repositories
→ maesoongan/{service-name}
→ Images
```

### EKS kubeconfig 연결

필요 도구:

```text
AWS CLI
kubectl
AWS 계정 권한
```

계정 확인:

```bash
aws sts get-caller-identity
```

EKS 클러스터 목록 확인:

```bash
aws eks list-clusters --region ap-northeast-2
```

kubeconfig 업데이트:

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name maesoongan-cluster
```

현재 context 확인:

```bash
kubectl config current-context
```

노드 확인:

```bash
kubectl get nodes
```

### EKS API Endpoint 접근 문제

발생한 문제:

```text
publicAccess: false
privateAccess: true
```

원인:

```text
EKS API Endpoint가 private only로 설정되어 있으면 VPC 외부 로컬 PC에서 kubectl 접근이 불가능하다.
```

해결 방법:

```text
1. Bastion EC2, Cloud9, VPN 등 VPC 내부 환경에서 kubectl 실행
2. 임시로 Public endpoint를 활성화하고 내 IP/32만 허용
```

Public endpoint 활성화 예시:

```bash
aws eks update-cluster-config \
  --region ap-northeast-2 \
  --name maesoongan-cluster \
  --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true
```

Public endpoint 활성화는 접근 범위를 넓히는 작업이므로 팀 합의 후 진행해야 합니다.

### Kubernetes 적용 순서

Namespace 생성:

```bash
kubectl apply -f infra/k8s/api/namespace.yaml
kubectl get ns
```

Secret 예시 파일 복사 후 실제 값 입력:

```bash
cp infra/k8s/api/secret.example.yaml /tmp/maesoongan-api-secret.yaml
```

Windows PowerShell:

```powershell
Copy-Item infra/k8s/api/secret.example.yaml C:\Temp\maesoongan-api-secret.yaml
```

Secret 적용:

```bash
kubectl apply -f /tmp/maesoongan-api-secret.yaml
kubectl get secret -n app
```

Windows PowerShell:

```powershell
kubectl apply -f C:\Temp\maesoongan-api-secret.yaml
kubectl get secret -n app
```

API 매니페스트 적용:

```bash
kubectl apply -k infra/k8s/api
kubectl get pods -n app
kubectl get svc -n app
kubectl get pods -n app -o wide
```

### GitHub Actions 트러블슈팅

#### Incorrect token audience

원인:

```text
OIDC Provider 또는 Trust Policy의 audience 값이 GitHub Actions 토큰 audience와 맞지 않음
```

해결:

```text
OIDC Provider Audience를 sts.amazonaws.com으로 설정
Trust Policy의 aud 조건도 sts.amazonaws.com으로 설정
```

올바른 설정:

```json
"StringEquals": {
  "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
}
```

#### sigstore로 바꾸면 안 되는 이유

`sigstore`는 이미지 서명이나 provenance 용도로 사용하는 audience입니다. 현재 작업은 AWS STS를 통해 IAM Role을 assume하는 작업이므로 반드시 다음 값을 사용해야 합니다.

```text
sts.amazonaws.com
```

정리:

```text
ECR push용 AWS 인증 → sts.amazonaws.com
이미지 서명용 Sigstore → sigstore
```

#### Not authorized to perform sts:AssumeRoleWithWebIdentity

원인 후보:

```text
브랜치가 develop/main이 아님
GitHub Owner 이름이 틀림
GitHub Repo 이름이 틀림
AWS_ROLE_ARN이 다른 Role을 가리킴
OIDC Provider ARN 계정 ID가 틀림
Trust Policy의 sub 조건이 틀림
```

확인할 값:

```text
repo:MaeSoonGan/Back-End:ref:refs/heads/develop
```

#### gradlew exit code 126

원인:

```text
Docker build 컨테이너 안에서 gradlew 실행 권한이 없음
```

해결:

```dockerfile
RUN chmod +x gradlew
```

#### Gradle 멀티프로젝트 디렉터리 오류

에러:

```text
Configuring project ':apps:worker' without an existing directory is not allowed.
```

원인:

```text
settings.gradle에는 여러 subproject가 포함되어 있는데 Dockerfile에서 특정 서비스 디렉터리만 복사해서 Gradle 설정 단계에서 실패
```

해결:

```dockerfile
COPY apps ./apps
```

### Kubernetes 트러블슈팅

#### Pod 목록 확인

```bash
kubectl get pods -n app
```

#### Pod 상세 확인

```bash
kubectl describe pod -n app {pod-name}
```

#### 로그 확인

```bash
kubectl logs -n app {pod-name}
kubectl logs -n app {pod-name} --previous
```

#### Service 확인

```bash
kubectl get svc -n app
```

#### Endpoint 확인

```bash
kubectl get endpoints -n app
```

Endpoint가 비어 있으면 다음을 확인합니다.

```text
Pod label과 Service selector가 일치하는지 확인
Pod가 Ready 상태인지 확인
readinessProbe 실패 여부 확인
```

#### Node label 확인

```bash
kubectl get nodes --show-labels
```

nodeSelector가 필요한 경우:

```bash
kubectl label node {node-name} role=service
```

### 자주 발생하는 Pod 상태

#### ImagePullBackOff

원인:

```text
ECR 이미지 경로 오류
ECR에 이미지 없음
EKS Node의 ECR pull 권한 없음
이미지 태그 없음
```

확인:

```bash
kubectl describe pod -n app {pod-name}
```

#### CrashLoopBackOff

원인:

```text
Spring Boot 실행 실패
환경변수 누락
DB 연결 실패
Redis 연결 실패
JWT secret 누락
application 설정 오류
```

확인:

```bash
kubectl logs -n app {pod-name}
```

#### Pending

원인:

```text
nodeSelector 조건에 맞는 노드 없음
리소스 부족
Node Group 준비 안 됨
taint/toleration 문제
```

확인:

```bash
kubectl describe pod -n app {pod-name}
```

#### CreateContainerConfigError

원인:

```text
Secret 없음
ConfigMap 없음
Secret key 이름 불일치
환경변수 참조 실패
```

확인:

```bash
kubectl get secret -n app
kubectl get configmap -n app
kubectl describe pod -n app {pod-name}
```

### 구축 작업 요약

```text
ECR 리포지토리 구조 정리
서비스별 Dockerfile 작성
GitHub Actions workflow 작성
OIDC 기반 IAM Role 구성
GitHub Secrets 등록
ECR push workflow 실행
Docker build 권한 문제 해결
Gradle 멀티프로젝트 복사 문제 해결
Kubernetes API 매니페스트 작성
namespace app으로 변경
EKS kubeconfig 연결
EKS private endpoint 문제 확인
```
