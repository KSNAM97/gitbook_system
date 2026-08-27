# 💻 NFS 클라이언트 마운트 & fstab 영구화

> **Tag:** #Linux #NFS #mount #fstab #autofs #nfsutils #_netdev  
> **핵심 요약:** NFS 클라이언트는 `nfs-utils`를 설치하고 마운트포인트를 만든 뒤 `mount -t nfs 서버IP:/공유디렉터리 /마운트포인트`로 연결한다. 이 마운트는 재부팅 시 해제되므로 `/etc/fstab`에 등록해 자동 마운트를 구성해야 한다. NFS는 네트워크 자원이므로 `_netdev` 옵션을 함께 쓰는 것이 안전하며, 서버 장애 시 부팅이 멈추지 않도록 `nofail` 사용을 검토한다.

---

## 1. 📖 개요 (Overview)

클라이언트 마운트 흐름은 다음과 같다.

```text
1) nfs-utils 설치
2) 마운트포인트 디렉터리 생성 (mkdir)
3) mount -t nfs 서버IP:/공유  /마운트포인트
4) df -h, mount 로 확인
5) /etc/fstab 등록 → 재부팅 후에도 유지
```

마운트포인트가 없으면 다음 오류가 난다.

```text
mount.nfs: mount point /NFSC does not exist
```

일반 사용자는 마운트 권한이 없으므로 `sudo`가 필요하다.

```text
mkdir: `/NFSC' 디렉토리를 만들 수 없습니다: 허가 거부
```

마운트 옵션을 정리하면 다음과 같다.

| 옵션 | 의미 |
|---|---|
| `hard` | 서버 응답이 없으면 무한 재시도(기본, 데이터 안전) |
| `soft` | 일정 횟수 후 I/O 오류 반환(응답성 우선, 손실 위험) |
| `timeo=600` | 재전송 대기 시간(0.1초 단위, 600 = 60초) |
| `retrans=2` | 재전송 횟수 |
| `rsize`, `wsize` | 읽기·쓰기 블록 크기(성능 튜닝) |
| `_netdev` | 네트워크 준비 후 마운트 |
| `nofail` | 마운트 실패해도 부팅 계속 |
| `vers=4.2` | NFS 버전 명시 |
| `ro` / `rw` | 클라이언트 측 읽기 전용·읽기 쓰기 |

마운트 후 실제 적용 옵션은 다음처럼 확인한다.

```text
192.168.10.100:/NFSS on /NFSC type nfs4
(rw,relatime,vers=4.2,rsize=262144,wsize=262144,namlen=255,hard,
proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.10.130,
local_lock=none,addr=192.168.10.100)
```

`type nfs4`와 `vers=4.2`는 NFSv4로 협상되었음을 뜻한다.

NFS의 기본 인증 방식(`sec=sys`)은 클라이언트가 보낸 UID/GID를 그대로 신뢰한다. 따라서 서버와 클라이언트의 UID가 다르면 같은 사용자 이름이어도 다른 사용자로 취급된다.

```text
서버:      user1 = UID 1001
클라이언트: user1 = UID 1005
→ 서버 입장에서는 UID 1005 = 다른 사용자 → 권한 거부
```

해결 방법으로는 UID/GID를 서버·클라이언트에서 통일하거나, `all_squash` + `anonuid`/`anongid`로 모든 접근을 하나의 계정으로 매핑하거나, LDAP·SSSD 같은 중앙 인증을 사용하는 방법이 있다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 클라이언트 패키지 설치

```bash
dnf install -y nfs-utils                   # NFS 클라이언트 도구
dnf install -y net-tools                   # ifconfig 등 확인용(선택)
rpm -qa | grep nfs                         # 설치 확인
```

---

### 2-2. 호스트명·IP 확인

```bash
sudo hostnamectl set-hostname Client-L     # 호스트명 설정
exec bash                                  # 프롬프트 반영
hostnamectl                                # 확인

ifconfig | grep 192                        # IP 확인
ip -4 addr show | grep inet                # (권장) ip 명령
```

---

### 2-3. 서버 공유 목록 확인

```bash
showmount -e 192.168.10.100                # 서버가 제공하는 export 목록
ping -c 2 192.168.10.100                   # 네트워크 도달 확인
```

---

### 2-4. 임시 마운트

```bash
sudo mkdir /NFSC                           # 마운트포인트 생성
sudo mount -t nfs 192.168.10.100:/NFSS /NFSC
```

옵션을 지정하는 방식:

```bash
sudo mount -t nfs -o rw,hard,timeo=600,vers=4.2 \
  192.168.10.100:/NFSS /NFSC
```

확인:

```bash
df -h                                      # 마운트된 용량 확인
mount | grep NFS                           # 마운트 옵션 확인
findmnt /NFSC                              # 계층적 확인
```

`df -h` 출력에 `192.168.10.100:/NFSS ... /NFSC`가 보이면 정상이다.

---

### 2-5. 읽기·쓰기 테스트

```bash
sudo cp -r /etc/a* /NFSC/                  # 테스트 복사
ls -l /NFSC/                               # 결과 확인
sudo touch /NFSC/testfile                  # 쓰기 테스트
```

`Permission denied`가 나면 서버 측 `rw` 옵션, 디렉터리 퍼미션, UID 매핑, SELinux를 순서대로 확인한다.

---

### 2-6. 마운트 해제

```bash
sudo umount /NFSC                          # 일반 해제
sudo umount -l /NFSC                       # lazy 해제(사용 중일 때)
sudo fuser -mv /NFSC                       # 사용 중인 프로세스 확인
```

---

### 2-7. fstab 자동 마운트

재부팅하면 마운트가 해제되므로 fstab에 등록한다.

```bash
sudo cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"
sudo vi /etc/fstab
```

```text
192.168.10.100:/NFSS   /NFSC        nfs  defaults,_netdev        0 0
192.168.10.100:/NFS_LC /NFS_CLIENT  nfs  defaults,_netdev,nofail 0 0
```

적용·검증:

```bash
findmnt --verify --verbose                 # fstab 문법 검증
sudo systemctl daemon-reload               # systemd 반영
sudo umount /NFSC
sudo mount -a                              # fstab 기준 마운트
findmnt /NFSC                              # 최종 확인
```

> NFS 서버가 꺼진 상태로 클라이언트를 재부팅하면 `_netdev`만으로는 부팅이 지연될 수 있다. 운영 환경에서는 `nofail`을 함께 지정하거나 autofs를 사용하는 것이 안전하다.

---

### 2-8. (선택) autofs 자동 마운트

접근할 때만 마운트하고 유휴 시 자동 해제하려면 autofs를 사용한다.

```bash
sudo dnf install -y autofs
echo "/-  /etc/auto.nfs" | sudo tee -a /etc/auto.master
echo "/NFSC  -fstype=nfs,rw  192.168.10.100:/NFSS" | sudo tee /etc/auto.nfs
sudo systemctl enable --now autofs
ls /NFSC                                   # 접근 시 자동 마운트
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 점검 명령

```bash
showmount -e 192.168.10.100                # 서버 export 확인
mount | grep nfs                           # 협상 버전·옵션
nfsstat -m                                 # 마운트별 상세 통계
dmesg | tail -20                           # 커널 NFS 오류
```

### 3-2. 대표 오류

| 증상 | 원인·조치 |
|---|---|
| `mount point /NFSC does not exist` | 마운트포인트 미생성 → `mkdir` |
| `access denied by server while mounting` | 서버 exports의 클라이언트 IP 불일치 |
| `Connection timed out` | 방화벽(2049·111) 차단, 서버 서비스 미실행 |
| `clnt_create: RPC: Program not registered` | `rpcbind`·`nfs-server` 미실행, `showmount` 포트 차단 |
| 마운트는 되는데 쓰기 불가 | exports `ro`, 디렉터리 퍼미션, UID 불일치 |
| 재부팅 후 마운트 사라짐 | fstab 미등록 |
| 부팅이 멈춤 | fstab에 `_netdev,nofail` 추가 |
| 파일 소유자가 `nobody` | `root_squash`/`all_squash` 또는 idmap 설정 |

### 3-3. 문제 해결 순서

```text
1) ping 서버IP                    # 네트워크
2) showmount -e 서버IP            # export 공개 여부
3) 서버: exportfs -v              # 허용 IP 확인
4) 서버: firewall-cmd --list-all  # 포트 확인
5) 클라이언트: mount -v -t nfs ... # 상세 로그로 재시도
6) dmesg / journalctl 확인
```

> 📌 **핵심 요약**
> - 클라이언트는 `nfs-utils` 설치 후 마운트포인트를 먼저 생성
> - `mount -t nfs 서버IP:/공유 /마운트포인트`로 연결
> - 재부팅 유지에는 fstab + `_netdev`(운영은 `nofail` 권장)
> - `sec=sys`는 UID/GID를 신뢰하므로 계정 UID 통일이 중요
> - 접근 거부 시 exports IP → 퍼미션 → UID → SELinux 순 점검
> - 관련: ⚙️ NFS 서버 구성 · 🏗️ 종합실습 다중 클라이언트 NFS 구성 · 🚨 Samba·NFS 트러블슈팅 치트시트
