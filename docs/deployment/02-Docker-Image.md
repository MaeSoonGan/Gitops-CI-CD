# 서비스별 Docker 이미지 구성

## 1. 개요 및 프로젝트 내 역할

MaeSoonGan의 각 Spring Boot 서비스는 독립적인 Docker 이미지로 패키징됩니다.

서비스별 이미지를 분리하면 다음과 같은 장점이 있습니다.

- 변경된 서비스만 독립적으로 재배포 가능
- 서비스별 확장 및 축소 가능
- 장애 영향 범위 분리
- 이미지 버전과 배포 이력 개별 관리
- Kubernetes Deployment와 일대일로 연결 가능

---

## 2. Dockerfile 작성 대상

```text
apps/api/admin-service/Dockerfile
apps/api/auth-service/Dockerfile
apps/api/contest-service/Dockerfile
apps/api/market-service/Dockerfile
apps/api/order-service/Dockerfile
```

서비스별 Dockerfile의 구조는 동일하며, 다음 항목만 서비스에 맞게 변경합니다.

- Gradle `bootJar` Task 경로
- JAR 파일 경로
- 서버 Port
- 필요한 실행 환경변수

---

## 3. Multi-stage Build 구조

Docker 이미지는 Build Stage와 Runtime Stage로 구분합니다.

```text
Build Stage
    ↓
JDK 환경에서 Gradle bootJar 실행
    ↓
실행 가능한 JAR 생성
    ↓
Runtime Stage
    ↓
JRE 환경으로 JAR 복사
    ↓
java -jar app.jar 실행
```

### 적용 이유

- 최종 이미지에 Gradle과 소스코드를 포함하지 않음
- 최종 이미지 크기 감소
- 공격 표면 축소
- 빌드 환경과 실행 환경 분리
- 이미지 관리 효율 향상

---

## 4. Dockerfile 기본 구조

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

COPY --from=build \
  /app/apps/api/auth-service/build/libs/*.jar \
  app.jar

ENV AUTH_SERVICE_PORT=8081

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 5. Build Stage 설명

### 5.1 Base Image

```dockerfile
FROM eclipse-temurin:17-jdk AS build
```

Gradle Build를 수행해야 하므로 JDK 이미지를 사용합니다.

### 5.2 작업 디렉터리

```dockerfile
WORKDIR /app
```

이후 모든 명령은 `/app`을 기준으로 실행됩니다.

### 5.3 Gradle 관련 파일 복사

```dockerfile
COPY gradlew .
COPY gradle ./gradle
COPY settings.gradle .
COPY build.gradle .
COPY apps ./apps
```

멀티 프로젝트 전체 구성을 Gradle이 읽을 수 있도록 `apps` 디렉터리를 함께 복사합니다.

### 5.4 실행 권한 부여

```dockerfile
RUN chmod +x gradlew
```

Linux 기반 Docker Build 환경에서 Gradle Wrapper를 실행할 수 있도록 권한을 부여합니다.

### 5.5 서비스별 JAR 빌드

```dockerfile
RUN ./gradlew :apps:api:auth-service:bootJar --no-daemon
```

해당 서비스의 실행 가능한 Spring Boot JAR를 생성합니다.

---

## 6. Runtime Stage 설명

### 6.1 Runtime Base Image

```dockerfile
FROM eclipse-temurin:17-jre
```

JAR 실행만 필요하므로 JDK보다 가벼운 JRE 이미지를 사용합니다.

### 6.2 JAR 복사

```dockerfile
COPY --from=build \
  /app/apps/api/auth-service/build/libs/*.jar \
  app.jar
```

Build Stage에서 생성된 JAR만 Runtime Stage로 가져옵니다.

### 6.3 Port 설정

```dockerfile
ENV AUTH_SERVICE_PORT=8081
EXPOSE 8081
```

`EXPOSE`는 컨테이너가 사용하는 Port를 문서화합니다. 실제 Port 바인딩과 Kubernetes Service 연결은 별도로 설정해야 합니다.

### 6.4 실행 명령

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

컨테이너 시작 시 Spring Boot 애플리케이션을 실행합니다.

---

## 7. 서비스별 설정

아래 값은 실제 `application.yml`, 환경변수, Kubernetes Manifest와 일치하도록 관리합니다.

| 서비스 | Gradle Task | JAR 경로 | Port |
|---|---|---|---:|
| `admin-service` | `:apps:api:admin-service:bootJar` | `apps/api/admin-service/build/libs/*.jar` | 실제 설정값 |
| `auth-service` | `:apps:api:auth-service:bootJar` | `apps/api/auth-service/build/libs/*.jar` | 8081 |
| `contest-service` | `:apps:api:contest-service:bootJar` | `apps/api/contest-service/build/libs/*.jar` | 실제 설정값 |
| `market-service` | `:apps:api:market-service:bootJar` | `apps/api/market-service/build/libs/*.jar` | 실제 설정값 |
| `order-service` | `:apps:api:order-service:bootJar` | `apps/api/order-service/build/libs/*.jar` | 8084 |

Port 값은 프로젝트 실제 설정에 맞춰 수정합니다.

---

## 8. Docker Build Context

Docker Build는 Repository Root에서 실행합니다.

```bash
docker build \
  -f apps/api/auth-service/Dockerfile \
  -t auth-service:latest \
  .
```

마지막의 `.`은 현재 디렉터리를 Build Context로 사용한다는 의미입니다.

서비스 디렉터리 내부에서 Build하면 다음 파일을 찾지 못할 수 있습니다.

```text
gradlew
gradle/
settings.gradle
build.gradle
다른 멀티 프로젝트 디렉터리
```

---

## 9. 로컬 빌드 및 실행

### 9.1 이미지 빌드

```bash
docker build \
  -f apps/api/auth-service/Dockerfile \
  -t auth-service:local \
  .
```

### 9.2 이미지 확인

```bash
docker images
```

### 9.3 컨테이너 실행

```bash
docker run --rm \
  -p 8081:8081 \
  --env-file .env.local \
  auth-service:local
```

### 9.4 로그 확인

```bash
docker logs {container-name-or-id}
```

### 9.5 컨테이너 내부 확인

```bash
docker exec -it {container-name-or-id} sh
```

---

## 10. 환경변수 주입

민감정보를 Dockerfile에 직접 작성하지 않습니다.

잘못된 예시:

```dockerfile
ENV DB_PASSWORD=real-password
ENV JWT_SECRET=real-secret
```

권장 방식:

```bash
docker run \
  -e SPRING_DATASOURCE_URL="..." \
  -e SPRING_DATASOURCE_USERNAME="..." \
  -e SPRING_DATASOURCE_PASSWORD="..." \
  auth-service:local
```

Kubernetes 환경에서는 ConfigMap과 Secret을 통해 주입합니다.

---

## 11. 이미지 태그 규칙

```text
{ECR_REGISTRY}/maesoongan/{service-name}:latest
{ECR_REGISTRY}/maesoongan/{service-name}:{GITHUB_SHA}
```

운영 배포에서는 Commit SHA 또는 Release Version 태그를 사용합니다.

```text
auth-service:3f2a9c...
auth-service:v1.0.0
```

---

## 12. `.dockerignore` 권장 설정

Docker Build Context에 불필요한 파일이 포함되지 않도록 `.dockerignore`를 구성합니다.

```text
.git
.github
.idea
.vscode
**/build
**/.gradle
*.log
.env
.env.*
secret.yaml
*.secret.yaml
node_modules
```

단, Build 과정에 필요한 파일까지 제외하지 않도록 주의합니다.

---

## 13. Health Check 고려사항

Dockerfile의 `HEALTHCHECK`보다 Kubernetes의 Readiness Probe와 Liveness Probe를 중심으로 상태를 관리합니다.

```yaml
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
```

Spring Boot Actuator 설정과 Endpoint 공개 범위를 함께 확인해야 합니다.

---

## 14. 트러블슈팅

### 14.1 `exit code: 126`

```text
process "/bin/sh -c ./gradlew ..." did not complete successfully: exit code: 126
```

해결:

```dockerfile
RUN chmod +x gradlew
```

### 14.2 멀티 프로젝트 디렉터리 오류

```text
Configuring project ':apps:worker' without an existing directory is not allowed.
```

해결:

```dockerfile
COPY apps ./apps
```

### 14.3 JAR 파일을 찾지 못함

```text
COPY failed: no source files were specified
```

확인할 항목:

- Gradle Task 경로
- 서비스 디렉터리명
- `build/libs` 경로
- JAR 이름
- `bootJar` 실행 성공 여부

### 14.4 Port 불일치

다음 값이 모두 일치해야 합니다.

```text
Spring Boot server.port
Dockerfile EXPOSE
Kubernetes containerPort
Kubernetes Service targetPort
Readiness/Liveness Probe Port
```

### 14.5 이미지 크기 증가

확인할 항목:

- JDK가 Runtime Stage에 사용되었는지
- 소스 전체가 Runtime Stage에 복사되었는지
- Build 결과 외 불필요한 파일이 포함되었는지
- `.dockerignore`가 구성되었는지

---

## 15. 개선 방향

- Gradle Dependency Layer Cache 적용
- Docker BuildKit Cache 적용
- Distroless 또는 더 작은 Runtime Image 검토
- Non-root User로 실행
- 이미지 취약점 스캔
- SBOM 생성
- 이미지 서명 및 Provenance 적용
