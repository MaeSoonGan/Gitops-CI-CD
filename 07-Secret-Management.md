# Kubernetes 애플리케이션 배포 구성

## 1. 개요 및 프로젝트 내 역할

Kubernetes Manifest는 MaeSoonGan 백엔드 서비스를 EKS에서 어떤 방식으로 실행할지 정의합니다.

주요 관리 대상은 다음과 같습니다.

```text
Namespace
Deployment
Service
ConfigMap
Secret
Readiness Probe
Liveness Probe
NodeSelector
Kustomization
```

---

## 2. 디렉터리 구조

```text
infra/k8s/api
├── namespace.yaml
├── configmap.yaml
├── secret.example.yaml
├── admin-service.yaml
├── auth-service.yaml
├── contest-service.yaml
├── market-service.yaml
├── order-service.yaml
├── kustomization.yaml
└── README.md
```

---

## 3. Namespace 구성

MaeSoonGan 애플리케이션은 `app` Namespace에 배포합니다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
```

다른 리소스에는 다음 값을 지정합니다.

```yaml
metadata:
  namespace: app
```

### ECR Prefix와 Namespace 구분

```text
Kubernetes Namespace: app
ECR Repository Prefix: maesoongan
```

두 값은 서로 다른 목적을 가집니다.

---

## 4. Deployment 구성

Deployment는 다음 항목을 관리합니다.

- Pod 생성
- Replica 수 유지
- Docker 이미지 지정
- 환경변수 주입
- Rolling Update
- Health Check
- Node 배치
- 장애 발생 시 Pod 재생성

예시:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: app
spec:
  replicas: 2

  selector:
    matchLabels:
      app: auth-service

  template:
    metadata:
      labels:
        app: auth-service

    spec:
      nodeSelector:
        role: service

      containers:
        - name: auth-service
          image: >-
            {ECR_REGISTRY}/maesoongan/auth-service:{IMAGE_TAG}

          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 8081

          envFrom:
            - configMapRef:
                name: api-config
            - secretRef:
                name: api-secret

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 20
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 40
            periodSeconds: 20

          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

실제 YAML에서는 `image` 값을 한 줄로 작성합니다.

---

## 5. Label과 Selector

Deployment의 Selector와 Pod Label은 반드시 일치해야 합니다.

```yaml
selector:
  matchLabels:
    app: auth-service
```

```yaml
template:
  metadata:
    labels:
      app: auth-service
```

Service도 동일한 Label을 선택해야 합니다.

```yaml
selector:
  app: auth-service
```

불일치하면 Service Endpoint가 생성되지 않습니다.

---

## 6. Replica 구성

```yaml
replicas: 2
```

Pod를 2개 이상 실행하면 단일 Pod 장애 시에도 트래픽을 처리할 수 있습니다.

단, 다음 조건을 함께 고려해야 합니다.

- 애플리케이션이 Stateless한가
- Session을 외부 저장소에 관리하는가
- DB Connection 수가 충분한가
- Node Resource가 충분한가
- Readiness Probe가 정확한가

---

## 7. Rolling Update

기본 Rolling Update 전략을 명시할 수 있습니다.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

| 설정 | 의미 |
|---|---|
| `maxUnavailable: 0` | 업데이트 중 사용 불가능한 Pod를 허용하지 않음 |
| `maxSurge: 1` | 기존 Replica보다 최대 1개 더 생성 |

충분한 Cluster Resource와 정상적인 Probe가 전제되어야 합니다.

---

## 8. Service 구성

Service는 Pod IP 변경과 관계없이 고정된 내부 접근 주소를 제공합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: app
spec:
  type: ClusterIP

  selector:
    app: auth-service

  ports:
    - name: http
      port: 8081
      targetPort: 8081
```

### 내부 통신 예시

같은 Namespace:

```text
http://auth-service:8081
```

다른 Namespace:

```text
http://auth-service.app.svc.cluster.local:8081
```

---

## 9. ConfigMap 구성

민감하지 않은 설정값을 관리합니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: app
data:
  SPRING_PROFILES_ACTIVE: "prod"
  REDIS_HOST: "redis-service"
```

권장 관리 대상:

- Spring Profile
- 서버 Port
- 내부 Service URL
- 일반 기능 Flag
- 민감하지 않은 Host 정보

---

## 10. Secret 구성

민감정보는 Secret으로 관리합니다.

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

Repository에는 실제 값이 아니라 `secret.example.yaml`만 저장합니다.

---

## 11. 환경변수 주입

전체 ConfigMap과 Secret을 한 번에 주입하는 방식:

```yaml
envFrom:
  - configMapRef:
      name: api-config
  - secretRef:
      name: api-secret
```

특정 Key만 주입하는 방식:

```yaml
env:
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: api-secret
        key: SPRING_DATASOURCE_PASSWORD
```

명시적인 Key 관리가 필요할 때 두 번째 방식을 사용할 수 있습니다.

---

## 12. Readiness Probe

Readiness Probe는 Pod가 트래픽을 받을 준비가 되었는지 판단합니다.

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8081
  initialDelaySeconds: 20
  periodSeconds: 10
```

Probe가 실패하면 Pod는 실행 중이더라도 Service Endpoint에서 제외됩니다.

---

## 13. Liveness Probe

Liveness Probe는 애플리케이션이 비정상 상태에 빠졌는지 판단합니다.

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
  initialDelaySeconds: 40
  periodSeconds: 20
```

연속 실패하면 Kubernetes가 컨테이너를 재시작합니다.

---

## 14. Startup Probe 권장

Spring Boot 시작 시간이 길면 Startup Probe를 추가할 수 있습니다.

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8081
  failureThreshold: 30
  periodSeconds: 10
```

Startup Probe가 성공하기 전에는 Liveness Probe가 애플리케이션을 조기에 재시작하지 않습니다.

---

## 15. Resource Requests와 Limits

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

| 구분 | 역할 |
|---|---|
| Requests | Scheduler가 Node 배치에 사용하는 최소 요구량 |
| Limits | 컨테이너가 사용할 수 있는 최대량 |

실제 모니터링 데이터를 기준으로 조정합니다.

---

## 16. NodeSelector 구성

서비스용 Node에만 Pod를 배치하기 위해 다음 설정을 사용합니다.

```yaml
nodeSelector:
  role: service
```

Node Label 확인:

```bash
kubectl get nodes --show-labels
```

Label 추가:

```bash
kubectl label node {node-name} role=service
```

Label이 없으면 Pod가 `Pending` 상태에 머물 수 있습니다.

---

## 17. Kustomize 구성

`kustomization.yaml` 예시:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - configmap.yaml
  - admin-service.yaml
  - auth-service.yaml
  - contest-service.yaml
  - market-service.yaml
  - order-service.yaml
```

실제 Secret은 Git에서 제외하므로 `kustomization.yaml`에 포함하지 않고 별도로 적용할 수 있습니다.

---

## 18. 적용 순서

### 18.1 Namespace 적용

```bash
kubectl apply \
  -f infra/k8s/api/namespace.yaml
```

### 18.2 Secret 적용

```bash
kubectl apply \
  -f /tmp/maesoongan-api-secret.yaml
```

### 18.3 Kustomize 결과 확인

```bash
kubectl kustomize infra/k8s/api
```

### 18.4 전체 리소스 적용

```bash
kubectl apply -k infra/k8s/api
```

---

## 19. 상태 확인

```bash
kubectl get pods -n app
kubectl get deployments -n app
kubectl get services -n app
kubectl get endpoints -n app
kubectl get pods -n app -o wide
```

정상 상태:

```text
STATUS: Running
READY: 1/1
```

---

## 20. 서비스 간 통신 확인

임시 Pod를 사용해 DNS와 Port를 확인할 수 있습니다.

```bash
kubectl run curl-test \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  -n app \
  -- curl http://auth-service:8081/actuator/health
```

확인 항목:

- Kubernetes DNS
- Service Selector
- Endpoint 생성
- Target Port
- 애플리케이션 응답

---

## 21. 트러블슈팅

### 21.1 `Pending`

확인 항목:

- NodeSelector
- Node Resource
- Taint와 Toleration
- PVC
- Node Group 상태

```bash
kubectl describe pod -n app {pod-name}
```

### 21.2 `ImagePullBackOff`

확인 항목:

- ECR 이미지 주소
- 이미지 태그
- EKS ECR Pull 권한
- Region
- Account

### 21.3 `CrashLoopBackOff`

```bash
kubectl logs -n app {pod-name}
kubectl logs -n app {pod-name} --previous
```

확인 항목:

- 환경변수
- DB 연결
- Redis 연결
- JWT Secret
- Port
- Spring Profile

### 21.4 `CreateContainerConfigError`

확인 항목:

- ConfigMap 존재 여부
- Secret 존재 여부
- Key 이름
- Namespace

### 21.5 Endpoint가 비어 있음

```bash
kubectl get endpoints -n app
```

확인 항목:

- Service Selector와 Pod Label 일치
- Pod Readiness 상태
- Service Port와 Target Port

---

## 22. 운영 체크리스트

- [ ] 모든 리소스의 Namespace가 `app`인가
- [ ] Deployment Selector와 Pod Label이 일치하는가
- [ ] Service Selector가 Pod Label과 일치하는가
- [ ] Container Port와 Target Port가 일치하는가
- [ ] 실제 Secret이 Git에서 제외됐는가
- [ ] Readiness/Liveness Probe가 실제 Endpoint와 일치하는가
- [ ] Node에 `role=service` Label이 있는가
- [ ] Resource Requests와 Limits가 설정됐는가
- [ ] 운영 이미지가 고정 태그를 사용하는가
