# 🚑 Docker - 트러블슈팅 치트시트

> **Tag:** #Docker #트러블슈팅 #에러 #치트시트 #부트캠프
> **핵심 요약:** 자주 발생하는 Docker 오류 시나리오와 해결 방법 핵심 정리

---

## 1. 🎯 핵심 기술 개념 (Concept)

### 트러블슈팅 핵심 원칙

| 상황 | 우선 확인 명령 |
|---|---|
| 컨테이너 상태 이상 | `docker ps -a` / `docker logs 컨테이너명` |
| 포트 충돌 | `ss -tlnp \| grep :포트` |
| 네트워크 통신 불가 | `docker network inspect 네트워크명` |
| 볼륨 마운트 문제 | `docker inspect 컨테이너명 \| grep -A 10 "Mounts"` |

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 시나리오 1. 컨테이너가 바로 종료됨

**증상:** `docker run` 직후 컨테이너가 바로 Exited 상태가 됨

**원인:** 메인 프로세스가 종료되면 컨테이너도 종료됨

```bash
# 확인
docker ps -a    # STATUS: Exited (0) 또는 Exited (1)
docker logs 컨테이너명

# 해결: 포그라운드로 실행되는 프로세스 필요
# 잘못된 예 (bash는 터미널 없으면 바로 종료)
docker run ubuntu /bin/bash

# 올바른 예 (대기 상태 유지)
docker run -it ubuntu /bin/bash           # 대화형
docker run -d ubuntu sleep infinity       # 백그라운드 무한 대기

# CMD/ENTRYPOINT에서 -D FOREGROUND 사용
CMD ["nginx", "-g", "daemon off;"]        # Nginx 포그라운드 실행
```

### 시나리오 2. 포트 충돌 오류

**증상:** `Bind for 0.0.0.0:80 failed: port is already allocated`

**원인:** 호스트의 해당 포트를 이미 다른 프로세스나 컨테이너가 사용 중

```bash
# 포트 사용 중인 프로세스 확인
ss -tlnp | grep :80
# 또는
netstat -tlnp | grep :80

# 기존 컨테이너 포트 확인
docker ps

# 해결 1: 다른 호스트 포트 사용
docker run -d -p 8080:80 nginx:latest

# 해결 2: 충돌 컨테이너 중지
docker stop 충돌컨테이너명

# 해결 3: 실행 중인 서비스 중지 (MariaDB 등)
sudo systemctl stop mariadb.service
```

### 시나리오 3. 이미지를 찾을 수 없음

**증상:** `Unable to find image 'xxx:latest' locally`, `Error response from daemon: pull access denied`

**원인:** 이미지 이름/태그 오타 또는 로그인 필요

```bash
# 이미지 검색
docker search nginx

# 정확한 이름/태그 확인
docker pull nginx:latest
docker pull nginx:1.25

# Docker Hub 로그인 필요 시
docker login

# Private Registry 사용 시
docker login 레지스트리주소:포트

# 이미지 목록 확인
docker images
```

### 시나리오 4. 볼륨 마운트 후 데이터가 보이지 않음

**증상:** `-v /hostpath:/containerpath` 후 컨테이너에서 파일 접근 안 됨

**원인:** 호스트 경로 오타, 권한 문제, 디렉터리 미생성

```bash
# 경로 확인
ls -la /hostpath

# 권한 확인 및 수정
sudo chmod 755 /hostpath
sudo chown -R guest:guest /hostpath

# 디렉터리 생성
sudo mkdir -p /hostpath

# 컨테이너 내부 마운트 확인
docker inspect 컨테이너명 | grep -A 10 "Mounts"

# 또는
docker exec -it 컨테이너명 ls /containerpath
```

### 시나리오 5. 컨테이너 내부에서 명령어를 찾을 수 없음

**증상:** `bash: ip: command not found`, `bash: ping: command not found`

**원인:** 경량 이미지(alpine 등)는 기본 툴 미포함

```bash
# Debian/Ubuntu 계열 컨테이너
apt-get update
apt-get install -y iproute2          # ip 명령
apt-get install -y iputils-ping      # ping 명령
apt-get install -y net-tools         # netstat 명령
apt-get install -y bridge-utils      # brctl 명령
apt-get install -y vim               # 편집기

# Rocky/CentOS 계열 컨테이너
dnf install -y iproute
dnf install -y bridge-utils
```

### 시나리오 6. Docker Compose 파일을 찾지 못함

**증상:** `no configuration file provided: not found`

**원인:** 기본 파일명이 아닌 파일을 사용하거나 잘못된 디렉터리에서 실행

```bash
# 기본 파일명 (자동 인식)
# 1) compose.yaml
# 2) compose.yml
# 3) docker-compose.yaml
# 4) docker-compose.yml

# 기본 파일명이 아닐 때 -f 옵션 사용
docker compose -f docker-compose-httpd.yaml up -d
docker compose -f docker-compose-httpd.yaml ps

# 설정 유효성 확인
docker compose config
docker compose -f 파일명.yaml config
```

### 시나리오 7. MySQL 컨테이너에 접속이 안 됨

**증상:** `ERROR 1045 (28000): Access denied for user 'root'`

**원인:** 패스워드 오타 또는 환경 변수 설정 오류

```bash
# 환경 변수 확인
docker inspect 컨테이너명 | grep -i mysql

# 올바른 접속 방법 (비밀번호 입력 프롬프트 사용)
docker exec -it mysql_db /bin/bash
mysql -u root -p
# Enter password: (비밀번호 직접 입력)

# 환경 변수 재설정 (컨테이너 재생성 필요)
docker rm -f mysql_db
docker run -d --name mysql_db \
  -e MYSQL_ROOT_PASSWORD=correct_password \
  mysql:latest
```

### 시나리오 8. 컨테이너 네트워크 통신 안 됨

**증상:** 같은 네트워크의 컨테이너끼리 통신 불가

**원인:** 기본 bridge 사용 시 이름으로 통신 불가 / 서로 다른 네트워크

```bash
# 기본 bridge는 이름으로 통신 불가
# → User-Defined Bridge 사용

# 네트워크 생성
docker network create webnet

# 같은 네트워크로 컨테이너 실행
docker run -d --name web1 --network webnet nginx:latest
docker run -d --name db --network webnet mysql:latest

# 컨테이너 내부에서 이름으로 통신 가능
docker exec -it web1 ping db

# 네트워크 확인
docker network inspect webnet
docker inspect web1 | grep -i network
```

### 시나리오 9. Read-only 파일시스템 에러

**증상:** `bash: /path/file: Read-only file system`

**원인:** `:ro` 옵션으로 읽기 전용 마운트된 경로에 쓰기 시도

```bash
# 현재 마운트 옵션 확인
docker inspect 컨테이너명 | grep -A 5 '"RW"'

# 해결: 쓰기가 필요하면 :ro 제거 또는 별도 writer 컨테이너 사용
# writer: 읽기/쓰기
docker run -d --name writer \
  -v /webdata:/webdata \
  ubuntu:latest sleep infinity

# web: 읽기 전용
docker run -d --name web \
  -v /webdata:/usr/share/nginx/html:ro \
  nginx:latest
```

### 시나리오 10. 한글이 깨짐 (컨테이너 내부)

**증상:** vi 편집기에서 한글 입력/표시 깨짐

**원인:** 컨테이너 기본 로케일이 영어 설정

```bash
# 환경 변수로 해결
export LANG=C.UTF-8
export LC_ALL=C.UTF-8

# Dockerfile에 영구 적용
FROM ubuntu:22.04
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8

# 또는 Docker Compose
environment:
  LANG: C.UTF-8
  LC_ALL: C.UTF-8
```

---

> 📌 **핵심 요약**
> - 컨테이너 즉시 종료: 메인 프로세스가 없음 → `-d sleep infinity` 또는 포그라운드 실행
> - 포트 충돌: `ss -tlnp | grep :포트` 로 점유 프로세스 확인 후 다른 포트 사용
> - 네트워크 통신 불가: 기본 bridge는 이름 통신 불가 → User-Defined Bridge 사용
> - Read-only 에러: `:ro` 마운트 경로에 쓰기 시도 → writer 컨테이너 분리 패턴 적용
> - 한글 깨짐: `ENV LANG=C.UTF-8` / `ENV LC_ALL=C.UTF-8` Dockerfile에 추가
> - 관련: 3. 🔄 Docker - 컨테이너 사용하기 · 6. 🌐 Docker - 컨테이너 네트워크 · 8. 🔧 Docker - Docker Compose
