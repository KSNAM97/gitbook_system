# Docker - 스토리지

> **Tag:** #Docker #스토리지 #볼륨 #바인드마운트 #부트캠프
> **핵심 요약:** 컨테이너 데이터 영구 보존을 위한 스토리지 구조와 Volume 사용법 핵심 정리

---

## 1. 개요 (Overview)

컨테이너 저장 구조는 세 가지 요소로 이루어진다. Image Layer(읽기 전용)는 Dockerfile로 만든 읽기 전용 레이어들이고, Container Write Layer는 컨테이너 실행 중 변경 사항을 저장하는 휘발성 레이어이며, Storage Driver는 여러 레이어를 하나의 파일시스템처럼 관리한다.

이 중 Container Write Layer는 컨테이너 실행 시 이미지 위에 얇게 추가되는 쓰기 가능한 레이어이다. 파일 생성/수정, 로그 생성, 패키지 설치 등이 모두 이곳에 저장되며, 컨테이너 삭제 시 함께 사라지는 휘발성을 가지고, 다른 컨테이너와 공유되지 않는다.

대표적인 Storage Driver로는 현재 Linux 표준이며 성능이 좋고 안정적인 **overlay2**, 과거 Ubuntu에서 사용했으나 현재 거의 쓰이지 않는 aufs, 과거 CentOS/RHEL에서 사용하던 devicemapper, 스냅샷과 압축을 지원하지만 사용 환경이 적은 btrfs, 데이터 무결성이 강하지만 메모리를 많이 사용하는 zfs가 있다. overlay2의 동작 순서는 다음과 같다. 먼저 lowerdir에 읽기 전용 이미지 레이어를 준비하고, 컨테이너 실행 시 upperdir에 쓰기 레이어를 생성하며, merged에서 lowerdir와 upperdir를 하나의 파일시스템으로 합쳐 표시한다.

Container Write Layer와 Docker Volume을 비교하면, Container Write Layer는 컨테이너 삭제 시 데이터가 사라지고 속도는 느린 편이며 로그·임시 파일 용도로 쓰이는 반면, Docker Volume은 삭제하지 않는 한 데이터가 유지되고 속도도 빠른 편이며 DB·웹 데이터 등 영구 저장 용도로 쓰인다. 핵심은 컨테이너 레이어는 일회성이고 Volume은 영구 저장이라는 점이다.

Docker Volume 사용법은 다음과 같다.

#### 익명 볼륨 (Anonymous Volume)
```bash
# -v 컨테이너경로 만 지정 → Docker가 자동으로 볼륨 생성
docker run -d --name mysql_db \
  -v /var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=admin1234 \
  mysql:latest

# 생성된 볼륨 위치 확인
docker inspect mysql_db
# → "Source": "/var/lib/docker/volumes/랜덤ID/_data"
```

#### Bind Mount (호스트 디렉터리 직접 연결)
```bash
# -v 호스트경로:컨테이너경로
docker run -d --name mysql_db \
  -v /dbdata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=admin1234 \
  mysql:latest

# 호스트의 /dbdata 디렉터리에 DB 파일 저장됨
ls /dbdata/
```

#### Read-Only 마운트
```bash
# :ro 옵션으로 읽기 전용 마운트
docker run -d --name web \
  -v /webdata:/usr/share/nginx/html:ro \
  -p 8080:80 \
  nginx:latest

# 컨테이너 내부에서 쓰기 시도 → 에러
echo "test" >> /usr/share/nginx/html/index.html
# bash: /usr/share/nginx/html/index.html: Read-only file system
```

writer + web(Read-Only) 패턴은 데이터 수정 전용 컨테이너와 서비스 전용 컨테이너를 분리하는 패턴이다.

```
호스트 /webdata
   ├── writer 컨테이너 (읽기/쓰기) → 파일 생성/수정
   └── web 컨테이너 (읽기 전용) → 웹 서비스만

```

```bash
# writer 컨테이너: 파일 수정 가능
docker run -d --name writer \
  -v /webdata:/webdata \
  ubuntu:latest sleep infinity

# web 컨테이너: 읽기 전용 서비스
docker run -d --name web \
  -v /webdata:/usr/share/nginx/html:ro \
  -p 8080:80 \
  nginx:latest
```

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** Docker Engine (Compose V2 내장) 설치 환경.

### Nginx autoindex 설정 예시

```nginx
# /nginx_config/default.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /text/ {
        autoindex on;              # 파일 목록 출력
        autoindex_exact_size off;  # 파일 크기 읽기 쉽게 (KB/MB)
        autoindex_localtime on;    # 로컬 시간 표시
    }
}
```

```bash
# Nginx 설정 파일도 볼륨으로 주입
docker run -d --name web \
  -v /webdata:/usr/share/nginx/html:ro \
  -v /nginx_config/default.conf:/etc/nginx/conf.d/default.conf:ro \
  -p 8080:80 \
  nginx:latest
```

### 웹 서버 2대 + DB 1대 구조는?

```
Rocky Linux (web1, mydb) ← 호스트 /mysqldb:/var/lib/mysql
Ubuntu Linux (web2)       ← Rocky의 DB에 원격 접속
```

```bash
# DB 컨테이너 (Rocky)
docker run -d --name mydb --network webnet \
  -e MYSQL_ROOT_PASSWORD=1234 \
  -e MYSQL_DATABASE=labdb \
  -v /mysqldb:/var/lib/mysql \
  -p 3306:3306 mysql:latest

# web1 컨테이너 (Rocky, PHP+Apache)
docker run -d --name web1 --network webnet \
  -p 80:80 web_db_conn:1.0

# web2 컨테이너 (Ubuntu, PHP+Apache)
docker run -d --name web2 \
  -p 80:80 web_db_conn:1.0
```

---

>  **핵심 요약**
> - Container Write Layer: 휘발성 (컨테이너 삭제 시 함께 삭제)
> - Docker Volume: 영구 보존 (삭제 명령 전까지 유지), 속도도 빠름
> - Bind Mount: 호스트 경로 직접 연결 (`-v /host:/container`)
> - `:ro` 옵션: 읽기 전용 마운트 — writer + web(ro) 패턴으로 역할 분리
> - overlay2: 현재 Linux 표준 Storage Driver
> - 관련: 3.  Docker - 컨테이너 사용하기 · 6.  Docker - 컨테이너 네트워크 · 8.  Docker - Docker Compose
