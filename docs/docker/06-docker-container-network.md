# Docker - 컨테이너 네트워크

> **Tag:** #Docker #네트워크 #포트포워딩 #브릿지
> **핵심 요약:** 컨테이너 간 통신 구조와 포트 포워딩, 사용자 정의 네트워크 핵심 정리

---

## 1. 개요 (Overview)

Docker는 설치 시 `docker0` 가상 브릿지(virtual ethernet bridge)를 자동으로 생성한다. 기본 네트워크 구조는 다음과 같다.

```
컨테이너1 eth0 ── veth ──┐
컨테이너2 eth0 ── veth ──┤── docker0 (172.17.0.1) ── 호스트 ens160 ── 인터넷
컨테이너3 eth0 ── veth ──┘
```

이 구조를 이루는 요소는 IP 대역이 172.17.0.0/16인 기본 브릿지 docker0, 컨테이너와 docker0를 연결하는 가상 케이블 쌍인 veth pair, 컨테이너 내부의 네트워크 인터페이스인 컨테이너 eth0이다.

veth pair는 컨테이너 내부 eth0와 호스트 docker0를 연결하는 가상 LAN 케이블 쌍으로, 두 개가 한 쌍으로 만들어지며 한쪽으로 들어간 패킷은 반드시 다른 쪽으로 나온다. 컨테이너를 생성하면 Docker가 자동으로 veth pair를 생성한다.

```bash
# 호스트에서 veth 인터페이스 확인
ip addr
brctl show    # bridge-utils 설치 필요: dnf install bridge-utils
```

포트 포워딩(Port Forwarding)은 컨테이너 내부 포트를 호스트 포트에 연결하여 외부에서 접근 가능하게 하는 기능이다. Docker는 내부적으로 **iptables NAT 규칙**을 자동 생성하며, 외부 요청은 호스트 포트를 거쳐 NAT를 통해 컨테이너 내부 포트로 전달된다.

`-p` 옵션에는 몇 가지 종류가 있다. `-p 호스트포트:컨테이너포트`는 특정 포트를 직접 연결하고(`-p 8080:80`), `-p 컨테이너포트`만 지정하면 호스트 포트가 랜덤으로 자동 배정되며(`-p 80` → 32768:80), `-P`는 Dockerfile의 EXPOSE 포트를 자동으로 매핑한다.

```bash
# 직접 지정
docker run -d --name web1 -p 80:80 nginx:latest

# 랜덤 포트 배정
docker run -d --name web2 -p 80 nginx:latest
docker ps    # 32768->80 확인

# EXPOSE 자동 매핑
docker run -d --name web3 -P webp:1.0
docker ps    # 32769->80 확인
```

포트 바인딩을 하지 않으면 컨테이너 내부에서는 정상 동작하지만 외부(다른 PC, 브라우저)에서는 접근이 불가능하다. docker0는 컨테이너 내부 네트워크(172.17.x.x)이기 때문에 외부 PC가 이 네트워크에 직접 접근할 수 없으며, 따라서 `-p` 옵션으로 NAT 포트 바인딩을 해주는 것이 필수적이다.

사용자 정의 브릿지 네트워크(User-Defined Bridge)는 기본 docker0 외에 사용자가 직접 만드는 가상 네트워크이다. 기본 bridge(docker0)와 비교하면, 기본 bridge는 컨테이너 이름으로 통신이 불가능하고 IP 고정이 필요하며 서비스 분리가 어려운 반면, User-Defined Bridge는 내부 DNS를 제공하여 컨테이너 이름으로 통신이 가능하고 DNS로 자동 해결되어 IP 고정이 불필요하며 네트워크별로 서비스 분리가 가능하다.

네트워크 생성 및 사용법은 다음과 같다.

```bash
# 기본 생성
docker network create webnet

# 고급 설정 (서브넷, 게이트웨이 직접 지정)
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.254 \
  mynet

# 네트워크 목록 확인
docker network ls

# 네트워크 상세 정보
docker network inspect mynet

# 컨테이너를 특정 네트워크로 실행
docker run -d --name web1 --network webnet nginx:latest

# 고정 IP 지정
docker run -d --name appjs \
  --net mynet \
  --ip 192.168.100.100 \
  -p 8080:8080 \
  node-app

# 실행 중인 컨테이너에 네트워크 추가 연결
docker network connect webnet web1

# 컨테이너 네트워크 연결 해제
docker network disconnect webnet web1

# 네트워크 삭제 (연결된 컨테이너 없을 때)
docker network rm webnet
```

네트워크를 추가로 사용하는 이유는 서비스 분리(web 서버는 webnet, db 서버는 dbnet으로 구분), 보안 강화(필요한 서비스끼리만 통신 허용), DNS 기반 통신(컨테이너 IP가 바뀌어도 이름으로 통신 가능) 등이 있다.

```bash
# 같은 네트워크의 컨테이너는 이름으로 통신 가능
# (컨테이너 내부에서)
ping web1
ping db
curl http://db:3306
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 네트워크 구조 확인 실습 명령어

```bash
# 호스트 네트워크 인터페이스 확인
ip addr

# docker0 브릿지 확인
ip addr show docker0

# 브릿지 연결 상태 확인
brctl show

# 컨테이너 IP 확인
docker exec -it 컨테이너명 ip addr
```

---

>  **핵심 요약**
> - Docker 설치 시 docker0(172.17.0.0/16) 가상 브릿지 자동 생성
> - veth pair: 컨테이너 eth0 ↔ docker0 연결하는 가상 LAN 케이블 쌍
> - 포트 포워딩: `-p 호스트:컨테이너` → iptables NAT 규칙 자동 생성
> - 기본 bridge는 이름으로 통신 불가 → User-Defined Bridge는 내부 DNS 지원
> - 같은 User-Defined 네트워크 내 컨테이너는 컨테이너 이름으로 직접 통신 가능
> - 관련: 5.  Docker - 스토리지 · 8.  Docker - Docker Compose · 10.  Docker - 트러블슈팅 치트시트
