# 배포 운영 및 트러블슈팅 가이드

## 1. 개요

이 문서는 MaeSoonGan 배포 과정에서 발생할 수 있는 문제를 단계별로 확인하기 위한 가이드입니다.

장애를 분석할 때는 다음 순서로 범위를 좁힙니다.

```text
GitHub Actions
    ↓
AWS OIDC / IAM
    ↓
Amazon ECR
    ↓
EKS API 접근
    ↓
Kubernetes Scheduling
    ↓
Container 실행
    ↓
Service / Endpoint
    ↓
애플리케이션 외부 연동
```

---

## 2. 기본 점검 순서

### 2.1 GitHub Actions 성공 여부

```text
Checkout
Configure AWS credentials
Login to ECR
Docker Build
Docker Push
```

### 2.2 ECR 이미지 존재 여부

```bash
aws ecr describe-images \
  --repository-name maesoongan/auth-service \
  --region ap-northeast-2
```

### 2.3 EKS 접근 여부

```bash
kubectl get nodes
```

### 2.4 Pod 상태

```bash
kubectl get pods \
  -n app \
  -o wide
```

### 2.5 Pod 이벤트

```bash
kubectl describe pod \
  -n app \
  {pod-name}
```

### 2.6 애플리케이션 로그

```bash
kubectl logs \
  -n app \
  {pod-name}
```

---

## 3. GitHub Actions 문제

### 3.1 `Incorrect token audience`

증상:

```text
Could not assume role with OIDC: Incorrect token audience
```

원인:

- OIDC Provider Audience 오류
- Trust Policy의 `aud` 오류

해결:

```text
sts.amazonaws.com
```

### 3.2 `Not authorized to perform sts:AssumeRoleWithWebIdentity`

원인 후보:

- Owner 불일치
- Repository 이름 불일치
- Branch 불일치
- Role ARN 오류
- OIDC Provider ARN 오류
- `id-token: write` 누락

확인 예시:

```text
repo:MaeSoonGan/Back-End:ref:refs/heads/develop
```

### 3.3 ECR Push 권한 오류

```text
not authorized to perform: ecr:PutImage
```

확인 항목:

- IAM Policy 연결
- Repository ARN
- Region
- Account ID
- `ecr:PutImage`
- Layer Upload 권한

---

## 4. Docker Build 문제

### 4.1 `gradlew exit code 126`

원인:

```text
gradlew 실행 권한 없음
```

해결:

```dockerfile
RUN chmod +x gradlew
```

### 4.2 멀티 프로젝트 디렉터리 오류

```text
Configuring project ':apps:worker' without an existing directory is not allowed.
```

해결:

```dockerfile
COPY apps ./apps
```

### 4.3 JAR 복사 실패

```text
COPY failed: no source files were specified
```

확인 항목:

- Gradle Task 성공
- 서비스 경로
- `build/libs`
- JAR 파일명
- `bootJar` 활성화 여부

### 4.4 Docker Build Context 오류

Repository Root에서 실행합니다.

```bash
docker build \
  -f apps/api/auth-service/Dockerfile \
  -t auth-service:local \
  .
```

---

## 5. ECR 문제

### 5.1 Repository가 없음

```text
repository does not exist
```

확인 항목:

- `maesoongan/{service-name}`
- AWS Region
- AWS Account
- 이름 오타

### 5.2 인증정보 없음

```text
no basic auth credentials
```

해결:

- ECR 로그인 확인
- AWS 자격증명 확인
- GitHub Actions OIDC Step 확인

### 5.3 태그 없음

Kubernetes Manifest가 참조하는 태그가 실제 ECR에 있는지 확인합니다.

```bash
aws ecr list-images \
  --repository-name maesoongan/auth-service \
  --region ap-northeast-2
```

---

## 6. EKS API 접근 문제

### 6.1 `kubectl` Timeout

원인 후보:

- Private Endpoint만 활성화
- VPN 미연결
- VPC 외부 접근
- Security Group
- Route Table
- DNS

Cluster 설정 확인:

```bash
aws eks describe-cluster \
  --region ap-northeast-2 \
  --name maesoongan-cluster \
  --query "cluster.resourcesVpcConfig"
```

### 6.2 `Unauthorized`

원인 후보:

- 다른 AWS Profile
- EKS Kubernetes 권한 없음
- Session 만료
- kubeconfig Context 오류

확인:

```bash
aws sts get-caller-identity
kubectl config current-context
```

---

## 7. Pod 상태별 문제

### 7.1 `Pending`

대표 원인:

- NodeSelector 불일치
- CPU/Memory 부족
- Node `NotReady`
- Taint와 Toleration
- PVC 문제

확인:

```bash
kubectl describe pod \
  -n app \
  {pod-name}
```

Node Label 확인:

```bash
kubectl get nodes --show-labels
```

필요한 Label:

```bash
kubectl label node \
  {node-name} \
  role=service
```

### 7.2 `ImagePullBackOff`

대표 원인:

- 이미지 경로 오류
- 태그 없음
- ECR Pull 권한 없음
- 다른 Region 또는 Account
- Repository 없음

확인:

```bash
kubectl describe pod \
  -n app \
  {pod-name}
```

### 7.3 `CrashLoopBackOff`

대표 원인:

- Spring Boot 실행 실패
- 환경변수 누락
- DB 연결 실패
- Redis 연결 실패
- JWT Secret 누락
- Port 충돌
- 잘못된 Profile

로그:

```bash
kubectl logs \
  -n app \
  {pod-name}
```

이전 컨테이너 로그:

```bash
kubectl logs \
  -n app \
  {pod-name} \
  --previous
```

### 7.4 `CreateContainerConfigError`

대표 원인:

- Secret 없음
- ConfigMap 없음
- Key 불일치
- Namespace 불일치

확인:

```bash
kubectl get secret -n app
kubectl get configmap -n app
kubectl describe pod -n app {pod-name}
```

### 7.5 `OOMKilled`

원인:

```text
컨테이너가 Memory Limit 초과
```

확인:

```bash
kubectl describe pod \
  -n app \
  {pod-name}
```

대응:

- Memory Limit 조정
- JVM Heap 설정 검토
- Memory Leak 분석
- 트래픽 및 Batch 처리량 확인

---

## 8. Probe 문제

### 8.1 Readiness Probe 실패

증상:

- Pod는 `Running`
- `READY 0/1`
- Service Endpoint에서 제외

확인 항목:

- Actuator Endpoint
- Port
- Path
- 초기 지연시간
- 인증 적용 여부

### 8.2 Liveness Probe 실패

증상:

- Pod가 반복 재시작
- `RESTARTS` 증가

확인 항목:

- 애플리케이션 시작 시간
- Probe Path
- DB 장애를 Liveness에 포함했는지
- `initialDelaySeconds`
- Startup Probe 필요 여부

---

## 9. Service 및 Endpoint 문제

### 9.1 Endpoint가 없음

```bash
kubectl get endpoints \
  -n app
```

원인 후보:

- Service Selector 불일치
- Pod Label 불일치
- Readiness 실패
- 다른 Namespace

### 9.2 Service 연결 실패

확인 항목:

- `port`
- `targetPort`
- `containerPort`
- 애플리케이션 `server.port`

### 9.3 DNS 확인

```bash
kubectl run dns-test \
  --rm -it \
  --restart=Never \
  --image=busybox:1.36 \
  -n app \
  -- nslookup auth-service
```

### 9.4 HTTP 연결 확인

```bash
kubectl run curl-test \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  -n app \
  -- curl -v http://auth-service:8081/actuator/health
```

---

## 10. ConfigMap 및 Secret 문제

### Secret Key 확인

```bash
kubectl describe secret \
  api-secret \
  -n app
```

### ConfigMap 확인

```bash
kubectl get configmap \
  api-config \
  -n app \
  -o yaml
```

### 변경 반영

```bash
kubectl rollout restart \
  deployment/auth-service \
  -n app
```

---

## 11. DB 연결 문제

로그에서 다음 항목을 확인합니다.

- Connection Refused
- Authentication Failed
- Unknown Host
- Timeout
- SSL 오류
- Connection Pool 부족

확인 항목:

```text
JDBC URL
DB Host
DB Port
Database Name
Username
Password
Security Group
Route
DNS
```

---

## 12. Redis 연결 문제

확인 항목:

```text
REDIS_HOST
REDIS_PORT
REDIS_PASSWORD
TLS 사용 여부
Security Group
Redis Cluster Mode
DNS
```

Pod 내부에서 연결을 점검할 때는 전용 진단 Pod를 사용하는 것이 좋습니다.

---

## 13. 배포 Rollout 문제

상태 확인:

```bash
kubectl rollout status \
  deployment/auth-service \
  -n app
```

이력 확인:

```bash
kubectl rollout history \
  deployment/auth-service \
  -n app
```

롤백:

```bash
kubectl rollout undo \
  deployment/auth-service \
  -n app
```

---

## 14. 이벤트 확인

Namespace 이벤트:

```bash
kubectl get events \
  -n app \
  --sort-by=.metadata.creationTimestamp
```

Cluster 전체 이벤트:

```bash
kubectl get events \
  -A \
  --sort-by=.metadata.creationTimestamp
```

이벤트는 Image Pull, Scheduling, Probe, Mount 오류를 빠르게 확인하는 데 유용합니다.

---

## 15. 장애 대응 기록 양식

```markdown
## 장애 제목

### 발생 시각

### 영향 범위

### 최초 증상

### 확인한 로그 및 이벤트

### 원인

### 임시 조치

### 근본 해결

### 재발 방지

### 관련 Commit / Image Tag
```

---

## 16. 최종 체크리스트

- [ ] GitHub Actions가 성공했는가
- [ ] ECR에 해당 태그가 존재하는가
- [ ] 현재 AWS Account가 정확한가
- [ ] 현재 Kubernetes Context가 정확한가
- [ ] Node가 `Ready`인가
- [ ] NodeSelector와 Label이 일치하는가
- [ ] Secret과 ConfigMap이 존재하는가
- [ ] Pod Event에 오류가 없는가
- [ ] 애플리케이션 로그가 정상인가
- [ ] Readiness가 성공하는가
- [ ] Service Endpoint가 생성됐는가
- [ ] 서비스 간 DNS와 Port가 정상인가
