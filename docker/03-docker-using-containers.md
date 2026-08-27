# 🔄 Docker - 컨테이너 사용하기

> **Tag:** #Docker #컨테이너 #라이프사이클 #명령어 #부트캠프
> **핵심 요약:** 컨테이너 생명주기와 주요 운영 명령어 핵심 정리

---

## 1. 📖 개요 (Overview)

컨테이너는 생명주기(Lifecycle) 동안 여러 단계를 거친다. 전체 흐름은 다음과 같다.

```
pull → create → start → (pause/unpause) → exec → stop/kill → rm
```

각 단계는 다음과 같다. pull(`docker pull 이미지`)은 레지스트리에서 이미지를 다운로드하고, create(`docker create 이미지`)는 컨테이너를 생성하되 아직 실행하지는 않으며, start(`docker start 컨테이너`)는 컨테이너를 시작한다. run(`docker run 이미지`)은 pull + create + start를 한 번에 처리하는 명령이다. pause(`docker pause 컨테이너`)는 컨테이너를 일시 정지시키고 unpause(`docker unpause 컨테이너`)는 그 정지를 해제하며, exec(`docker exec 컨테이너 명령`)는 실행 중인 컨테이너에 명령을 전달한다. stop(`docker stop 컨테이너`)은 SIGTERM으로 컨테이너를 정상 종료시키는 반면 kill(`docker kill 컨테이너`)은 SIGKILL로 강제 종료시키며, 마지막으로 rm(`docker rm 컨테이너`)은 컨테이너를 삭제한다.

컨테이너 상태를 확인할 때는 다음 명령어들을 사용한다.

```bash
# 실행 중인 컨테이너 목록
docker ps

# 모든 컨테이너 목록 (종료된 것 포함)
docker ps -a

# 컨테이너 상세 정보 (IP, 마운트 등)
docker inspect 컨테이너명

# 컨테이너 내 실행 중인 프로세스 확인
docker top 컨테이너명

# 컨테이너 로그 확인
docker logs 컨테이너명
docker logs -f 컨테이너명    # 실시간 로그 스트림
```

컨테이너 내부에 접속할 때는 `docker exec`를 사용한다.

```bash
# 컨테이너 내부 bash 쉘 접속
docker exec -it 컨테이너명 /bin/bash

# 컨테이너 내부에서 명령 실행 후 종료
docker exec 컨테이너명 ls /var/www/html

# 옵션 설명
# -i : 표준입력 유지 (Interactive)
# -t : 가상 터미널 할당 (TTY)
```

컨테이너를 삭제할 때는 몇 가지 주의사항이 있다. 실행 중인 컨테이너는 먼저 중지한 뒤 삭제해야 하며, 실행 중이어도 강제로 삭제하려면 `-f` 옵션을 사용한다. 중지된 컨테이너를 일괄 삭제하려면 `docker container prune`을 사용한다.

```bash
# 실행 중인 컨테이너는 먼저 중지 후 삭제
docker stop 컨테이너명
docker rm 컨테이너명

# 실행 중이어도 강제 삭제 (-f)
docker rm -f 컨테이너명

# 모든 중지된 컨테이너 일괄 삭제
docker container prune
```

> ⚠️ 컨테이너 삭제 시 컨테이너 내부 데이터(쓰기 레이어)는 함께 삭제됨
> 영구 데이터는 반드시 Volume 사용

패키지 설치는 컨테이너 내부에서 직접 하기보다 Dockerfile에 포함하는 것이 권장된다. 컨테이너 내부에서 직접 설치하면 즉시 적용은 되지만 컨테이너가 삭제될 때 함께 사라지는 반면, Dockerfile의 `RUN`으로 설치하면 이미지에 포함되어 영구적이고 재현 가능하다.

```dockerfile
# 올바른 방법: Dockerfile에 포함
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx
```

컨테이너를 일괄 정리할 때는 다음 명령어를 사용한다. `docker system prune`은 중지된 컨테이너, 사용하지 않는 이미지, 네트워크를 일괄 정리하고, `docker system prune -a --volumes`는 볼륨까지 포함하여 정리한다. 실행 중인 컨테이너 전체를 중지하려면 `docker stop $(docker ps -q)`를, 전체 컨테이너를 삭제하려면 `docker rm $(docker ps -aq)`를 사용한다.

```bash
# 중지된 컨테이너 + 사용 안 하는 이미지 + 네트워크 일괄 정리
docker system prune

# 볼륨까지 포함하여 정리
docker system prune -a --volumes

# 실행 중인 컨테이너 전체 중지
docker stop $(docker ps -q)

# 전체 컨테이너 삭제
docker rm $(docker ps -aq)
```

자주 쓰는 컨테이너 운영 패턴은 다음과 같다.

```bash
# Nginx 웹서버 실행
docker run -d --name web -p 80:80 nginx:latest

# MySQL DB 실행 (환경변수 + 볼륨)
docker run -d --name mysql_db \
  -e MYSQL_ROOT_PASSWORD=admin1234 \
  -v /dbdata:/var/lib/mysql \
  -p 3306:3306 \
  mysql:latest

# 컨테이너 내부 bash 접속
docker exec -it mysql_db /bin/bash

# 컨테이너 로그 실시간 확인
docker logs -f web
```

---

> 📌 **핵심 요약**
> - 생명주기: pull → create → start → (pause/unpause) → exec → stop/kill → rm
> - `docker run` = pull + create + start 를 한번에 처리
> - stop(SIGTERM, 정상 종료) vs kill(SIGKILL, 강제 종료) 구분
> - 컨테이너 삭제 시 내부 데이터 함께 삭제 → 영구 데이터는 반드시 Volume 사용
> - 패키지 설치는 Dockerfile RUN에 포함해야 이미지에 영구 반영됨
> - 관련: 1. 🐳 Docker - 도커와 컨테이너의 이해 · 5. 💾 Docker - 스토리지 · 11. ⚡ Docker - 퀵 레퍼런스
