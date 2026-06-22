# GitHub Actions AWS OIDC 인증 구성

## 1. 개요 및 프로젝트 내 역할

GitHub Actions가 Amazon ECR에 이미지를 Push하려면 AWS 권한이 필요합니다.

MaeSoonGan은 장기 Access Key를 GitHub Secrets에 저장하는 대신 GitHub OIDC와 AWS IAM Role을 사용합니다.

```text
GitHub Actions 실행
    ↓
GitHub OIDC Token 발급
    ↓
AWS STS에 Token 전달
    ↓
IAM Trust Policy 검증
    ↓
임시 AWS 자격증명 발급
    ↓
Amazon ECR 접근
```

---

## 2. OIDC 선택 이유

### Access Key 방식의 문제

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

장기 Key를 GitHub에 저장하면 다음 위험이 있습니다.

- Key 유출
- 교체 및 폐기 관리 부담
- 사용 주체 식별 어려움
- 불필요하게 긴 유효기간
- 권한 오남용 가능성

### OIDC 방식의 장점

- 장기 Access Key 저장 불필요
- 실행 시점에만 임시 권한 발급
- Repository와 Branch 조건 제한 가능
- AWS CloudTrail을 통한 추적 가능
- 최소 권한 원칙 적용 가능

---

## 3. 전체 구성 요소

```text
GitHub Repository
GitHub Actions Workflow
GitHub OIDC Provider
AWS IAM Policy
AWS IAM Role
IAM Trust Policy
GitHub Secrets
```

---

## 4. IAM OIDC Provider 구성

AWS Console 경로:

```text
AWS Console
→ IAM
→ Identity providers
→ Add provider
```

설정값:

| 항목 | 값 |
|---|---|
| Provider type | OpenID Connect |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

Audience는 반드시 다음 값을 사용합니다.

```text
sts.amazonaws.com
```

`sigstore`는 이미지 서명 등 다른 용도의 Audience이며, AWS STS Role Assume에는 사용하지 않습니다.

---

## 5. ECR Push IAM Policy

정책 이름 예시:

```text
MaesoonganGitHubActionsEcrPushPolicy
```

정책 예시:

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

### JSON 작성 주의사항

- 주석을 넣지 않음
- 문자열은 큰따옴표 사용
- 마지막 항목 뒤에 쉼표를 넣지 않음
- `{AWS_ACCOUNT_ID}`를 실제 값으로 교체
- Repository ARN과 Region을 정확히 확인

---

## 6. IAM Role 생성

Role 이름 예시:

```text
MaesoonganGitHubActionsEcrRole
```

이 Role은 GitHub Actions가 Assume하며, 앞에서 생성한 ECR Push Policy를 연결합니다.

### Role 역할

- GitHub OIDC Token 검증
- 조건이 일치하면 임시 자격증명 발급
- ECR Push 권한 제공
- 허용된 Repository와 Branch만 접근 허용

---

## 7. Trust Policy 구성

GitHub Owner:

```text
MaeSoonGan
```

GitHub Repository:

```text
Back-End
```

허용 Branch:

```text
develop
main
```

Trust Policy 예시:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::{AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:MaeSoonGan/Back-End:ref:refs/heads/develop",
            "repo:MaeSoonGan/Back-End:ref:refs/heads/main"
          ]
        }
      }
    }
  ]
}
```

---

## 8. `sub` 조건 이해

GitHub OIDC Token의 `sub`는 Workflow 실행 주체를 나타냅니다.

Branch 기준 형식:

```text
repo:{OWNER}/{REPOSITORY}:ref:refs/heads/{BRANCH}
```

MaeSoonGan 예시:

```text
repo:MaeSoonGan/Back-End:ref:refs/heads/develop
```

다음 항목은 대소문자와 기호까지 정확히 일치해야 합니다.

- Owner
- Repository
- Branch
- `/`
- `-`
- 대소문자

---

## 9. GitHub Actions 권한

Workflow에 다음 권한을 선언합니다.

```yaml
permissions:
  id-token: write
  contents: read
```

`id-token: write`가 없으면 GitHub가 OIDC Token을 발급할 수 없습니다.

---

## 10. GitHub Secrets 등록

```text
Repository
→ Settings
→ Secrets and variables
→ Actions
```

등록 항목:

```text
AWS_REGION
AWS_ROLE_ARN
ECR_REGISTRY
```

예시:

```text
AWS_REGION=ap-northeast-2
AWS_ROLE_ARN=arn:aws:iam::{AWS_ACCOUNT_ID}:role/MaesoonganGitHubActionsEcrRole
ECR_REGISTRY={AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com
```

실제 Access Key는 등록하지 않습니다.

---

## 11. Workflow 연동

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ${{ secrets.AWS_REGION }}
```

이 단계가 성공하면 이후 Step에서 AWS CLI 및 ECR Action이 임시 자격증명을 사용합니다.

---

## 12. 설정 검증

### 12.1 Workflow에서 AWS Identity 확인

일시적으로 다음 Step을 추가할 수 있습니다.

```yaml
- name: Verify AWS identity
  run: aws sts get-caller-identity
```

확인 항목:

- Account
- Assumed Role ARN
- Session 이름

민감정보 노출 가능성을 고려해 검증 후 제거하거나 로그 공개 범위를 확인합니다.

### 12.2 IAM Role 확인

```bash
aws iam get-role \
  --role-name MaesoonganGitHubActionsEcrRole
```

### 12.3 OIDC Provider 확인

```bash
aws iam list-open-id-connect-providers
```

---

## 13. 트러블슈팅

### 13.1 `Incorrect token audience`

```text
Could not assume role with OIDC: Incorrect token audience
```

확인 항목:

- OIDC Provider Audience
- Trust Policy의 `aud`
- Workflow Action 설정

정상 값:

```text
sts.amazonaws.com
```

### 13.2 `Not authorized to perform sts:AssumeRoleWithWebIdentity`

가능한 원인:

- Trust Policy의 Owner 불일치
- Repository 이름 불일치
- Branch 조건 불일치
- 다른 IAM Role ARN 사용
- OIDC Provider ARN Account 불일치
- `id-token: write` 누락

### 13.3 Branch가 허용되지 않음

Workflow를 Feature Branch에서 수동 실행하면 Trust Policy가 거부할 수 있습니다.

해결 방법:

- `develop` 또는 `main`에서 실행
- 필요한 Branch만 Trust Policy에 추가
- 과도한 Wildcard 사용은 피함

### 13.4 IAM Policy는 연결됐지만 ECR Push 실패

확인 항목:

- Policy의 Repository ARN
- Region
- AWS Account ID
- `ecr:PutImage`
- Layer Upload 관련 권한

### 13.5 Role ARN 오타

GitHub Secret의 `AWS_ROLE_ARN`이 실제 Role을 가리키는지 확인합니다.

```text
arn:aws:iam::{AWS_ACCOUNT_ID}:role/MaesoonganGitHubActionsEcrRole
```

---

## 14. 보안 권장사항

### 최소 권한

전체 ECR이 아니라 MaeSoonGan Repository에만 권한을 부여합니다.

```text
arn:aws:ecr:ap-northeast-2:{AWS_ACCOUNT_ID}:repository/maesoongan/*
```

### Branch 제한

```text
develop
main
```

필요한 Branch만 허용합니다.

### Environment 보호 규칙

운영 배포는 GitHub Environment를 활용하여 승인 절차를 추가할 수 있습니다.

```text
development
staging
production
```

### CloudTrail 확인

Assume Role과 ECR 작업 이력을 CloudTrail에서 확인할 수 있습니다.

---

## 15. 운영 체크리스트

- [ ] OIDC Provider URL이 정확한가
- [ ] Audience가 `sts.amazonaws.com`인가
- [ ] IAM Role에 ECR Push Policy가 연결됐는가
- [ ] Trust Policy의 Owner와 Repository가 정확한가
- [ ] 허용 Branch가 실제 실행 Branch와 일치하는가
- [ ] Workflow에 `id-token: write`가 있는가
- [ ] GitHub Secret의 Role ARN이 정확한가
- [ ] 장기 Access Key를 사용하지 않는가
