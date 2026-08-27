# 🐧 Linux Samba 서버 구축 (smbd·nmbd·smb.conf)

> **Tag:** #Linux #Samba #smbd #nmbd #smbconf #smbpasswd #firewalld #SELinux  
> **핵심 요약:** Linux Samba 서버는 파일 공유를 담당하는 `smbd`(TCP 445·139)와 NetBIOS 이름 해석을 담당하는 `nmbd`(UDP 137·138) 두 데몬으로 동작한다. 공유 디렉터리를 만들고 Linux 계정을 `smbpasswd -a`로 Samba 사용자 DB에 등록한 뒤, `/etc/samba/smb.conf`에 공유 섹션을 정의하고 방화벽에서 samba 서비스를 허용하면 Windows 탐색기에서 네트워크 드라이브로 연결할 수 있다. RHEL 계열에서는 SELinux 컨텍스트(`samba_share_t`) 설정이 필수적인 경우가 많다.

---

## 1. 📖 개요 (Overview)

Samba는 크게 두 가지 데몬으로 동작한다.

**1) SMB 데몬(smbd)** 은 SMB 프로토콜을 사용해 파일·폴더 공유 기능을 수행하는 핵심 데몬이다. Windows 탐색기에서 공유 폴더를 열 때 실제로 응답하는 부분이며 인증(로그인), 파일 읽기/쓰기, 파일 잠금(lock)을 처리한다.

```text
smbd 사용 포트
TCP 445   Direct SMB (최신 Windows 기본)
TCP 139   NetBIOS 기반 SMB (구 버전 호환)
```

**2) NMB 데몬(nmbd)** 은 Windows 네트워크 환경에서 컴퓨터 이름을 검색할 수 있게 하는 데몬이다. 예를 들어 네트워크에서 "SERVER-A"를 자동 탐색하게 해주며 NetBIOS 기반 이름 해석 기능을 제공한다.

```text
nmbd 사용 포트
UDP 137   NetBIOS Name Service
UDP 138   NetBIOS Datagram Service
```

> **참고:** IP로 직접 접속(`\\192.168.10.100\SHARE`)만 사용한다면 `nmbd` 없이 `smbd`만으로도 동작한다. 이름 탐색이 필요할 때 `nmbd`를 함께 켠다.

Samba 접속용 계정은 반드시 Linux에 먼저 존재해야 하며, 그 계정을 `smbpasswd -a`로 Samba 사용자 데이터베이스에 추가하고 별도의 SMB 비밀번호를 설정한다. 즉 Linux 로그인 비밀번호와 Samba 접속 비밀번호는 서로 다른 값일 수 있다.

```text
useradd samba      → Linux 계정 생성(인증용)
passwd samba       → Linux 로그인 비밀번호
smbpasswd -a samba → Samba 접속 비밀번호(별도 DB)
```

서버 접속만 담당하고 실제 셸 로그인이 필요 없다면 `useradd -s /sbin/nologin samba`처럼 로그인 셸을 제한하는 것이 더 안전하다.

smb.conf의 주요 옵션은 다음과 같다.

| 옵션 | 의미 |
|---|---|
| `path` | 공유 디렉터리의 실제 경로 |
| `writable = yes` | 쓰기 권한 부여 |
| `browseable = yes` | 탐색기 공유 목록에 표시 |
| `guest ok = no` | 익명(게스트) 접근 차단 |
| `valid users = @SG` | SG 그룹 소속 사용자만 접속 허용 |
| `force group = SG` | 생성되는 파일·디렉터리를 SG 그룹 소유로 강제 |
| `create mask = 0666` | 새 파일 생성 시 권한 |
| `directory mask = 0777` | 새 디렉터리 생성 시 권한 |

`valid users`에서 `@`는 그룹을 의미한다. `force group`을 지정하면 공유 디렉터리에서 만들어지는 모든 파일이 해당 그룹 소유가 되어 그룹 단위 협업이 쉬워진다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 패키지 설치

```bash
dnf install -y samba samba-client samba-common       # Samba 서버·클라이언트
rpm -qa | grep samba                                 # 설치 확인
```

---

### Step 2. 공유 디렉터리·계정·그룹 준비

```bash
mkdir /SHARE                               # 공유 디렉터리 생성

useradd samba                              # Samba 인증용 Linux 계정
groupadd SG                                # 공유 디렉터리 연동 그룹
usermod -aG SG samba                       # 계정을 그룹에 추가

chmod 777 /SHARE                           # 실습용 권한
ls -l / | grep SHARE                       # 권한 확인
```

> **참고:** `usermod -G`는 기존 보조 그룹을 덮어쓴다. 기존 그룹을 유지하려면 `-aG`를 사용한다.

---

### Step 3. Samba 비밀번호 등록

```bash
passwd samba                               # Linux 로그인 비밀번호
smbpasswd -a samba                         # Samba 접속 비밀번호 등록
pdbedit -L                                 # Samba 사용자 DB 목록 확인
```

`-a`는 Linux에 존재하는 사용자를 Samba 사용자 DB에 추가하고 Samba 접속용 비밀번호를 설정한다는 의미이다.

기타 관리 옵션:

```bash
smbpasswd -d samba                         # 계정 비활성화
smbpasswd -e samba                         # 계정 활성화
smbpasswd -x samba                         # Samba DB에서 삭제
```

---

### Step 4. 방화벽 설정

```bash
firewall-cmd --permanent --add-service=samba          # samba 서비스 일괄 허용
firewall-cmd --permanent --add-port=445/tcp
firewall-cmd --permanent --add-port=139/tcp
firewall-cmd --permanent --add-port=137/udp
firewall-cmd --permanent --add-port=138/udp
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-port                   # 허용 포트 확인
firewall-cmd --list-services               # 허용 서비스 확인
firewall-cmd --list-all                    # 전체 확인
```

> **참고:** `--add-service=samba`만으로도 137·138·139·445가 함께 허용된다. 개별 포트 추가는 중복이지만 실습 확인용으로는 문제 없다. `--permanent` 없이 추가한 규칙은 `--reload`나 재부팅 시 사라진다.

---

### Step 5. smb.conf 공유 섹션 설정

```bash
cp -a /etc/samba/smb.conf /etc/samba/smb.conf.bak     # 원본 백업
vi /etc/samba/smb.conf
```

파일 하단에 공유 섹션을 추가한다.

```text
[Share]
        path = /SHARE
        writable = yes
        browseable = yes
        guest ok = no
        valid users = @SG
        force group = SG
        create mask = 0666
        directory mask = 0777
```

문법 검증:

```bash
testparm                                   # smb.conf 문법·해석 결과 확인
testparm -s                                # 요약 출력
```

---

### Step 6. SELinux 설정 (RHEL·Rocky 필수 확인)

```bash
getenforce                                 # SELinux 상태 확인
semanage fcontext -a -t samba_share_t "/SHARE(/.*)?"   # 컨텍스트 규칙 추가
restorecon -Rv /SHARE                                  # 컨텍스트 적용
ls -Zd /SHARE                                          # samba_share_t 확인
```

`semanage`가 없으면 `dnf install -y policycoreutils-python-utils`로 설치한다.

> **참고:** 권한과 설정이 모두 맞는데도 접근이 거부된다면 대부분 SELinux 컨텍스트 문제다. `ausearch -m avc -ts recent`로 거부 로그를 확인한다.

---

### Step 7. 데몬 시작·자동 실행

```bash
systemctl start smb                        # SMB 데몬 시작
systemctl enable smb                       # 부팅 시 자동 실행
systemctl status smb                       # 상태 확인

systemctl start nmb                        # NMB 데몬 시작
systemctl enable nmb
systemctl status nmb
```

정상 상태에서는 `Active: active (running)`과 `Status: "smbd: ready to serve connections..."`가 표시된다.

설정 변경 후에는 재시작한다.

```bash
systemctl restart smb nmb                  # 설정 반영
```

---

### Step 8. Windows에서 접속

```text
파일 탐색기 → 내 PC → 컴퓨터 → 네트워크 드라이브 연결
드라이브: Z:
폴더:     \\192.168.10.100\SHARE
[다른 자격 증명을 사용하여 연결] 체크
마침 → 계정 samba / 비밀번호 입력
```

명령으로 연결·해제하는 방법:

```text
net use Z: \\192.168.10.100\SHARE /user:samba 1234
net use Z: /delete
net use                                    # 현재 연결 목록
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 서버 측 자체 점검

```bash
testparm                                   # 설정 문법 확인
smbclient -L localhost -U samba            # 로컬에서 공유 목록 조회
smbstatus                                  # 접속 세션·잠금 상태 확인
ss -tulnp | egrep '445|139|137|138'        # 리스닝 포트 확인
```

### 3-2. 대표 오류

| 증상 | 원인·조치 |
|---|---|
| `NT_STATUS_LOGON_FAILURE` | Samba 비밀번호 미등록 → `smbpasswd -a` |
| `NT_STATUS_ACCESS_DENIED` | `valid users` 불일치, 디렉터리 퍼미션 부족 |
| 공유는 보이는데 쓰기 불가 | `writable = yes`, 디렉터리 쓰기 권한, SELinux 확인 |
| 탐색기에서 공유가 안 보임 | `browseable = yes`, `nmbd` 미실행, 방화벽 |
| Windows가 이전 계정으로 접속 | 자격 증명 캐시 → `net use * /delete` 후 재접속 |
| 설정 변경이 반영 안 됨 | `systemctl restart smb nmb` |

### 3-3. 로그 확인

```bash
journalctl -u smb -n 50                    # smbd 로그
journalctl -u nmb -n 50                    # nmbd 로그
ls -l /var/log/samba/                      # Samba 로그 디렉터리
```

> 📌 **핵심 요약**
> - `smbd`는 파일 공유(445·139), `nmbd`는 이름 해석(137·138)
> - Linux 계정 생성 → `smbpasswd -a`로 Samba DB 등록
> - `smb.conf`에 공유 섹션 정의 후 `testparm`으로 검증
> - 방화벽은 `--permanent --add-service=samba` + `--reload`
> - RHEL 계열은 SELinux `samba_share_t` 설정 필수
> - 관련: 🪟 Windows SMB 서버 & Linux CIFS 클라이언트 · 🏗️ 종합실습 Samba 공유 & 권한 제어 · 🚨 Samba·NFS 트러블슈팅 치트시트
