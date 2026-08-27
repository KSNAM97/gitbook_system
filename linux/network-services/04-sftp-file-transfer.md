# 🔒 SFTP 파일 전송

> **Tag:** #Linux #SFTP #SSH #TCP22 #put #get #lcd #FileTransfer
> **핵심 요약:** SFTP는 SSH(TCP 22) 세션 안에서 동작하는 파일 전송 전용 프로토콜로, 기존 FTP(TCP 20·21)와 구조적으로 완전히 다르며 vsftpd와도 무관하다. SSH가 설치되어 있으면 `internal-sftp` 또는 `sftp-server`를 통해 자동으로 SFTP 기능이 제공되므로 별도 포트 개방이 필요 없다. 인증 정보와 파일 내용이 모두 암호화 터널 안에서 전송되어 안전하며, 업로드(`put`)는 로컬 현재 디렉터리 기준, 다운로드(`get`)는 원격 경로 지정이 가능하다.

---

## 1. 📖 개요 (Overview)

FTP는 TCP/IP 기반으로 Client와 Server가 상호 간 파일을 전송하기 위해 사용하는 오래된 파일 전송 프로토콜이다. 암호화되지 않은 평문 전송 방식이기 때문에 보안에 취약하며 포트 20(데이터)·21(제어)을 사용한다.

SSH(Secure Shell)는 인터넷을 통한 원격 접속 시 암호화를 제공해 안전한 통신을 가능하게 하는 보안 프로토콜이다. 데이터 전송은 대칭키 암호화 방식으로 보호하며, 사용자 인증은 비대칭키(공개키/개인키) 또는 비밀번호 기반 인증을 사용한다. 기본 포트는 TCP 22번이다.

SFTP는 SSH(22번 포트) 세션 안에서 동작하는 파일 전송 전용 프로토콜이다. 기존 FTP(20/21번 포트)와는 구조적으로 완전히 다르며, FTP 서버 프로그램(vsftpd)과도 아무 관련이 없다. SSH가 설치되어 있으면 SSH 서버가 자동으로 SFTP 기능을 제공한다(예: `/usr/libexec/openssh/sftp-server` 또는 `internal-sftp`). 모든 데이터(인증 정보·파일 내용)가 SSH 암호화 터널 안에서 전송되기 때문에 매우 안전하며, 별도의 FTP 포트 개방이 필요 없이 TCP 22번만 개방하면 된다.

| 구분 | FTP | SFTP |
|---|---|---|
| 포트 | TCP 20, 21 | TCP 22 |
| 암호화 | 없음(평문) | SSH 터널로 암호화 |
| 서버 데몬 | vsftpd 등 별도 설치 | sshd에 내장 |
| 방화벽 | 20·21 및 Passive 범위 | 22번만 |

SFTP 세션에는 로컬 현재 디렉터리와 원격 현재 디렉터리가 각각 존재한다.

```text
upload 조건   : 업로드할 디렉터리에서 접속해야 한다. 파일이 있는 폴더로 이동 후 SFTP 접속.
download 조건 : download는 경로를 지정할 수 있기 때문에 어떤 디렉터리에서 실행해도 관계없다.
```

경로를 지정하지 않고 파일을 업로드하면 sol 계정이 자신의 홈 디렉터리에 있다고 가정하고 검색하므로 다음과 같은 오류가 발생한다.

```text
sftp> put passwd
stat passwd: No such file or direct    # sol 계정의 위치는 sol 계정의 홈 디렉터리이므로 passwd 파일이 검색되지 않는다
```

파일이 있는 위치로 이동한 뒤 접속해야 한다.

```bash
cd /temp                                # 업로드할 파일이 있는 위치로 이동
pwd
# /temp
```

---

## 2. 🛠️ 표준 사용법 (Configuration)

### 2-1. SSH 접속 방법

```bash
ssh A.B.C.D                             # 현재 계정으로 접속
ssh -l 계정명 A.B.C.D                   # 계정 지정 방식 1
ssh 계정명@A.B.C.D                      # 계정 지정 방식 2
```

---

### 2-2. SFTP 접속

```bash
sftp guest@192.168.10.100
```

```text
The authenticity of host '192.168.10.100 (192.168.10.100)' can't be established.
ED25519 key fingerprint is SHA256:hSo7vPq/6MKH+FikOCDeY8V3GgrVvdPE8e+NHDAJV6g.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.10.100' (ED25519) to the list of known hosts.
guest@192.168.10.100's password:
Connected to 192.168.10.100.
sftp>
```

---

### 2-3. 세션 내 기본 명령

```text
pwd            원격 현재 디렉터리
lpwd           로컬 현재 디렉터리
ls -l          원격 목록
lls -l         로컬 목록
cd 경로        원격 디렉터리 이동
lcd 경로       로컬 디렉터리 이동
put 파일       업로드
get 파일       다운로드
exit / quit    종료
```

---

### 2-4. 업로드 실습

파일이 있는 로컬 디렉터리로 이동한 뒤 접속해야 업로드가 가능하다.

```bash
cd /temp
ls -l passwd
# -rw-r--r-- 1 sol sol 2206  7월 20 11:10 passwd
```

```bash
sftp guest@192.168.10.100
```

```text
sftp> put  passwd
Uploading passwd to /home/guest/passwd
passwd                                  100% 2206   940.5KB/s   00:00

sftp> ls -l passwd
-rw-r--r--    ? guest    guest        2206 Jul 20 11:37 passwd
```

원격 특정 디렉터리로 업로드하는 방법 2가지:

```text
# 방법 1: 원격 디렉터리로 이동 후 업로드
sftp> cd  /home/guest/linux-A
sftp> put  sol-c*

# 방법 2: 목적지 경로를 직접 지정해 업로드
sftp> cd  /home/guest
sftp> put  lo*   ./linux-A/
```

---

### 2-5. 다운로드 실습

다운로드는 원격 경로를 직접 지정할 수 있어 접속 위치와 무관하다.

```text
sftp> get  /temp/host*                  # 로컬 현재 디렉터리로 다운로드
sftp> get  /temp/*.conf
sftp> get  *so*
```

로컬 대상 디렉터리를 지정하는 방식:

```text
sftp> get  /temp/host*  ./solLinux-A/
sftp> get  /temp/*so*   ./solCisco-A
```

디렉터리까지 포함:

```text
sftp> get -r ./*                        # 파일 및 디렉터리 전체
```

---

### 2-6. Windows에서 SFTP 사용

Windows 10 이상은 OpenSSH 클라이언트가 기본 포함되어 cmd·PowerShell에서 바로 사용할 수 있다.

```text
C:\sol> sftp  guest@192.168.10.100
guest@192.168.10.100's password:
Connected to 192.168.10.100.
sftp> get  ./*                          # 파일만 다운로드
sftp> get  -r  ./*                      # 파일 및 디렉터리 다운로드
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 업로드 시 파일을 못 찾을 때

```text
sftp> lpwd                              # 로컬 현재 위치 확인
sftp> lls -l                            # 로컬 파일 목록 확인
sftp> lcd /temp                         # 로컬 디렉터리 이동
sftp> put passwd
```

### 3-2. Permission denied가 발생할 때

```bash
ls -ld /업로드대상디렉터리               # 원격 디렉터리 권한 확인
id 사용자명                              # 소유·그룹 확인
```

원격 디렉터리에 해당 계정의 쓰기 권한이 있어야 업로드가 가능하다.

### 3-3. 호스트 키 경고가 발생할 때

서버를 재설치하거나 IP를 재사용하면 호스트 키 불일치 경고가 나타난다.

```bash
ssh-keygen -R 192.168.10.100            # known_hosts에서 해당 항목 제거
```

### 3-4. 트러블슈팅 시나리오

#### 🚨 시나리오 1. `stat passwd: No such file or directory` 오류

- **원인:** sol 계정의 현재 위치(홈 디렉터리)에 업로드할 파일이 없음. SFTP는 SSH 로그인 시점의 계정 홈 디렉터리를 기준으로 로컬 파일을 찾는다.
- **해결:** SFTP 접속 전에 파일이 있는 디렉터리로 먼저 이동한다.

```bash
sftp> exit
pwd
# /home/sol
cd /temp
pwd
# /temp
sftp guest@192.168.10.100
sftp> put passwd
```

#### 🚨 시나리오 2. 다운로드 위치를 헷갈려 파일이 엉뚱한 곳에 저장됨

- **원인:** `get`은 원격 경로를 지정할 수 있지만 로컬 저장 위치는 지정하지 않으면 현재 로컬 디렉터리가 기본값.
- **해결:** 대상 디렉터리를 명시적으로 지정한다(`get /temp/host* ./solLinux-A/`).

> 📌 **핵심 요약**
> - SFTP는 SSH(TCP 22) 기반 파일 전송으로 vsftpd와 무관
> - 인증·파일 내용이 모두 암호화되어 안전
> - 업로드는 로컬 현재 디렉터리 기준, `lcd`로 변경 가능
> - 다운로드는 원격 경로 지정 가능, 접속 위치와 무관
> - Windows cmd·PowerShell에서도 동일한 세션 명령 사용 가능
> - 관련: 🖥️ SSH 개념 & 프로세스·보안 설정 · 📤 SCP 파일 전송 (Linux·Windows) · 📁 vsFTP 설치 & 접근 제어 (user_list·chroot)
