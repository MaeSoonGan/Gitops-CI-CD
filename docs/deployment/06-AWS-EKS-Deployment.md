# AWS EKS 클러스터 연결 및 배포

## 1. 개요 및 프로젝트 내 역할

Amazon EKS는 MaeSoonGan 백엔드 컨테이너가 실행되는 Kubernetes 환경입니다.

EKS 배포 과정은 다음과 같습니다.

```text
AWS CLI 인증
    ↓
EKS Cluster 확인
    ↓
kubeconfig 설정
    ↓
Kubernetes Context 확인
    ↓
Worker Node 확인
    ↓
Namespace와 Secret 적용
    ↓
애플리케이션 Manifest 적용
    ↓
Pod와 Service 상태 확인
```

---

## 2. 사전 요구사항

다음 도구가 설치되어 있어야 합니다.

```bash
aws --version
kubectl version --client
```

필요한 권한:

- EKS Cluster 조회
- EKS 인증 Token 발급
- Kubernetes API 접근
- 해당 Cluster의 Kubernetes 권한

---

## 3. AWS 계정 확인

```bash
aws sts get-caller-identity
```

확인 항목:

```text
Account
Arn
UserId
```

의도한 AWS Account와 IAM Principal을 사용하고 있는지 확인합니다.

---

## 4. EKS Cluster 목록 확인

```bash
aws eks list-clusters \
  --region ap-northeast-2
```

MaeSoonGan Cluster 이름:

```text
maesoongan-cluster
```

---

## 5. Cluster 상태 확인

```bash
aws eks describe-cluster \
  --region ap-northeast-2 \
  --name maesoongan-cluster
```

필요한 정보만 조회하는 예시:

```bash
aws eks describe-cluster \
  --region ap-northeast-2 \
  --name maesoongan-cluster \
  --query "cluster.{
    status:status,
    endpoint:endpoint,
    publicAccess:resourcesVpcConfig.endpointPublicAccess,
    privateAccess:resourcesVpcConfig.endpointPrivateAccess,
    publicCidrs:resourcesVpcConfig.publicAccessCidrs
  }"
```

---

## 6. kubeconfig 업데이트

```bash
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name maesoongan-cluster
```

성공 시 Kubernetes Context가 로컬 kubeconfig에 추가됩니다.

---

## 7. 현재 Context 확인

```bash
kubectl config current-context
```

전체 Context 목록:

```bash
kubectl config get-contexts
```

필요한 경우 Context를 전환합니다.

```bash
kubectl config use-context {context-name}
```

잘못된 Cluster에 배포하지 않도록 적용 전 Context를 확인합니다.

---

## 8. Cluster 접근 확인

```bash
kubectl cluster-info
```

```bash
kubectl get namespaces
```

정상 응답이 오면 Kubernetes API에 접근 가능한 상태입니다.

---

## 9. Worker Node 확인

```bash
kubectl get nodes
```

상세 확인:

```bash
kubectl get nodes -o wide
```

정상 상태:

```text
STATUS: Ready
```

Node가 보이지 않거나 `NotReady`라면 Node Group과 네트워크 구성을 확인합니다.

---

## 10. Node Label 구성

현재 Label 확인:

```bash
kubectl get nodes --show-labels
```

서비스용 Label 추가:

```bash
kubectl label node {node-name} role=service
```

Label 확인:

```bash
kubectl get nodes \
  -l role=service
```

Deployment의 `nodeSelector`가 `role=service`를 요구한다면 최소 한 개 이상의 Node에 해당 Label이 있어야 합니다.

---

## 11. EKS API Endpoint 접근 구조

현재 구성 예시:

```text
publicAccess: false
privateAccess: true
```

Private Endpoint만 활성화된 경우 VPC 외부의 로컬 PC에서는 Kubernetes API에 접근할 수 없습니다.

---

## 12. Private Endpoint 접근 방법

### 방법 1. Bastion EC2

같은 VPC의 Bastion EC2에 접속한 후 AWS CLI와 `kubectl`을 사용합니다.

### 방법 2. VPN

로컬 PC를 EKS VPC와 연결한 후 Private Endpoint에 접근합니다.

### 방법 3. VPC 내부 관리 서버

배포 전용 EC2 또는 운영 관리 서버를 사용합니다.

### 방법 4. Cloud 기반 개발 환경

VPC 연결이 가능한 관리 환경을 사용합니다.

Private Endpoint 유지 방식은 보안상 유리하지만 운영 접근 경로를 별도로 준비해야 합니다.

---

## 13. Public Endpoint 임시 활성화

팀 협의 후 제한적으로 활성화할 수 있습니다.

```bash
aws eks update-cluster-config \
  --region ap-northeast-2 \
  --name maesoongan-cluster \
  --resources-vpc-config \
  endpointPublicAccess=true,endpointPrivateAccess=true
```

Public CIDR는 현재 공인 IP만 허용하는 것을 권장합니다.

```text
현재 공인 IP/32
```

전체 인터넷을 의미하는 `0.0.0.0/0` 사용은 피합니다.

---

## 14. Kubernetes 배포 순서

### 14.1 Namespace 생성

```bash
kubectl apply \
  -f infra/k8s/api/namespace.yaml
```

확인:

```bash
kubectl get namespace app
```

### 14.2 Secret 생성

```bash
cp \
  infra/k8s/api/secret.example.yaml \
  /tmp/maesoongan-api-secret.yaml
```

실제 값을 입력한 후 적용합니다.

```bash
kubectl apply \
  -f /tmp/maesoongan-api-secret.yaml
```

확인:

```bash
kubectl get secret -n app
```

### 14.3 Manifest 검증

```bash
kubectl kustomize infra/k8s/api
```

### 14.4 애플리케이션 배포

```bash
kubectl apply -k infra/k8s/api
```

---

## 15. 배포 상태 확인

```bash
kubectl get deployments -n app
kubectl get pods -n app
kubectl get services -n app
kubectl get endpoints -n app
```

Pod 배치 Node와 IP 확인:

```bash
kubectl get pods -n app -o wide
```

---

## 16. Rollout 상태 확인

```bash
kubectl rollout status \
  deployment/auth-service \
  -n app
```

전체 Deployment 확인:

```bash
kubectl get deployments -n app
```

Rollout 이력:

```bash
kubectl rollout history \
  deployment/auth-service \
  -n app
```

---

## 17. 이미지 변경

```bash
kubectl set image \
  deployment/auth-service \
  auth-service={ECR_REGISTRY}/maesoongan/auth-service:{IMAGE_TAG} \
  -n app
```

다만 GitOps 도입 후에는 직접 변경하지 않고 Git Manifest를 수정합니다.

---

## 18. 롤백

```bash
kubectl rollout undo \
  deployment/auth-service \
  -n app
```

특정 Revision으로 롤백:

```bash
kubectl rollout undo \
  deployment/auth-service \
  --to-revision=2 \
  -n app
```

운영에서는 Rollback 대상 이미지 태그와 변경 이력을 함께 기록합니다.

---

## 19. 주요 운영 명령어

### Pod 상세 확인

```bash
kubectl describe pod \
  -n app \
  {pod-name}
```

### 로그 확인

```bash
kubectl logs \
  -n app \
  {pod-name}
```

### 이전 컨테이너 로그

```bash
kubectl logs \
  -n app \
  {pod-name} \
  --previous
```

### 이벤트 확인

```bash
kubectl get events \
  -n app \
  --sort-by=.metadata.creationTimestamp
```

---

## 20. 트러블슈팅

### 20.1 `kubectl` Timeout

가능한 원인:

- EKS Private Endpoint
- VPN 미연결
- Security Group
- Route Table
- DNS
- 잘못된 Endpoint 접근

### 20.2 `Unauthorized`

가능한 원인:

- AWS 자격증명 불일치
- EKS Access Entry 또는 `aws-auth` 권한 없음
- 만료된 Session
- 다른 AWS Profile 사용

### 20.3 Node가 표시되지 않음

확인 항목:

- Managed Node Group 상태
- EC2 Instance 상태
- Node IAM Role
- Subnet
- Security Group
- Cluster Version 호환성

### 20.4 Pod가 `Pending`

```bash
kubectl describe pod -n app {pod-name}
```

확인 항목:

- NodeSelector
- Resource 부족
- Taint
- Node Ready 상태

### 20.5 `ImagePullBackOff`

확인 항목:

- ECR 주소
- 이미지 태그
- Node Role의 ECR Pull 권한
- Region
- Repository 존재 여부

---

## 21. 보안 주의사항

- EKS Public Endpoint는 필요한 경우에만 활성화
- Public CIDR는 신뢰 가능한 IP로 제한
- kubeconfig 파일 공유 금지
- 운영 Cluster 접근 권한 최소화
- 개인 IAM User보다 Role 기반 접근 권장
- 배포 명령 전 Context 확인
- 실제 Secret을 터미널 로그와 Git에 남기지 않음

---

## 22. 배포 체크리스트

- [ ] AWS Account가 정확한가
- [ ] Region이 `ap-northeast-2`인가
- [ ] 현재 Context가 `maesoongan-cluster`인가
- [ ] Kubernetes API 접근이 가능한가
- [ ] Node가 `Ready` 상태인가
- [ ] `role=service` Label이 존재하는가
- [ ] `app` Namespace가 생성됐는가
- [ ] 실제 Secret이 적용됐는가
- [ ] ECR 이미지가 존재하는가
- [ ] Kustomize 결과를 검토했는가
- [ ] Pod가 `Running`, `Ready 1/1`인가
- [ ] Service Endpoint가 생성됐는가
