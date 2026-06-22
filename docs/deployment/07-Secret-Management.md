# 배포 환경변수 및 Secret 관리

## 1. 개요 및 프로젝트 내 역할

MaeSoonGan 백엔드 서비스는 DB, Redis, JWT, 외부 API 등 다양한 환경설정이 필요합니다.

환경변수는 민감도에 따라 ConfigMap과 Secret으로 분리합니다.

```text
일반 설정
    ↓
ConfigMap

민감정보
    ↓
Secret
```

---

## 2. 분리 기준

### ConfigMap

노출되어도 즉시 보안 사고로 이어지지 않는 일반 설정을 관리합니다.

예시:

```text
SPRING_PROFILES_ACTIVE
서버 Port
내부 서비스 URL
Redis Host
기능 Flag
일반 Timeout 값
```

### Secret

노출 시 인증 우회, 데이터 접근, 외부 서비스 악용으로 이어질 수 있는 값을 관리합니다.

예시:

```text
DB Username
DB Password
JWT Secret
Redis Password
외부 API Key
OAuth Client Secret
암호화 Key
```

---

## 3. ConfigMap 예시

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: app
data:
  SPRING_PROFILES_ACTIVE: "prod"
  REDIS_HOST: "redis-service"
  AUTH_SERVICE_URL: "http://auth-service:8081"
```

---

## 4. Secret 예제 파일

Repository에는 실제 값이 아닌 예제 파일만 저장합니다.

파일명:

```text
infra/k8s/api/secret.example.yaml
```

예시:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secret
  namespace: app
type: Opaque
stringData:
  SPRING_DATASOURCE_URL: "CHANGE_ME"
  SPRING_DATASOURCE_USERNAME: "CHANGE_ME"
  SPRING_DATASOURCE_PASSWORD: "CHANGE_ME"
  APP_JWT_SECRET: "CHANGE_ME"
  REDIS_PASSWORD: "CHANGE_ME"
```

`stringData`는 적용 시 Kubernetes가 자동으로 Base64 형태로 변환합니다.

---

## 5. 실제 Secret 파일 생성

Linux/macOS:

```bash
cp \
  infra/k8s/api/secret.example.yaml \
  /tmp/maesoongan-api-secret.yaml
```

PowerShell:

```powershell
Copy-Item `
  infra/k8s/api/secret.example.yaml `
  C:\Temp\maesoongan-api-secret.yaml
```

복사한 파일에 실제 값을 입력합니다.

Repository 내부 경로에 실제 파일을 만들지 않는 것을 권장합니다.

---

## 6. 필수 환경변수 예시

```text
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
APP_JWT_SECRET
REDIS_HOST
REDIS_PASSWORD
외부 API KEY
```

서비스별로 필요한 Key가 다르다면 Secret을 서비스별로 분리할 수 있습니다.

```text
auth-service-secret
order-service-secret
market-service-secret
```

공통 Secret 하나에 모든 값을 넣으면 관리가 간단하지만, 특정 서비스가 불필요한 Secret까지 읽게 될 수 있습니다.

---

## 7. Secret 적용

```bash
kubectl apply \
  -f /tmp/maesoongan-api-secret.yaml
```

확인:

```bash
kubectl get secret \
  -n app
```

상세 Metadata 확인:

```bash
kubectl describe secret \
  api-secret \
  -n app
```

`describe`는 값 자체를 출력하지 않지만 Key 이름은 확인할 수 있습니다.

---

## 8. 환경변수 주입

### 전체 Secret 주입

```yaml
envFrom:
  - secretRef:
      name: api-secret
```

### 특정 Key 주입

```yaml
env:
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: api-secret
        key: SPRING_DATASOURCE_PASSWORD
```

서비스별 최소 권한 관리가 필요하면 특정 Key 주입 방식을 고려합니다.

---

## 9. `.gitignore` 설정

```text
.env
.env.*
!.env.example

secret.yaml
*.secret.yaml
!secret.example.yaml

application-secret.yml
application-prod-secret.yml
```

예제 파일까지 제외되지 않도록 예외 규칙을 추가합니다.

---

## 10. Git에 이미 올라간 Secret 대응

`.gitignore`를 추가해도 이미 Commit된 파일은 자동으로 제거되지 않습니다.

Tracking 제거:

```bash
git rm --cached path/to/secret.yaml
```

그 후 Commit합니다.

```bash
git commit \
  -m "REMOVE: tracked secret file"
```

이미 외부에 노출되었다면 파일 삭제만으로 충분하지 않습니다.

반드시 다음 조치를 수행합니다.

- DB Password 변경
- JWT Secret 교체
- API Key 폐기 및 재발급
- Redis Password 변경
- Git History 정리 검토
- 접근 로그 확인

---

## 11. Base64에 대한 주의

Kubernetes Secret의 `data` 값은 Base64로 인코딩되지만 암호화 자체는 아닙니다.

```text
Base64 ≠ Encryption
```

권한이 있는 사용자는 쉽게 복호화할 수 있습니다.

따라서 다음 항목이 중요합니다.

- Kubernetes RBAC
- etcd Encryption at Rest
- 접근 로그
- Secret Store 연동
- 최소 권한

---

## 12. Secret 변경 및 Pod 반영

Secret을 수정합니다.

```bash
kubectl apply \
  -f /tmp/maesoongan-api-secret.yaml
```

환경변수로 주입된 Secret은 기존 Pod에 자동 반영되지 않습니다.

Deployment 재시작:

```bash
kubectl rollout restart \
  deployment/auth-service \
  -n app
```

상태 확인:

```bash
kubectl rollout status \
  deployment/auth-service \
  -n app
```

---

## 13. ConfigMap 변경 반영

ConfigMap도 환경변수로 주입한 경우 기존 Pod에 자동 반영되지 않습니다.

```bash
kubectl apply \
  -f infra/k8s/api/configmap.yaml
```

```bash
kubectl rollout restart \
  deployment/auth-service \
  -n app
```

GitOps 도입 후에는 ConfigMap/Secret 변경 방식과 재배포 정책을 별도로 정의합니다.

---

## 14. 서비스별 Secret 분리 전략

### 공통 Secret

장점:

- 파일 수가 적음
- 초기 관리가 단순함

단점:

- 모든 서비스가 불필요한 값까지 접근
- 변경 영향 범위가 큼

### 서비스별 Secret

장점:

- 최소 권한 적용
- 서비스별 변경 및 회전 가능
- 장애 영향 범위 분리

단점:

- 리소스와 관리 항목 증가

운영 환경에서는 서비스별 분리를 권장합니다.

---

## 15. 외부 Secret Manager 연동

향후 다음 도구를 검토할 수 있습니다.

- AWS Secrets Manager
- AWS Systems Manager Parameter Store
- External Secrets Operator
- Secrets Store CSI Driver

예상 구조:

```text
AWS Secrets Manager
    ↓
External Secrets Operator
    ↓
Kubernetes Secret
    ↓
Application Pod
```

장점:

- Secret 중앙 관리
- Rotation 자동화
- Git에 Secret 미저장
- 접근 이력 관리
- IAM 기반 접근 제어

---

## 16. Secret 이름 규칙

```text
{service-name}-secret
{environment}-{service-name}-secret
```

예시:

```text
dev-auth-service-secret
prod-auth-service-secret
prod-order-service-secret
```

환경과 서비스 단위를 명확히 구분합니다.

---

## 17. 트러블슈팅

### 17.1 `CreateContainerConfigError`

가능한 원인:

- Secret 없음
- Secret 이름 불일치
- Key 없음
- Namespace 불일치

```bash
kubectl describe pod \
  -n app \
  {pod-name}
```

### 17.2 환경변수가 비어 있음

확인 항목:

- `envFrom` 또는 `env` 설정
- Secret Key 대소문자
- 적용 Namespace
- Pod 재시작 여부
- Spring Boot Property Mapping

### 17.3 Secret 변경이 반영되지 않음

환경변수 주입 방식이라면 Pod를 재시작합니다.

```bash
kubectl rollout restart \
  deployment/{deployment-name} \
  -n app
```

### 17.4 잘못된 DB URL

```bash
kubectl logs \
  -n app \
  {pod-name}
```

JDBC URL 형식, Host, Port, Database Name, SSL 옵션을 확인합니다.

---

## 18. 보안 체크리스트

- [ ] 실제 Secret 파일이 Git에 없는가
- [ ] `.env` 파일이 Git에서 제외됐는가
- [ ] 공개 문서에 AWS Account와 Key가 노출되지 않았는가
- [ ] 서비스가 필요한 Secret만 읽는가
- [ ] Secret 변경 시 Pod 재시작 절차가 있는가
- [ ] 유출된 Secret은 즉시 폐기하고 재발급했는가
- [ ] Kubernetes RBAC가 최소 권한인가
- [ ] 운영 Secret의 외부 Secret Manager 연동을 검토했는가
