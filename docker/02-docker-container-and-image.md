# 📦 Docker - 컨테이너와 이미지

> **Tag:** #Docker #이미지 #Dockerfile #컨테이너 #부트캠프
> **핵심 요약:** Dockerfile 작성법과 이미지 생성, 컨테이너 실행 방법 핵심 정리

---

## 1. 📖 개요 (Overview)

Dockerfile은 Docker 이미지를 자동으로 빌드하기 위한 명령어 모음 파일이다. 전체 흐름은 다음과 같다.

```
Dockerfile → docker build → Docker Image → docker run → Container
```

Dockerfile의 주요 명령어를 살펴보면, `FROM`은 베이스 이미지를 지정하며 필수 항목으로 항상 최상단에 위치한다(`FROM ubuntu:22.04`). `LABEL`은 이미지 메타데이터를 추가하고(`LABEL version="1.0"`), `COPY`는 호스트 파일을 컨테이너로 복사하며(`COPY app.py /app/`), `ADD`는 COPY 기능에 더해 URL 다운로드와 압축 해제까지 지원한다(`ADD archive.tar.gz /app/`). `RUN`은 이미지 빌드 시 명령을 실행하며 새 레이어를 생성하고(`RUN apt-get install -y nginx`), `WORKDIR`은 작업 디렉터리를 설정하며(`WORKDIR /app`), `ENV`는 환경 변수를 설정한다(`ENV NODE_ENV=production`). `VOLUME`은 마운트 포인트를 선언하고(`VOLUME /data`), `USER`는 실행 사용자를 변경하며(`USER www-data`), `EXPOSE`는 컨테이너 포트를 문서화한다(단, 실제로 포트를 개방하는 것은 아니다, `EXPOSE 80`). `CMD`는 컨테이너 시작 시 기본 명령으로 덮어쓰기가 가능하고(`CMD ["nginx", "-g", "daemon off;"]`), `ENTRYPOINT`는 컨테이너 시작 시 고정 명령을 지정하며(`ENTRYPOINT ["python", "app.py"]`), `HEALTHCHECK`는 컨테이너 상태를 확인한다(`HEALTHCHECK CMD curl -f http://localhost/`).

CMD와 ENTRYPOINT는 둘 다 컨테이너 시작 시 실행되는 명령을 지정하지만 성격이 다르다. CMD는 기본 실행 명령으로 `docker run` 시 다른 명령으로 덮어쓸 수 있어 기본값 설정에 주로 쓰이는 반면, ENTRYPOINT는 고정 실행 명령으로 `docker run` 시 덮어쓸 수 없고 인자 추가만 가능하여 실행 파일을 고정할 때 사용한다.

```dockerfile
# CMD 예시 - docker run 시 다른 명령으로 교체 가능
CMD ["nginx", "-g", "daemon off;"]

# ENTRYPOINT 예시 - 항상 python으로 실행
ENTRYPOINT ["python"]
CMD ["app.py"]     # ENTRYPOINT의 기본 인자
```

COPY와 ADD도 자주 비교되는데, 둘 다 파일 복사는 지원하지만 ADD만 URL 다운로드와 tar 자동 압축 해제를 지원한다. 일반적인 파일 복사에는 예측 가능성이 높은 COPY를 권장하고, tar 압축 해제가 필요한 경우에만 ADD를 사용한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** Docker Engine (Compose V2 내장) 설치 환경.

### 실전 Dockerfile 예시 (Node.js)

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

### 실전 Dockerfile 예시 (Rocky Linux + httpd)

```dockerfile
FROM rockylinux:9.3

RUN dnf install -y httpd && dnf clean all

COPY index.html /var/www/html/index.html

EXPOSE 80

CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### Docker 이미지 빌드 및 관리 명령어

```bash
# 이미지 빌드 (현재 디렉터리의 Dockerfile 사용)
docker build -t 이미지명:태그 .

# 이미지 목록 확인
docker images

# 이미지 삭제
docker rmi 이미지명:태그

# 사용하지 않는 이미지 일괄 삭제
docker image prune -a -f

# 이미지 상세 정보
docker inspect 이미지명
```

### 컨테이너 실행 주요 옵션

```bash
docker run [옵션] 이미지명 [명령]
```

| 옵션 | 설명 | 예시 |
|---|---|---|
| `-d` | 백그라운드 실행 | `docker run -d nginx` |
| `--name` | 컨테이너 이름 지정 | `--name web1` |
| `-p 호스트:컨테이너` | 포트 포워딩 | `-p 8080:80` |
| `-v 호스트:컨테이너` | 볼륨 마운트 | `-v /data:/app/data` |
| `-e KEY=VALUE` | 환경 변수 전달 | `-e MYSQL_ROOT_PASSWORD=1234` |
| `--network` | 네트워크 지정 | `--network webnet` |
| `--rm` | 종료 시 자동 삭제 | `--rm` |
| `-it` | 대화형 터미널 | `-it /bin/bash` |

### Registry에서 이미지 push/pull

```bash
# Docker Hub 로그인
docker login

# 이미지 태그 변경
docker tag myapp:1.0 myusername/myapp:1.0

# Docker Hub에 push
docker push myusername/myapp:1.0

# Private Registry 실행
docker run -d --name registry -p 5000:5000 registry:3

# Private Registry에 push
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
```

---

> 📌 **핵심 요약**
> - Dockerfile: 이미지 자동 빌드 명령 파일 (FROM → RUN → COPY → CMD 순서)
> - CMD(덮어쓰기 가능) vs ENTRYPOINT(고정) — 실행 명령 제어 방식 차이
> - COPY(단순 복사) vs ADD(URL/압축 해제 지원) — 일반 파일은 COPY 권장
> - 레이어 생성: FROM, RUN, COPY, ADD → 이미지 크기에 직접 영향
> - 관련: 1. 🐳 Docker - 도커와 컨테이너의 이해 · 3. 🔄 Docker - 컨테이너 사용하기 · 11. ⚡ Docker - 퀵 레퍼런스
