# GitHub Actions CI 파이프라인 구성

## 1. 개요 및 프로젝트 내 역할

MaeSoonGan 백엔드 프로젝트는 여러 Spring Boot 서비스로 구성되어 있습니다. GitHub Actions는 개발자가 `develop` 또는 `main` 브랜치에 코드를 Push했을 때 각 서비스의 실행 파일과 Docker 이미지를 생성하고, Amazon ECR에 업로드하는 역할을 담당합니다.

현재 파이프라인의 책임 범위는 다음과 같습니다.

```text
GitHub Push
    ↓
Repository Checkout
    ↓
GitHub OIDC 기반 AWS 인증
    ↓
서비스별 Docker 이미지 빌드
    ↓
이미지 태그 생성
    ↓
Amazon ECR Push
```

GitHub Actions는 CI를 담당하고, 실제 Kubernetes 배포 상태 관리는 향후 Argo CD가 담당하도록 역할을 분리합니다.

---

## 2. Workflow 파일 위치

```text
.github/workflows/build-and-push-ecr.yml
```

Workflow 파일은 GitHub가 자동으로 인식할 수 있도록 `.github/workflows` 디렉터리 아래에 배치합니다.

---

## 3. Workflow 실행 조건

### 3.1 브랜치 Push 실행

```yaml
on:
  push:
    branches:
      - develop
      - main
```

`develop`과 `main` 브랜치에 코드가 Push되면 Workflow가 자동 실행됩니다.

### 3.2 수동 실행

```yaml
on:
  workflow_dispatch:
```

`workflow_dispatch`를 사용하면 GitHub Actions 화면에서 수동으로 Workflow를 실행할 수 있습니다.

수동 실행은 다음 상황에서 유용합니다.

- Workflow 설정 검증
- 특정 서비스 이미지 재빌드
- ECR Push 재시도
- 운영 배포 전 사전 점검

---

## 4. 권한 설정

GitHub Actions가 OIDC Token을 발급하고 Repository 코드를 읽으려면 다음 권한이 필요합니다.

```yaml
permissions:
  id-token: write
  contents: read
```

| 권한 | 역할 |
|---|---|
| `id-token: write` | GitHub OIDC Token 발급 |
| `contents: read` | Repository 코드 Checkout |

AWS Access Key를 사용하지 않으므로 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`를 GitHub Secrets에 저장하지 않습니다.

---

## 5. 서비스별 Matrix Build 구성

여러 서비스를 동일한 Workflow 구조로 빌드하기 위해 Matrix 전략을 사용합니다.

```yaml
strategy:
  fail-fast: false
  matrix:
    service:
      - admin-service
      - auth-service
      - contest-service
      - market-service
      - order-service
```

Matrix를 사용하면 각 서비스별 Job이 독립적으로 실행됩니다.

```text
admin-service Job
auth-service Job
contest-service Job
market-service Job
order-service Job
```

### `fail-fast: false`

한 서비스의 빌드가 실패하더라도 다른 서비스의 빌드를 계속 실행합니다.

이를 통해 특정 서비스의 문제 때문에 전체 빌드 결과를 확인하지 못하는 상황을 방지합니다.

---

## 6. Workflow 주요 단계

### 6.1 Repository Checkout

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```

Workflow Runner가 Repository 코드를 내려받습니다.

### 6.2 AWS 자격증명 구성

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ${{ secrets.AWS_REGION }}
```

GitHub OIDC Token을 이용해 AWS IAM Role을 Assume하고 임시 자격증명을 발급받습니다.

### 6.3 Amazon ECR 로그인

```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v2
```

발급받은 AWS 임시 자격증명을 이용해 ECR에 로그인합니다.

### 6.4 Docker 이미지 빌드

```yaml
- name: Build Docker image
  run: |
    docker build \
      -f apps/api/${{ matrix.service }}/Dockerfile \
      -t ${{ secrets.ECR_REGISTRY }}/maesoongan/${{ matrix.service }}:${{ github.sha }} \
      -t ${{ secrets.ECR_REGISTRY }}/maesoongan/${{ matrix.service }}:latest \
      .
```

Repository Root를 Docker Build Context로 사용합니다.

마지막의 `.`은 다음 파일과 디렉터리를 Docker Build Context에 포함한다는 의미입니다.

```text
gradlew
gradle/
settings.gradle
build.gradle
apps/
```

### 6.5 ECR Push

```yaml
- name: Push Docker image
  run: |
    docker push \
      ${{ secrets.ECR_REGISTRY }}/maesoongan/${{ matrix.service }}:${{ github.sha }}

    docker push \
      ${{ secrets.ECR_REGISTRY }}/maesoongan/${{ matrix.service }}:latest
```

각 이미지는 Commit SHA와 `latest` 두 가지 태그로 Push합니다.

---

## 7. 전체 Workflow 예시

```yaml
name: Build and Push Docker Images to ECR

on:
  push:
    branches:
      - develop
      - main
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        service:
          - admin-service
          - auth-service
          - contest-service
          - market-service
          - order-service

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker build \
            -f apps/api/${{ matrix.service }}/Dockerfile \
            -t ${{ secrets.ECR_REGISTRY }}/maesoongan/${{ matrix.service }}:${{ github.sha }} \
            -t ${{ secrets.ECR_REGISTRY }}/maesoongan/${{ matrix.service }}:latest \
            .

      - name: Push Docker image
        run: |
          docker push \
            ${{ secrets.ECR_REGISTRY }}/maesoongan/${{ matrix.service }}:${{ github.sha }}

          docker push \
            ${{ secrets.ECR_REGISTRY }}/maesoongan/${{ matrix.service }}:latest
```

---

## 8. GitHub Secrets 구성

Repository에서 다음 경로로 이동합니다.

```text
Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

등록할 Secret은 다음과 같습니다.

| Secret 이름 | 예시 | 설명 |
|---|---|---|
| `AWS_REGION` | `ap-northeast-2` | AWS Region |
| `AWS_ROLE_ARN` | `arn:aws:iam::{AWS_ACCOUNT_ID}:role/MaesoonganGitHubActionsEcrRole` | Assume할 IAM Role |
| `ECR_REGISTRY` | `{AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com` | ECR Registry 주소 |

### 잘못된 등록 방식

```text
Name: AWS_REGION=ap-northeast-2
```

### 올바른 등록 방식

```text
Name: AWS_REGION
Value: ap-northeast-2
```

---

## 9. 이미지 태그 전략

### 9.1 `latest`

```text
{ECR_REGISTRY}/maesoongan/auth-service:latest
```

가장 최근에 성공적으로 빌드된 이미지를 가리킵니다.

### 9.2 Commit SHA

```text
{ECR_REGISTRY}/maesoongan/auth-service:{GITHUB_SHA}
```

어떤 Git Commit으로 생성된 이미지인지 추적할 수 있습니다.

Commit SHA 태그의 장점은 다음과 같습니다.

- 배포 버전 식별
- 장애 발생 시 이전 이미지로 롤백
- 빌드 결과 재현
- 배포 이력 추적

운영 환경의 Kubernetes Manifest에서는 가능한 한 Commit SHA와 같은 고정 태그를 사용합니다.

---

## 10. Workflow 실행 방법

### 자동 실행

```bash
git add .
git commit -m "ADD: ECR image build workflow"
git push origin develop
```

### 수동 실행

```text
GitHub Repository
→ Actions
→ Build and Push Docker Images to ECR
→ Run workflow
```

---

## 11. 실행 결과 확인

GitHub Actions에서 다음 단계가 모두 성공했는지 확인합니다.

```text
Checkout repository
Configure AWS credentials
Login to Amazon ECR
Build Docker image
Push Docker image
```

ECR에서는 다음 경로에서 이미지를 확인합니다.

```text
AWS Console
→ ECR
→ Repositories
→ maesoongan/{service-name}
→ Images
```

확인할 태그는 다음과 같습니다.

```text
latest
GitHub Commit SHA
```

---

## 12. 트러블슈팅

### 12.1 `Incorrect token audience`

```text
Could not assume role with OIDC: Incorrect token audience
```

원인은 AWS IAM OIDC Provider 또는 Trust Policy의 Audience가 GitHub Actions Token과 일치하지 않는 것입니다.

```text
sts.amazonaws.com
```

ECR Push를 위한 AWS STS 인증에서는 반드시 위 값을 사용합니다.

### 12.2 `Not authorized to perform sts:AssumeRoleWithWebIdentity`

다음 항목을 확인합니다.

- GitHub Owner
- Repository 이름
- 실행 Branch
- AWS Role ARN
- OIDC Provider ARN
- AWS Account ID
- Trust Policy의 `sub` 조건

예시:

```text
repo:MaeSoonGan/Back-End:ref:refs/heads/develop
```

### 12.3 `gradlew exit code 126`

Docker Build 환경에서 `gradlew` 실행 권한이 없을 때 발생합니다.

```dockerfile
RUN chmod +x gradlew
```

### 12.4 Gradle 하위 프로젝트 디렉터리 오류

```text
Configuring project ':apps:worker' without an existing directory is not allowed.
```

`settings.gradle`에 등록된 하위 프로젝트가 Docker Build Context에 포함되지 않았을 때 발생합니다.

```dockerfile
COPY apps ./apps
```

### 12.5 ECR Registry 경로 오류

`ECR_REGISTRY`에는 Repository 경로가 아니라 Registry 주소까지만 저장합니다.

잘못된 값:

```text
{AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/maesoongan/auth-service
```

올바른 값:

```text
{AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com
```

---

## 13. 운영 개선 사항

향후 다음 항목을 적용할 수 있습니다.

- 변경된 서비스만 선택적으로 빌드
- 테스트 성공 후에만 이미지 빌드
- Docker Build Cache 적용
- 이미지 취약점 스캔
- ECR Lifecycle Policy 적용
- Manifest Repository의 이미지 태그 자동 갱신
- Argo CD 기반 자동 배포
- Argo Rollouts 기반 Canary 배포
