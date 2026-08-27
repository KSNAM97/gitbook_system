# 🪟 Windows SMB 서버 & Linux CIFS 클라이언트

> **Tag:** #Windows #SMB #CIFS #smbclient #mount #fstab #netuser  
> **핵심 요약:** Windows는 Samba 데몬(smbd)을 갖고 있지 않아 Linux처럼 Samba 서버를 직접 구성할 수는 없지만, Windows의 공유 폴더 자체가 SMB 프로토콜을 사용하므로 결과적으로 Windows가 SMB 서버 역할을 수행한다. Linux에서는 `smbclient -L`로 공유 목록을 확인하고 `mount -t cifs`로 마운트한다. 재부팅 후에도 유지하려면 `/etc/fstab`에 `cifs` 항목과 `_netdev` 옵션을 등록하며, 비밀번호는 credentials 파일로 분리하는 것이 안전하다.

---

## 1. 📖 개요 (Overview)

Windows에는 Samba 데몬이 없지만 공유 폴더 기능 자체가 SMB 서버 구현이다. 따라서 Windows에서 폴더를 공유하고 계정 권한을 부여하면, Linux는 그 공유 폴더에 CIFS(SMB) 클라이언트로 접속할 수 있다.

```text
Windows 공유 폴더(SMB Server)  ←→  Linux mount -t cifs (SMB Client)
```

Windows 측 준비 사항으로는, 먼저 `C:` 드라이브에 공유 폴더(예: `winShare`)를 만들고 읽기·쓰기 권한을 부여한다. 그다음 Linux에서 인증에 사용할 로컬 계정을 생성한다. 계정 생성은 반드시 관리자 권한 명령 프롬프트에서 수행해야 한다.

```text
C:\Users\aaa> net user root 1234 /add
시스템 오류 5이(가) 생겼습니다.
액세스가 거부되었습니다.          ← 일반 권한에서는 실패
```

관리자 권한으로 실행하면 정상 생성된다.

```text
C:\Windows\system32> net user root 1234 /add
명령을 잘 실행했습니다.

C:\Windows\system32> net user      # 계정 목록 확인
C:\Windows\system32> ipconfig      # SMB 서버 IP 확인 (예: 192.168.10.131)
```

Windows 방화벽에서 "파일 및 프린터 공유"가 허용되어 있어야 하고, 네트워크 프로필이 "개인/도메인"이어야 접속이 원활하다. 공용 네트워크로 잡혀 있으면 445 포트가 차단된다.

공유 목록에 보이는 `$` 공유는 다음과 같다.

| 공유 이름 | 설명 |
|---|---|
| `ADMIN$` | 원격 관리용 기본 관리 공유(`C:\Windows`) |
| `C$` | 드라이브 전체 기본 관리 공유 |
| `IPC$` | 프로세스 간 통신(원격 IPC), 인증·열거에 사용 |
| `Users` | 사용자 프로필 공유 |
| `winShare` | 관리자가 직접 만든 일반 공유 |

`$`로 끝나는 공유는 숨김(관리) 공유로 일반 사용자에게는 보이지 않으며 관리자 계정만 접근 가능하다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. Linux에 클라이언트 패키지 설치

```bash
dnf install -y samba-client samba-common samba-winbind cifs-utils
rpm -qa | grep samba                       # 설치 확인
```

`mount -t cifs`는 `cifs-utils`의 `mount.cifs`가 있어야 동작한다.

---

### 2-2. 공유 목록 확인 (smbclient)

```bash
smbclient -L 192.168.10.131                # 공유 목록 조회
smbclient -L 192.168.10.131 -U root        # 계정 명시 조회
```

출력 예:

```text
        Sharename    Type      Comment
        ---------    ----      -------
        ADMIN$       Disk      원격 관리
        C$           Disk      기본 공유
        IPC$         IPC       원격 IPC
        Users        Disk
        winShare     Disk
SMB1 disabled -- no workgroup available
```

대화형 접속도 가능하다.

```bash
smbclient //192.168.10.131/winShare -U root   # ls, get, put 사용 가능
```

---

### 2-3. 임시 마운트

```bash
mkdir /smbClient                                            # 마운트포인트 생성
mount -t cifs //192.168.10.131/winShare /smbClient          # 대화형 비밀번호 입력
```

옵션을 직접 지정하는 방식:

```bash
mount -t cifs //192.168.10.131/winShare /smbClient \
  -o username=root,password=1234,iocharset=utf8,vers=3.0
```

확인:

```bash
mount | grep smbClient                     # cifs 마운트·협상 버전 확인
df -h | grep smbClient                     # 용량 확인
```

출력에서 `type cifs (rw,...,vers=3.1.1,username=root,...)`처럼 SMB3로 협상된 것을 볼 수 있다.

---

### 2-4. 파일 복사 테스트

```bash
cp -r /etc/a* /smbClient/                  # 테스트 데이터 복사
ls -l /smbClient/                          # 결과 확인
```

CIFS 마운트에서는 Linux 퍼미션이 그대로 보이지 않고 `file_mode`, `dir_mode` 옵션 값(기본 0755)으로 표시된다.

---

### 2-5. fstab 영구 마운트

마운트는 서버 재부팅 시 해제되므로 자동 마운트를 설정한다.

```text
//192.168.10.131/winShare  /smbClient  cifs  username=root,password=1234,iocharset=utf8,_netdev  0 0
```

옵션 의미는 다음과 같다.

| 옵션 | 의미 |
|---|---|
| `username` | SMB 서버(Windows)의 계정 |
| `password` | SMB 서버 계정의 비밀번호 |
| `iocharset=utf8` | 한글 파일명 처리를 위한 문자셋 |
| `_netdev` | 네트워크 장치이므로 네트워크 준비 후 마운트 |
| `vers=3.0` | SMB 버전 고정(협상 실패 시 유용) |
| `uid=`, `gid=` | 마운트된 파일의 소유자·그룹 지정 |

적용·검증:

```bash
cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"   # 백업
vi /etc/fstab                              # 위 항목 추가
findmnt --verify --verbose                 # 문법 검증
systemctl daemon-reload
mount -a                                   # 적용
findmnt /smbClient                         # 최종 확인
```

---

### 2-6. credentials 파일로 비밀번호 분리(권장)

`/etc/fstab`은 누구나 읽을 수 있으므로 비밀번호를 직접 적으면 노출된다.

```bash
cat <<EOF > /root/.smbcred
username=root
password=1234
EOF

chmod 600 /root/.smbcred                   # root만 읽기 가능
```

fstab에는 credentials를 지정한다.

```text
//192.168.10.131/winShare  /smbClient  cifs  credentials=/root/.smbcred,iocharset=utf8,_netdev  0 0
```

> 실습에서는 평문 비밀번호를 쓰더라도, 운영 환경에서는 반드시 credentials 파일(권한 600)을 사용한다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 상태 확인 명령

```bash
smbclient -L 192.168.10.131 -U root        # 공유 목록 조회
mount | grep cifs                          # 마운트 옵션·버전 확인
dmesg | tail -20                           # CIFS 커널 오류 메시지 확인
```

### 3-2. 대표 오류

| 증상 | 원인·조치 |
|---|---|
| `mount: unknown filesystem type 'cifs'` | `cifs-utils` 미설치 → `dnf install -y cifs-utils` |
| `mount error(13): Permission denied` | 계정·비밀번호 오류, Windows 공유 권한 부족 |
| `mount error(112): Host is down` | SMB 버전 협상 실패 → `vers=3.0` 또는 `vers=2.1` 지정 |
| `mount error(2): No such file or directory` | 공유 이름 오타, `//IP/공유명` 확인 |
| 한글 파일명 깨짐 | `iocharset=utf8` 옵션 추가 |
| 마운트포인트 없음 | `mkdir /smbClient` 후 재시도 |

### 3-3. 연결 확인 순서

```text
1) ping 192.168.10.131            # 네트워크 도달
2) 445 포트 확인 (Windows 방화벽)
3) smbclient -L 로 인증 확인
4) mount -t cifs 로 마운트
5) mount | grep cifs 로 옵션 검증
```

> 📌 **핵심 요약**
> - Windows는 공유 폴더 자체가 SMB 서버 역할
> - Linux 클라이언트는 `samba-client` + `cifs-utils` 필요
> - `smbclient -L`로 공유 확인, `mount -t cifs`로 마운트
> - fstab 등록 시 `_netdev`와 `iocharset=utf8` 사용
> - 비밀번호는 credentials 파일(600)로 분리 권장
> - 관련: 🔗 파일 공유 개요 (NFS vs Samba/SMB) · 🐧 Linux Samba 서버 구축 · 🚨 Samba·NFS 트러블슈팅 치트시트

---
