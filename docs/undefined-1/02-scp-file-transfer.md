# SCP 파일 전송 (Linux·Windows)

> **Tag:** #Linux #SCP #SSH #SecureCopy #Upload #Download #Windows
> **핵심 요약:** SCP(Secure Copy)는 SSH 프로토콜을 기반으로 원격 서버와 파일을 송수신하는 보안 전송 방식으로, SSH가 사용하는 TCP 22번 포트를 그대로 사용하며 모든 데이터가 암호화되어 전송된다. 별도의 데몬 설치가 필요 없고 SSH만 켜져 있으면 바로 사용할 수 있어 단순 백업이나 단일 파일 전송에 적합하다. 다운로드는 `scp 계정@호스트:원격경로 로컬경로`, 업로드는 `scp 로컬경로 계정@호스트:원격경로` 형식이며, 디렉터리는 `-r`이 필요하고 Windows의 cmd·PowerShell에서도 동일한 문법으로 사용할 수 있다.

---

## 1. 개요 (Overview)

SCP는 Secure Copy의 약자로, SSH 프로토콜을 기반으로 원격 서버와 파일을 송수신하는 보안 전송 방식이다. SSH에서 사용하는 TCP 22번 포트를 그대로 사용하며 모든 데이터가 암호화되어 전송된다. FTP처럼 별도의 데몬을 설치할 필요가 없고, 암호화되지 않아 보안이 취약한 FTP와 달리 SSH만 켜져 있으면 바로 사용할 수 있는 안전한 파일 전송 방식이다.

SCP의 주요 특징은 다음과 같다.

- SSH 기반으로 동작하므로 별도의 서비스 데몬이 필요 없다.
- SSH 포트(TCP 22)만 열려 있으면 사용 가능하다.
- 로그인 인증 방식도 SSH와 동일하다(계정/비밀번호 또는 공개키 인증).
- 파일 내용뿐 아니라 사용자 인증 정보, 명령어까지 모두 암호화되어 안전하다.
- 서버 간 파일 복사도 가능하다.
- 단순히 파일·디렉터리를 복사하는 목적에 최적화되어 있어 간단한 백업이나 단일 파일 전송에 많이 사용된다.

업로드와 다운로드의 문법 구조는 다음과 같다.

```text
다운로드 : scp 계정@호스트:소스경로/파일명   목적지경로     (Server → Client)
업로드   : scp 소스경로/파일명   계정@호스트:목적지경로     (Client → Server)
```

콜론(`:`)이 어느 쪽에 붙었는지가 핵심이다. 콜론이 왼쪽(소스)에 있으면 원격 파일을 가져오는 다운로드이고, 오른쪽(목적지)에 있으면 로컬 파일을 원격으로 보내는 업로드이다. 두 경우 모두 대상 계정의 비밀번호를 알고 있어야 한다.

SCP 사용 전 SSH 접속 방식은 다음과 같다.

```bash
ssh A.B.C.D                         # 현재 계정명으로 root 등 접속
ssh -l 계정명 A.B.C.D               # -l 옵션으로 계정 지정
ssh 계정명@A.B.C.D                  # @ 표기로 계정 지정
```

최초 접속 시에는 호스트 키 확인 메시지가 나타나며 `yes`를 입력해야 한다.

```text
The authenticity of host '192.168.10.100 (192.168.10.100)' can't be established.
ED25519 key fingerprint is SHA256:hSo7vPq/6MKH+FikOCDeY8V3GgrVvdPE8e+NHDAJV6g.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.10.100' (ED25519) to the list of known hosts.
```

SCP는 기본적으로 파일만 복사한다. 디렉터리를 `-r` 없이 복사하려 하면 다음과 같이 실패한다.

```text
scp: download /temp/dnf/: not a regular file
```

디렉터리는 반드시 `-r`을 붙여야 하며, 와일드카드나 중괄호 확장으로 여러 파일을 한 번에 지정할 수도 있다.

```bash
scp -r guest@192.168.10.100:/temp/dnf /client        # 디렉터리 복사
scp guest@192.168.10.100:/temp/sol-a* /client         # 와일드카드
scp guest@192.168.10.100:/SHARE/{rpc,resolv.conf,subgid,subuid} /client  # 중괄호 확장
```

> **참고:** 중괄호 확장이나 여러 소스를 나열하는 방식은 파일마다 인증을 요구할 수 있다. 반복 입력이 번거로우면 공개키 인증을 설정하거나 `tar`로 묶어 한 번에 전송하는 방법을 검토한다.

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 사전 확인 및 방화벽 개방

```bash
rpm -qa | grep openssh
# openssh-9.9p1-7.el9_8.rocky.0.1.x86_64
# openssh-clients-9.9p1-7.el9_8.rocky.0.1.x86_64
# openssh-server-9.9p1-7.el9_8.rocky.0.1.x86_64

firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
firewall-cmd --list-port
firewall-cmd --list-services
```

---

### Step 2. 다운로드 (Server → Client)

```bash
scp guest@192.168.10.100:/temp/aliases /client              # 단일 파일
scp guest@192.168.10.100:/temp/bashrc /client
scp guest@192.168.10.100:/temp/dnsmasq.conf /client
scp guest@192.168.10.100:/temp/sol-a* /client                # 와일드카드 다중 파일
scp -r guest@192.168.10.100:/temp/dnf /client                # 디렉터리
```

전송 결과 확인:

```bash
ls -l /client
# -rw-r--r-- 1 root root 1529 7월 16 12:47 aliases
# -rw-r--r-- 1 root root 2658 7월 16 12:49 bashrc
# -rw-r--r-- 1 root root 27839 7월 16 12:51 dnsmasq.conf
```

---

### Step 3. 업로드 (Client → Server)

```bash
scp /scpC/exports guest@192.168.10.100:/temp                 # 단일 파일
scp /scpC/CL-A* guest@192.168.10.100:/temp                   # 와일드카드
scp -r /scpC/project guest@192.168.10.100:/temp               # 디렉터리
```

---

### Step 4. Windows에서 SCP 사용

Windows 10 이상은 OpenSSH 클라이언트가 기본 포함되어 cmd·PowerShell에서 바로 사용할 수 있다. 리눅스는 경로 구분자로 `/`를, Windows는 `\`를 사용한다.

업로드(Windows → Linux):

```text
C:\Windows\system32> scp  C:\data\a.txt  guest@192.168.10.100:/temp
guest@192.168.10.100's password:
a.txt                                             100%   58     0.1KB/s   00:00
```

```bash
ls -ld /temp/a.txt
# -rw-r--r-- 1 guest guest 58 7월 16 15:45 /temp/a.txt
cat /temp/a.txt
# scp를 사용하여 윈도우에서 리눅스로 업로드
```

PowerShell에서도 동일하게 동작한다.

```text
PS C:\Users\aaa> scp  C:\data\b.txt  guest@192.168.10.100:/temp
guest@192.168.10.100's password:
b.txt                                             100%   58    56.6KB/s   00:00
```

다운로드(Linux → Windows):

```text
PS C:\Users\aaa> scp  guest@192.168.10.100:/etc/hosts  C:\Users\aaa\Desktop
guest@192.168.10.100's password:
hosts
```

디렉터리 전체 업로드·다운로드:

```text
PS C:\Users\ryu>  scp   -r   C:\data\project\*  guest@192.168.10.100:/home/guest/
PS C:\Users\aaa>  scp  -r  guest@192.168.10.100:/home/guest/*  C:\data\"Data Folder"
```

경로에 공백이 있으면 따옴표로 감싼다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 전송 결과 확인

```bash
ls -l /client                                 # 로컬 결과
ssh guest@192.168.10.100 'ls -l /temp'        # 원격 결과 한 번에 확인
```

### 3-2. 오류별 대응

| 오류 메시지 | 원인·대응 |
|---|---|
| `not a regular file` | 디렉터리 복사인데 `-r` 누락 |
| `Permission denied`(local) | 로컬 대상 디렉터리 쓰기 권한 부족 |
| `dest open ... Permission denied` | 원격 대상 디렉터리 쓰기 권한 부족 |
| `No such file or directory` | 경로 오타 또는 상대경로 착각 |
| `Connection refused` | sshd 미동작 또는 포트·방화벽 문제 |
| `Host key verification failed` | 서버 재설치·IP 재사용 → `ssh-keygen -R 호스트` |

### 3-3. 트러블슈팅 시나리오

#### 시나리오 1. 디렉터리 복사 시 `not a regular file` 오류

- **원인:** SCP는 기본적으로 파일만 복사하는데 디렉터리를 `-r` 없이 지정함.
- **해결:** `scp -r guest@192.168.10.100:/temp/dnf /client`처럼 `-r` 옵션을 추가한다.

#### 시나리오 2. 방화벽 미개방으로 접속 자체가 실패

```bash
firewall-cmd --list-port          # 22/tcp 포함 여부 확인
firewall-cmd --permanent --add-port=22/tcp --add-service=ssh
firewall-cmd --reload
```

#### 시나리오 3. 대용량·다수 파일 전송이 느리거나 중단됨

- **해결/대안:** SCP는 매번 전체를 다시 전송하므로 중단·재개가 필요하면 `rsync`를 검토한다.

```bash
rsync -avz --progress /src/ guest@host:/dst/     # 증분·재개 지원
```

>  **핵심 요약**
> - SCP는 SSH(22) 기반 암호화 파일 복사, 별도 데몬 불필요
> - 콜론(`:`)이 붙은 쪽이 원격, 왼쪽이면 다운로드·오른쪽이면 업로드
> - 디렉터리는 반드시 `-r` 필요
> - Windows cmd·PowerShell에서도 동일한 문법으로 사용 가능
> - 대용량·증분 전송은 `rsync` 검토
> - 관련:  SSH 개념 & 프로세스·보안 설정 ·  SFTP 파일 전송 ·  퀵 레퍼런스 (SSH·SCP·SFTP·vsFTP·DHCP·DNS)
