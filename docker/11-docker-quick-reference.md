# Docker - 퀵 레퍼런스

> **Tag:** #Docker #퀵레퍼런스 #치트시트 #명령어 #부트캠프
> **핵심 요약:** 코딩·운영 중 바로 찾아보는 Docker 명령어 전체 패턴 모음

---

## 1. 핵심 기술 개념 (Concept)

### 자주 헷갈리는 항목

| 항목 | 설명 |
|---|---|
| `docker stop` vs `docker kill` | stop: SIGTERM(정상 종료) / kill: SIGKILL(강제 종료) |
| `docker rm` vs `docker rmi` | rm: 컨테이너 삭제 / rmi: 이미지 삭제 |
| `docker run` vs `docker start` | run: 이미지로 새 컨테이너 생성+실행 / start: 기존 컨테이너 재시작 |
| `CMD` vs `ENTRYPOINT` | CMD: docker run 시 덮어쓰기 가능 / ENTRYPOINT: 고정 |
| `-p 80:80` vs `-P` | -p: 직접 지정 / -P: EXPOSE 자동 매핑 |
| `COPY` vs `ADD` | COPY: 단순 복사 / ADD: URL+압축 해제 지원 |
| Bind Mount vs Named Volume | Bind: 호스트 경로 직접 / Named: Docker 관리 볼륨 |
| `docker compose down` vs `stop` | down: 컨테이너+네트워크 삭제 / stop: 중지만 |

---

## 2. 표준 설정 템플릿 (Configuration)

### Dockerfile 기본 템플릿

```dockerfile
# Nginx 기반
FROM nginx:latest
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8
COPY ./html /usr/share/nginx/html
EXPOSE 80
```

```dockerfile
# Node.js 기반
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

```dockerfile
# PHP + Apache 기반
FROM php:8.2-apache
RUN docker-php-ext-install mysqli
COPY . /var/www/html/
EXPOSE 80
```

### Docker Compose 기본 템플릿

```yaml
services:
  web:
    image: nginx:latest
    container_name: my_web
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    restart: always
    networks:
      - webnet
    depends_on:
      - db   # db가 먼저 실행된 후 web 실행

  db:
    image: mysql:8.0
    container_name: my_db
    environment:
      MYSQL_ROOT_PASSWORD: "admin1234"
      MYSQL_DATABASE: "mydb"
    ports:
      - "3306:3306"
    volumes:
      - dbdata:/var/lib/mysql
    restart: always
    networks:
      - webnet

volumes:
  dbdata:

networks:
  webnet:
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 설치 (Rocky Linux)

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
docker compose version
```

### 이미지 명령어

```bash
docker images                         # 이미지 목록
docker pull nginx:latest              # 이미지 다운로드
docker push myapp:1.0                 # 이미지 업로드
docker build -t myapp:1.0 .           # 이미지 빌드
docker rmi nginx:latest               # 이미지 삭제
docker image prune -a -f              # 미사용 이미지 일괄 삭제
docker tag myapp:1.0 user/myapp:1.0  # 이미지 태그 변경
docker inspect 이미지명               # 이미지 상세 정보
docker search nginx                   # Docker Hub 검색
docker history 이미지명               # 이미지 레이어 히스토리
```

### 컨테이너 실행 패턴

```bash
# 기본 실행
docker run nginx:latest

# 백그라운드 + 이름 + 포트 + 볼륨 + 환경변수
docker run -d \
  --name web \
  -p 8080:80 \
  -v /webdata:/usr/share/nginx/html:ro \
  -e NGINX_HOST=myserver \
  nginx:latest

# MySQL 실행
docker run -d \
  --name mysql_db \
  -p 3306:3306 \
  -v /dbdata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=admin1234 \
  -e MYSQL_DATABASE=mydb \
  mysql:latest

# 대화형 접속
docker run -it ubuntu /bin/bash

# 실행 후 자동 삭제
docker run --rm nginx:latest

# 특정 네트워크, 고정 IP
docker run -d --name web1 \
  --network webnet \
  --ip 192.168.100.10 \
  nginx:latest
```

### 컨테이너 관리 명령어

```bash
docker ps                        # 실행 중인 컨테이너
docker ps -a                     # 전체 컨테이너
docker start 컨테이너명          # 컨테이너 시작
docker stop 컨테이너명           # 컨테이너 정지
docker restart 컨테이너명        # 컨테이너 재시작
docker rm 컨테이너명             # 컨테이너 삭제
docker rm -f 컨테이너명          # 실행 중이어도 강제 삭제
docker exec -it 컨테이너명 bash  # 컨테이너 내부 접속
docker logs 컨테이너명           # 로그 확인
docker logs -f 컨테이너명        # 실시간 로그 스트림
docker inspect 컨테이너명        # 컨테이너 상세 정보
docker top 컨테이너명            # 컨테이너 내 프로세스
docker stats                     # 리소스 사용량 실시간 모니터링
docker events                    # Docker 이벤트 확인
docker pause 컨테이너명          # 일시 정지
docker unpause 컨테이너명        # 일시 정지 해제
```

### 리소스 제한 패턴

```bash
# 메모리 제한
docker run -d -m 512m nginx:latest

# 메모리 + 스왑 제한
docker run -d -m 512m --memory-swap 1g nginx:latest

# CPU 제한
docker run -d --cpus 1.5 nginx:latest

# CPU 코어 지정
docker run -d --cpuset-cpus 0,1 nginx:latest

# CPU 가중치
docker run -d -c 512 nginx:latest
```

### 볼륨(Volume) 패턴

```bash
# Bind Mount (호스트 경로)
docker run -d -v /hostpath:/containerpath nginx:latest

# Read-Only Bind Mount
docker run -d -v /hostpath:/containerpath:ro nginx:latest

# Named Volume (Docker 관리)
docker run -d -v myvolume:/containerpath nginx:latest

# 볼륨 목록
docker volume ls

# 볼륨 상세 정보
docker volume inspect myvolume

# 볼륨 삭제
docker volume rm myvolume

# 미사용 볼륨 일괄 삭제
docker volume prune
```

### 네트워크 명령어

```bash
# 네트워크 목록
docker network ls

# 네트워크 생성
docker network create webnet
docker network create --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.254 mynet

# 네트워크 상세 정보
docker network inspect webnet

# 컨테이너 네트워크 연결
docker network connect webnet web1

# 컨테이너 네트워크 해제
docker network disconnect webnet web1

# 네트워크 삭제
docker network rm webnet
```

### Docker Compose 명령어

```bash
# 컨테이너 생성 및 시작
docker compose up -d

# 이미지 빌드 후 실행
docker compose up -d --build

# 특정 파일 지정
docker compose -f 파일명.yaml up -d

# 컨테이너 + 네트워크 삭제
docker compose down

# 볼륨까지 삭제
docker compose down -v

# 컨테이너 목록
docker compose ps

# Compose 프로젝트 목록
docker compose ls

# 로그 확인
docker compose logs
docker compose logs 서비스명

# 설정 유효성 확인
docker compose config

# 특정 서비스만 재시작
docker compose restart 서비스명
```

---

>  **핵심 요약**
> - stop(SIGTERM) vs kill(SIGKILL) / rm(컨테이너) vs rmi(이미지) / run(신규) vs start(재시작)
> - `-p 직접지정` vs `-P EXPOSE자동매핑` / COPY(단순) vs ADD(URL+압축)
> - `docker compose down`(컨테이너+네트워크 삭제) vs `stop`(중지만)
> - Bind Mount(호스트 경로 직접) vs Named Volume(Docker 관리, 이식성 높음)
> - 관련: 1.  Docker - 도커와 컨테이너의 이해 · 8.  Docker - Docker Compose · 9.  Docker - 통합 정리
