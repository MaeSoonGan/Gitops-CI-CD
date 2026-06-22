# AWS ECR 이미지 저장소 구성

## 1. 개요 및 프로젝트 내 역할

Amazon ECR은 MaeSoonGan 백엔드 서비스의 Docker 이미지를 저장하는 Private Container Registry입니다.

```text
GitHub Actions
    ↓
Docker Image Build
    ↓
Amazon ECR Push
    ↓
Amazon EKS가 Image Pull
    ↓
Kubernetes Pod 실행
```

---

## 2. ECR 선택 이유

### 주요 이유

- AWS IAM과 자연스럽게 연동
- EKS Worker Node에서 이미지 Pull이 쉬움
- Private Repository 지원
- 이미지 태그 및 Digest 관리
- Lifecycle Policy 지원
- 취약점 스캔 기능 활용 가능

### 대안 비교

| 구분 | Amazon ECR | Docker Hub | GitHub Container Registry |
|---|---|---|---|
| AWS IAM 연동 | 강함 | 별도 인증 필요 | GitHub Token 중심 |
| EKS 연동 | 편리 | ImagePullSecret 필요 가능 | ImagePullSecret 필요 가능 |
| Private Registry | 지원 | 요금제에 따라 상이 | 지원 |
| AWS 내부 전송 | 유리 | 외부 Registry | 외부 Registry |
| 운영 적합성 | AWS 환경에 적합 | 범용 | GitHub 중심 환경에 적합 |

---

## 3. Repository 구성

서비스별로 ECR Repository를 분리합니다.

```text
maesoongan/admin-service
maesoongan/auth-service
maesoongan/contest-service
maesoongan/market-service
maesoongan/order-service
```

### 서비스별 분리 이유

- 특정 서비스만 독립적으로 배포
- 서비스별 이미지 이력 관리
- 서비스별 Lifecycle Policy 적용
- 접근 권한 세분화 가능
- 문제 발생 시 영향 범위 분리

---

## 4. Naming Convention

```text
{project-name}/{service-name}
```

MaeSoonGan에서는 다음 규칙을 사용합니다.

```text
maesoongan/{service-name}
```

서비스 이름은 애플리케이션 디렉터리 및 Kubernetes Deployment 이름과 가능한 한 일치시킵니다.

---

## 5. Registry 주소

```text
{AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com
```

`ECR_REGISTRY`에는 Repository 경로를 포함하지 않습니다.

### 올바른 값

```text
{AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com
```

### 잘못된 값

```text
{AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/maesoongan/auth-service
```

최종 이미지 주소는 Workflow에서 조합합니다.

```text
$ECR_REGISTRY/maesoongan/${SERVICE_NAME}:${IMAGE_TAG}
```

---

## 6. Repository 생성

AWS CLI 예시:

```bash
aws ecr create-repository \
  --repository-name maesoongan/auth-service \
  --region ap-northeast-2 \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability MUTABLE
```

다른 서비스도 동일한 방식으로 생성합니다.

```bash
for service in \
  admin-service \
  auth-service \
  contest-service \
  market-service \
  order-service
do
  aws ecr create-repository \
    --repository-name maesoongan/$service \
    --region ap-northeast-2
done
```

이미 생성된 Repository에 대해서는 오류가 발생할 수 있으므로 실제 운영에서는 존재 여부를 먼저 확인합니다.

---

## 7. 이미지 Push 흐름

### 7.1 ECR 로그인

로컬에서 테스트하는 경우:

```bash
aws ecr get-login-password \
  --region ap-northeast-2 \
| docker login \
  --username AWS \
  --password-stdin \
  {AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com
```

GitHub Actions에서는 `aws-actions/amazon-ecr-login` Action을 사용합니다.

### 7.2 이미지 빌드

```bash
docker build \
  -f apps/api/auth-service/Dockerfile \
  -t auth-service:local \
  .
```

### 7.3 이미지 태그 지정

```bash
docker tag \
  auth-service:local \
  {ECR_REGISTRY}/maesoongan/auth-service:{IMAGE_TAG}
```

### 7.4 이미지 Push

```bash
docker push \
  {ECR_REGISTRY}/maesoongan/auth-service:{IMAGE_TAG}
```

---

## 8. 이미지 태그 전략

### `latest`

```text
maesoongan/auth-service:latest
```

가장 최근의 성공 빌드를 나타내는 편의용 태그입니다.

### Commit SHA

```text
maesoongan/auth-service:{GITHUB_SHA}
```

빌드한 Git Commit을 식별합니다.

### Release Version

```text
maesoongan/auth-service:v1.0.0
```

운영 Release 단위를 명확히 관리할 때 사용할 수 있습니다.

### 권장 기준

| 환경 | 권장 태그 |
|---|---|
| 로컬 개발 | `local`, `dev` |
| 개발 서버 | Commit SHA |
| 스테이징 | Commit SHA 또는 RC Version |
| 운영 | Release Version 또는 Commit SHA |

Kubernetes 운영 Manifest에서 `latest`만 사용하는 것은 권장하지 않습니다.

---

## 9. 이미지 확인

### Repository 목록

```bash
aws ecr describe-repositories \
  --region ap-northeast-2
```

### 특정 Repository 이미지 목록

```bash
aws ecr list-images \
  --repository-name maesoongan/auth-service \
  --region ap-northeast-2
```

### 이미지 상세 확인

```bash
aws ecr describe-images \
  --repository-name maesoongan/auth-service \
  --region ap-northeast-2
```

---

## 10. EKS Image Pull 권한

EKS Worker Node 또는 Pod Identity/IRSA 구성에 따라 ECR Pull 권한이 필요합니다.

대표적으로 필요한 권한은 다음과 같습니다.

```json
{
  "Effect": "Allow",
  "Action": [
    "ecr:GetAuthorizationToken",
    "ecr:BatchCheckLayerAvailability",
    "ecr:GetDownloadUrlForLayer",
    "ecr:BatchGetImage"
  ],
  "Resource": "*"
}
```

AWS 관리형 정책을 사용할 경우 Node Role에 다음 정책이 연결되어 있는지 확인합니다.

```text
AmazonEC2ContainerRegistryReadOnly
```

---

## 11. GitHub Actions Push 권한

GitHub Actions IAM Role에는 다음 권한이 필요합니다.

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
      "Resource": "arn:aws:ecr:ap-northeast-2:{AWS_ACCOUNT_ID}:repository/maesoongan/*"
    }
  ]
}
```

최소 권한 원칙에 따라 MaeSoonGan Repository 범위로 제한합니다.

---

## 12. Lifecycle Policy

이미지가 계속 쌓이면 저장 비용과 관리 부담이 증가하므로 Lifecycle Policy를 설정합니다.

예시:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only the latest 30 untagged images",
      "selection": {
        "tagStatus": "untagged",
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

운영에 필요한 Rollback 범위를 고려해 삭제 기준을 결정합니다.

---

## 13. 이미지 취약점 스캔

Repository 생성 시 Scan on Push를 활성화할 수 있습니다.

```bash
aws ecr put-image-scanning-configuration \
  --repository-name maesoongan/auth-service \
  --image-scanning-configuration scanOnPush=true \
  --region ap-northeast-2
```

취약점 결과는 배포 차단 정책 또는 보안 검토 자료로 활용할 수 있습니다.

---

## 14. Kubernetes에서 이미지 참조

```yaml
containers:
  - name: auth-service
    image: >-
      {AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/
      maesoongan/auth-service:{IMAGE_TAG}
```

실제 YAML에서는 한 줄로 작성합니다.

```yaml
image: {ECR_REGISTRY}/maesoongan/auth-service:{IMAGE_TAG}
```

### `imagePullPolicy`

고정 태그 사용 시:

```yaml
imagePullPolicy: IfNotPresent
```

`latest` 사용 시 Kubernetes가 기본적으로 `Always`를 사용할 수 있지만, 운영에서는 고정 태그 사용을 권장합니다.

---

## 15. 트러블슈팅

### 15.1 `repository does not exist`

확인할 항목:

- Repository 이름
- AWS Region
- AWS Account
- `maesoongan/` Prefix
- 오타 여부

### 15.2 `no basic auth credentials`

확인할 항목:

- ECR 로그인 여부
- Docker Credential 만료 여부
- AWS 자격증명 유효성
- GitHub Actions의 AWS Role Assume 성공 여부

### 15.3 `ImagePullBackOff`

확인할 항목:

- Kubernetes Manifest의 이미지 주소
- 태그 존재 여부
- EKS Node Role의 ECR Pull 권한
- ECR Repository Region
- AWS Account 일치 여부

### 15.4 Push 권한 거부

```text
is not authorized to perform: ecr:PutImage
```

GitHub Actions IAM Policy의 `Resource`와 실제 Repository ARN이 일치하는지 확인합니다.

### 15.5 이미지 태그가 덮어써짐

`latest`는 변경 가능한 태그입니다. 배포 추적이 필요하면 Commit SHA 또는 Version 태그를 사용합니다.

---

## 16. 운영 체크리스트

- [ ] 서비스별 Repository가 생성되어 있는가
- [ ] Repository Region이 `ap-northeast-2`인가
- [ ] GitHub Actions Role에 Push 권한이 있는가
- [ ] EKS Node Role에 Pull 권한이 있는가
- [ ] Commit SHA 태그가 생성되는가
- [ ] 취약점 스캔이 활성화되어 있는가
- [ ] Lifecycle Policy가 적용되어 있는가
- [ ] 운영 Manifest가 고정 태그를 사용하는가
