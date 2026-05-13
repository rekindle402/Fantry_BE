# 인프라·배포·로깅 환경 구성

## 1. 문제 상황

팀 프로젝트에서 로컬 실행만으로는 실제 요청 흐름과 운영 환경을 검증하기 어려웠습니다.

또한 수동 배포는 빌드, 코드 반영, JAR 교체, 애플리케이션 재시작 과정에서 실수 가능성이 있었고, 배포 후 오류 확인을 위해 서버에 직접 접속해 로그를 확인해야 했습니다.

팀원이 공통 개발 서버와 DB에 접근하는 과정도 개인별 설정에 의존하면 반복적인 안내와 설정 오류가 발생할 수 있다고 판단했습니다.

---

## 2. 구성 목표

- 외부 요청이 서버 애플리케이션까지 도달하는 흐름 구성
- Dev / Prod 환경 분리
- 배포 과정 자동화
- 애플리케이션 실행 및 재시작 관리 표준화
- 배포 후 로그 확인 흐름 개선
- 팀원별 서버 접속 및 DB 접근 절차 표준화

---

## 3. 전체 구성

| 구분 | 구성 |
|---|---|
| 서버 | Mini PC + Ubuntu 24.04 LTS |
| 요청 처리 | DDNS, Port Forwarding, Nginx Reverse Proxy |
| 애플리케이션 실행 | Spring Boot JAR, systemd |
| 배포 자동화 | GitHub Actions, Shell Script, self-hosted runner |
| 환경 설정 | Spring Profile, GitHub Secrets, env 파일 |
| DB / Cache | MySQL, Redis |
| 로그 확인 | Promtail, Loki, Grafana |
| 팀 개발 환경 | Linux 계정, SSH Key, DB/Redis SSH Tunneling, bat script |
| DB 변경 관리 | Flyway |

---

## 4. 요청 처리 구조 - Nginx

Nginx를 외부 요청의 진입점으로 두고, 요청 도메인과 경로에 따라 Production, Development, Grafana 서버로 분기했습니다.

### 구성 의도

- 외부 요청 진입점 단일화
- Dev / Prod 요청 분리
- API, WebSocket, SSE 요청 처리
- 정적 파일 업로드 경로 제공
- Vue 기반 SPA 라우팅 처리
- Grafana 로그 확인 화면 접근 경로 제공

### 주요 분기

| 요청 | 처리 |
|---|---|
| `prod.example.com` | Production Frontend / Backend |
| `dev.example.com` | Development Frontend / Backend |
| `log.example.com` | Grafana |
| `/api`, `/webhooks` 등 API 요청 | Spring Boot API 서버로 proxy |
| WebSocket 요청 | Spring Boot WebSocket endpoint로 proxy |
| SSE 요청 | Spring Boot SSE endpoint로 proxy |
| `/static/` | 업로드 파일 정적 서빙 |
| 그 외 요청 | Vue SPA의 `index.html`로 fallback |

### Nginx 설정 예시

```nginx
server {
    listen 80;
    server_name prod.example.com;

    client_max_body_size 20m;

    location /api/ {
        proxy_pass http://127.0.0.1:{PROD_BACKEND_PORT};

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /path/to/uploads/prod/;
    }

    location / {
        root /path/to/frontend/prod;
        try_files $uri $uri/ /index.html;
    }
}
```

### 선택 이유

Spring Boot 내장 Tomcat에 직접 외부 요청을 연결하기보다, Nginx를 앞단에 두어 요청 진입점을 분리하고 Dev / Prod 요청을 명확히 나누고자 했습니다.

또한 이후 HTTPS 적용, 정적 파일 서빙, WebSocket/SSE 처리, 프론트엔드 SPA 라우팅 확장까지 고려했을 때 Nginx가 적절하다고 판단했습니다.

---

## 5. 배포 자동화 - GitHub Actions / Shell Script

GitHub Actions에서 브랜치 push 이벤트를 기준으로 배포 스크립트를 실행하도록 구성했습니다.

| 브랜치 | 배포 환경 |
|---|---|
| `develop` | dev 서버 |
| `main` | prod 서버 |

### 배포 흐름

1. GitHub Actions workflow 실행
2. self-hosted runner에서 배포 Shell Script 실행
3. 서버의 프로젝트 디렉토리에서 대상 브랜치 최신 코드 동기화
4. GitHub Secrets 값을 서버 env 파일로 생성
5. Gradle build 실행
6. 기존 systemd 서비스 중지
7. 새 JAR 파일 배포 경로로 복사
8. systemd 서비스 재시작

### GitHub Actions 예시

```yaml
name: Deploy Backend

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: self-hosted

    env:
      SECRET_DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}
      SECRET_REDIS_PASSWORD: ${{ secrets.PROD_REDIS_PASSWORD }}
      SECRET_JWT_SECRET_KEY: ${{ secrets.JWT_SECRET_KEY }}

    steps:
      - name: Execute backend deployment script
        run: ~/scripts/deploy-backend-prod.sh
```

### Shell Script 처리 흐름 예시

```bash
#!/bin/bash

set -e

BRANCH_NAME="main"
JAR_NAME="app.jar"
SERVICE_NAME="app-prod.service"
ENV_FILE_PATH="/etc/app/app-prod.env"
ENV_FILE_TMP_PATH="/tmp/app-prod.env"

PROJECT_PATH="/path/to/source/backend"
DEPLOY_PATH="/path/to/deploy/prod"

cd "$PROJECT_PATH"

git fetch origin "$BRANCH_NAME"
git reset --hard origin/"$BRANCH_NAME"

> "$ENV_FILE_TMP_PATH"

env | grep '^SECRET_' | while IFS= read -r var; do
  key=$(echo "$var" | sed -e 's/^SECRET_//' | cut -d= -f1)
  value=$(echo "$var" | cut -d= -f2-)
  echo "$key=\"$value\"" >> "$ENV_FILE_TMP_PATH"
done

echo "DB_URL=\"jdbc:mysql://localhost:{MYSQL_PORT}/{DB_NAME}\"" >> "$ENV_FILE_TMP_PATH"
echo "DB_USER=\"{DB_USER}\"" >> "$ENV_FILE_TMP_PATH"
echo "REDIS_DATABASE={REDIS_DATABASE_INDEX}" >> "$ENV_FILE_TMP_PATH"

sudo mv "$ENV_FILE_TMP_PATH" "$ENV_FILE_PATH"
sudo chown root:{DEPLOY_GROUP} "$ENV_FILE_PATH"
sudo chmod 640 "$ENV_FILE_PATH"

chmod +x ./gradlew
source "$ENV_FILE_PATH"

./gradlew clean build -x test

sudo systemctl stop "$SERVICE_NAME"
cp build/libs/*-SNAPSHOT.jar "$DEPLOY_PATH/$JAR_NAME"
sudo systemctl start "$SERVICE_NAME"
```

### 선택 이유

GitHub Actions는 GitHub 저장소와 자연스럽게 연동되고, 별도 CI 서버를 운영하지 않아도 push 기반 배포 자동화를 구성할 수 있었습니다.

프로젝트 규모상 Jenkins 서버를 별도로 운영하는 것은 오버엔지니어링이라고 판단했고, GitHub Actions와 self-hosted runner만으로 팀 프로젝트에 필요한 배포 자동화 요구를 충족할 수 있다고 보았습니다.

---

## 6. 애플리케이션 실행 관리 - systemd

Spring Boot 애플리케이션은 dev/prod 각각 별도 systemd service로 실행했습니다.

| 환경 | service | profile | env file |
|---|---|---|---|
| dev | `app-dev.service` | `dev` | `/etc/app/app-dev.env` |
| prod | `app-prod.service` | `prod` | `/etc/app/app-prod.env` |

### 구성 의도

단순히 `nohup`으로 JAR를 실행하면 프로세스 상태 확인, 재시작, 장애 시 복구 흐름이 불명확하다고 판단했습니다.

systemd를 사용해 애플리케이션 실행 방식을 서비스 단위로 관리하고, 실패 시 재시작 정책을 적용했습니다.

### systemd 설정 예시

```ini
[Unit]
Description=Application Prod Server
After=network.target

[Service]
EnvironmentFile=/etc/app/app-prod.env
User={APP_USER}
Group={APP_GROUP}

WorkingDirectory=/path/to/deploy/prod

ExecStart=java -jar -Dspring.profiles.active=prod /path/to/deploy/prod/app.jar

SuccessExitStatus=143
TimeoutStopSec=10
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 주요 설정

- `EnvironmentFile`로 환경 변수 파일 분리
- `-Dspring.profiles.active`로 실행 profile 지정
- `Restart=on-failure`로 실패 시 재시작
- dev/prod JAR 배포 경로 분리

---

## 7. 환경 설정 분리 - Spring Profile / GitHub Secrets / env

환경별 설정은 Spring Profile과 env 파일로 분리했습니다.

### dev / prod 분리 항목

- DB URL
- DB 계정
- Redis database index
- 로그 파일 경로
- 서버 포트
- CORS 허용 주소
- Actuator 노출 범위
- Swagger UI 활성화 여부
- 파일 업로드 경로
- 외부 API Key

### properties 예시

```properties
# Database
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}

# Redis
spring.data.redis.password=${REDIS_PASSWORD}
spring.data.redis.database=${REDIS_DATABASE}

# Logging
logging.file.name=/path/to/logs/app-prod.log

# Server
server.port={PROD_BACKEND_PORT}

# Actuator
management.endpoints.web.exposure.include=health,info,prometheus
management.endpoint.health.show-details=never

# Swagger
springdoc.swagger-ui.enabled=false
springdoc.api-docs.enabled=false

# File Upload
file.upload-dir=/path/to/uploads/prod
file.base-url=http://prod.example.com/static/

# Security
spring.jwt.secret=${JWT_SECRET_KEY}
```

### GitHub Secrets 활용

DB 비밀번호, Redis 비밀번호, JWT Secret, 외부 API Key, Mail Password 등 민감 정보는 GitHub Secrets로 관리했습니다.

배포 시 GitHub Actions가 `SECRET_` prefix를 가진 환경 변수를 서버로 전달하고, 배포 스크립트가 이를 env 파일로 생성하도록 구성했습니다.

### 선택 이유

민감 정보를 코드나 properties 파일에 직접 작성하지 않고, 환경별로 분리해 관리하기 위해 GitHub Secrets와 env 파일을 함께 사용했습니다.

---

## 8. 로그 확인 환경 - Promtail / Loki / Grafana

Spring Boot 애플리케이션 로그를 파일로 남기고, Promtail을 통해 Loki로 수집한 뒤 Grafana에서 확인할 수 있도록 구성했습니다.

### 구성 목적

- 배포 후 오류 로그 확인 경로 통일
- 서버에 직접 접속해 로그 파일을 확인하는 불편 감소
- dev/prod 로그 확인 흐름 개선
- 장애 원인 파악을 위한 로그 조회 기반 마련

### Loki systemd 설정 예시

```ini
[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/loki -config.file /etc/loki/loki-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 선택 이유

ELK Stack은 검색 기능이 강하지만, 팀 프로젝트 규모에서는 구축과 운영 비용이 크다고 판단했습니다.

Loki는 Grafana와 함께 사용하기 쉽고, 파일 로그 수집 및 조회 목적에는 충분하다고 보아 선택했습니다.

---

## 9. 팀 개발 환경 표준화

팀원이 공통 개발 서버와 DB에 접근할 수 있도록 서버 계정, SSH Key, DB/Redis 터널링 절차를 정리했습니다.

### 구성 내용

- 팀원별 Linux 계정 생성
- SSH Key 기반 서버 접속
- MySQL / Redis SSH Tunneling 구성
- 터널링 실행 bat 파일 배포
- Spring Profile과 properties 예시 제공
- Flyway 기반 DB 변경 이력 관리

### DB / Redis 터널링 bat 예시

```bat
@echo off
echo SSH tunneling start...

ssh ^
  -L {WINDOWS_MYSQL_PORT}:127.0.0.1:{MYSQL_PORT} ^
  -L {WINDOWS_REDIS_PORT}:127.0.0.1:{REDIS_PORT} ^
  {USERNAME}@prod.example.com ^
  -p {SSH_PORT}

pause
```

### 구성 의도

DB와 Redis를 외부에 직접 노출하지 않고, SSH 터널링을 통해 필요한 팀원만 접근할 수 있도록 구성했습니다.

또한 bat 파일을 제공해 팀원이 복잡한 SSH 터널링 명령어를 매번 직접 입력하지 않아도 되도록 했습니다.

---

## 10. 결과

| 항목 | 결과 |
|---|---|
| 배포 자동화 | 수동 배포 시간 약 10분 → 2~3분 단축 |
| 환경 분리 | Dev / Prod 설정, 민감 정보, 로그 파일 경로 분리 |
| 실행 관리 | systemd 기반 애플리케이션 실행·재시작 관리 |
| 요청 분리 | Nginx 기반 Prod / Dev / Grafana 요청 분리 |
| 로그 확인 | Grafana 기반 배포 후 오류 로그 확인 흐름 구성 |
| 팀 개발 환경 | 팀원별 서버 접속 및 DB 터널링 절차 표준화 |
| DB 변경 관리 | Flyway 기반 DB 변경 이력 관리 |

---

## 11. 개선 가능점

### HTTPS 미적용

현재는 HTTP 기반으로 구성했습니다. 실제 운영 환경이라면 SSL/TLS 인증서를 적용해 HTTPS로 전환해야 합니다.

### 단일 서버 구조

Mini PC 단일 서버에 애플리케이션, DB, Redis, 로그 도구가 함께 구성되어 있어 서버 장애 시 전체 서비스에 영향이 발생할 수 있습니다. 실제 서비스 환경이라면 클라우드 인프라, 백업, 장애 복구, 이중화 구성이 필요합니다.

### 무중단 배포 미적용

systemd restart 기반 배포이므로 재시작 중 짧은 중단 시간이 발생할 수 있습니다. 실제 운영 환경이라면 Blue-Green 배포, Rolling 배포, Load Balancer 기반 무중단 배포를 고려할 수 있습니다.

### 배포 로그 민감 정보 노출 주의

초기 디버깅 과정에서 env 파일 내용을 출력하는 코드가 포함될 수 있습니다. 실제 운영 환경에서는 GitHub Actions 로그에 민감 정보가 노출되지 않도록 해당 디버깅 코드는 제거해야 합니다.

---

## 12. 관련 파일

| 구분 | 파일 |
|---|---|
| GitHub Actions Workflow | `.github/workflows/...` |
| Backend dev 배포 스크립트 | `deploy-backend-dev.sh` |
| Backend prod 배포 스크립트 | `deploy-backend-prod.sh` |
| systemd dev service | `app-dev.service` |
| systemd prod service | `app-prod.service` |
| Nginx 설정 | `sites-enabled/app` |
| Spring properties | `application-dev.properties`, `application-prod.properties`, `application-local.properties` |

---

## 13. 보안상 비공개 처리한 정보

본 문서에서는 현재 운영 중인 서버 보호를 위해 아래 정보는 예시 값으로 대체했습니다.

- 실제 DDNS 주소
- 실제 SSH 포트
- 실제 DB / Redis 포트
- 실제 Linux 사용자명
- 실제 서버 내부 경로
- 실제 DB 이름과 계정명
- 실제 외부 API Key
- 실제 이메일 계정
- GitHub Secrets 값
