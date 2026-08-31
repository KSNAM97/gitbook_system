# Docker - Docker Compose

> **Tag:** #Docker #DockerCompose #YAML #멀티컨테이너 #부트캠프
> **핵심 요약:** 여러 컨테이너를 하나의 YAML 파일로 관리하는 Compose 사용법 핵심 정리

---

## 1. 개요 (Overview)

Docker Compose는 여러 개의 컨테이너를 하나의 `docker-compose.yaml` 파일로 정의하고 한 번에 실행/중지하는 도구이다. 기존 방식에서는 컨테이너마다 `docker run` 명령을 따로 실행하고 실행 순서를 직접 관리하며 네트워크와 볼륨도 수동으로 설정해야 했지만, Compose 방식에서는 `docker compose up -d` 한 번으로 전체를 실행하고, `depends_on`으로 실행 순서를 자동 관리하며, 네트워크와 볼륨도 자동으로 생성된다.

docker-compose.yaml의 주요 옵션은 다음과 같다.

#### version
```yaml
version: "3.9"    # 최신 Compose에서는 생략 가능
```

#### services (컨테이너 정의)
```yaml
services:
  webserver:
    image: nginx
  db:
    image: mysql:8.0
```

#### build (Dockerfile로 이미지 빌드)
```yaml
services:
  webapp:
    build:
      context: .               # 빌드 컨텍스트 (현재 디렉터리)
      dockerfile: Dockerfile.dev
```

#### image (기존 이미지 사용)
```yaml
services:
  webapp:
    image: nginx:latest
```

#### command (시작 명령 지정)
```yaml
services:
  app:
    image: node:18
    command: sh -c "npm install && npm start"
```

#### ports (포트 포워딩)
```yaml
services:
  webapp:
    ports:
      - "80:80"
      - "443:443"
```

#### expose (내부 포트만 공개)
```yaml
services:
  app:
    expose:
      - "8080"    # 외부 노출 안 됨, 같은 네트워크 내에서만 접근
```

#### volumes (볼륨 마운트)
```yaml
services:
  webapp:
    volumes:
      - ./html:/usr/local/apache2/htdocs    # Bind Mount
      - dbdata:/var/lib/mysql               # Named Volume

volumes:
  dbdata:    # Named Volume 선언
```

#### environment (환경 변수)
```yaml
services:
  database:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: pass
      MYSQL_DATABASE: appdb
      TZ: Asia/Seoul
```

#### restart (재시작 정책)
```yaml
services:
  database:
    restart: always    # no | always | on-failure | unless-stopped
```

| 값 | 설명 |
|---|---|
| `no` | 기본값, 자동 재시작 없음 |
| `always` | 항상 재시작 |
| `on-failure` | 비정상 종료 시만 재시작 |
| `unless-stopped` | 사용자가 stop하면 멈춤, 그 외 항상 재시작 |

#### depends_on (실행 순서)
```yaml
services:
  web:
    depends_on:
      - db    # db 컨테이너가 먼저 실행된 후 web 실행
  db:
    image: mysql:8.0
```

#### depends_on + healthcheck (DB 준비 완료 후 실행)
```yaml
services:
  db:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 20

  web:
    image: php:8.2-apache
    depends_on:
      db:
        condition: service_healthy    # DB가 healthy 상태일 때 시작
```

#### networks (사용자 정의 네트워크)
```yaml
networks:
  webnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1

services:
  web:
    networks:
      - webnet
  db:
    networks:
      - webnet
```

Docker Compose의 주요 명령어는 다음과 같다.

| 명령어 | 설명 |
|---|---|
| `docker compose up` | 컨테이너 생성 및 시작 |
| `docker compose up -d` | 백그라운드로 실행 |
| `docker compose up --build` | 이미지 빌드 후 실행 |
| `docker compose down` | 컨테이너 + 네트워크 삭제 |
| `docker compose down -v` | 볼륨까지 삭제 |
| `docker compose ps` | 컨테이너 목록 확인 |
| `docker compose ls` | Compose 프로젝트 목록 |
| `docker compose logs` | 전체 로그 확인 |
| `docker compose logs 서비스명` | 특정 서비스 로그 |
| `docker compose build` | 이미지만 빌드 |
| `docker compose config` | Compose 설정 유효성 확인 |
| `docker compose stop` | 컨테이너 중지 (삭제 안 함) |
| `docker compose start` | 중지된 컨테이너 재시작 |
| `docker compose restart` | 컨테이너 재시작 |

```bash
# 파일명이 기본값이 아닐 때 -f 옵션 사용
docker compose -f docker-compose-httpd.yaml up -d
docker compose -f docker-compose-httpd.yaml ps
docker compose -f docker-compose-httpd.yaml down
```

**기본 파일명 우선순위:**
1. `compose.yaml`
2. `compose.yml`
3. `docker-compose.yaml`
4. `docker-compose.yml`

Named Volume의 이름 규칙은 `프로젝트이름_볼륨이름` 형식으로 자동 지정된다.

```
프로젝트이름_볼륨이름

예) 디렉터리명: step1-mysql-pull
    볼륨 이름: dbdata
    → 실제 볼륨명: step1-mysql-pull_dbdata
    → 경로: /var/lib/docker/volumes/step1-mysql-pull_dbdata/_data
```

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** Docker Engine (Compose V2 내장) 설치 환경.

### Reverse Proxy 구성 예시 (Nginx)

```yaml
# /compose-lab/step3-1/docker-compose.yaml
services:
  app1:
    build:
      context: ./app1
      dockerfile: dockerfile
    image: step3-1-app1
    container_name: step3-1-app1

  app2:
    build:
      context: ./app2
      dockerfile: dockerfile
    image: step3-1-app2
    container_name: step3-1-app2

  proxy:
    build:
      context: ./proxy
      dockerfile: dockerfile
    image: step3-1-proxy
    container_name: step3-1-proxy
    ports:
      - "80:80"
    depends_on:
      - app1
      - app2
```

**Nginx Reverse Proxy 설정 (라운드 로빈):**
```nginx
upstream backend {
    server app1;    # 컨테이너 이름 = DNS 호스트명
    server app2;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

**로드밸런싱 알고리즘:**
```nginx
upstream backend {
    # 기본: round-robin (순차)
    server app1;
    server app2;
    
    # 또는 최소 연결
    # least_conn;
    
    # 또는 IP 기반 (세션 유지)
    # ip_hash;
}
```

### MySQL + phpMyAdmin Compose 예시

```yaml
services:
  db:
    image: mysql:8.0
    container_name: step2-mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: "admin1234"
      MYSQL_DATABASE: "sampledb"
      MYSQL_USER: "testuser"
      MYSQL_PASSWORD: "1234"
    ports:
      - "3308:3306"
    volumes:
      - "mydbdata:/var/lib/mysql"

  pma:
    image: phpmyadmin/phpmyadmin:latest
    container_name: step2-mysql-pma
    environment:
      PMA_HOST: db          # db 컨테이너 이름 = DNS 호스트명
      PMA_PORT: 3306
      PMA_USER: root
      PMA_PASSWORD: "admin1234"
    ports:
      - "8086:80"
    depends_on:
      - db

volumes:
  mydbdata:
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### Docker Compose 설치 방법 (Rocky Linux)

```bash
# 1. 기존 패키지 제거 (선택)
sudo dnf remove docker docker-client ...

# 2. DNF 플러그인 설치
sudo dnf -y install dnf-plugins-core

# 3. Docker 공식 저장소 추가
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# 4. Docker Engine + Compose 플러그인 설치
sudo dnf install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# 5. 서비스 시작
sudo systemctl enable --now docker

# 6. 버전 확인
docker compose version
```

---

>  **핵심 요약**
> - Docker Compose = 멀티 컨테이너를 단일 YAML 파일로 정의·실행하는 도구
> - `docker compose up -d` / `docker compose down` — 전체 환경 시작/삭제
> - `depends_on + healthcheck` — DB가 실제 준비 완료된 후 앱 시작 보장
> - Named Volume: `프로젝트이름_볼륨이름` 형식으로 자동 생성
> - Reverse Proxy + 로드밸런싱: Nginx upstream 블록에 컨테이너 이름으로 설정
> - 관련: 7.  Docker - YAML 문법 · 6.  Docker - 컨테이너 네트워크 · 10.  Docker - 트러블슈팅 치트시트
