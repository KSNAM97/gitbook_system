# 📁 vsFTP 설치 & 접근 제어 (user_list·chroot)

> **Tag:** #Linux #FTP #vsFTP #vsftpd #TCP20 #TCP21 #userlist_deny #chroot #xferlog
> **핵심 요약:** FTP는 TCP 21(제어)·20(데이터) 포트를 사용하는 평문 파일 전송 프로토콜이며, vsftpd(Very Secure FTP Daemon)는 Rocky·RHEL 등 대부분 배포판에서 기본으로 제공되는 FTP 서버다. `anonymous_enable`로 익명 접속을, `write_enable`로 업로드 권한을, `userlist_enable`·`userlist_deny`로 접속 허용/차단 대상을, `user_config_dir`로 계정별 예외 설정을, `chroot_local_user`로 홈 디렉터리 격리를 제어한다. 모든 설정 변경 후에는 `systemctl restart vsftpd`로 반영해야 한다.

---

## 1. 📖 개요 (Overview)

FTP는 네트워크를 통해 파일을 송수신(업로드·다운로드)하기 위한 TCP/IP 기반의 표준 프로토콜로, 사용자가 원격 서버에 접속해 파일을 주고받는 전용 통신 규칙이다. FTP는 TCP 포트 20·21을 사용한다.

```text
TCP 21 (FTP)       FTP Server  <----  FTP Client   (제어 연결)
TCP 20 (FTP-Data)  FTP Server  ---->  FTP Client   (데이터 전송)
```

인증 방식은 계정(ID, Password) 기반 로그인이며 평문 전송(암호화 없음)이라 보안에 취약하다.

vsFTP(Very Secure FTP Daemon)는 Linux에서 가장 널리 사용되는 FTP 서버 프로그램이다. 이름 그대로 매우 안전한 FTP 서버를 목표로 설계된 오픈소스 소프트웨어로, 보안·속도·안정성이 뛰어나 Red Hat, Rocky, CentOS, Ubuntu 등 대부분 배포판에서 기본 FTP 서버로 제공된다.

익명 사용자와 로컬 사용자 설정은 다음과 같이 구분된다. 익명 사용자(anonymous) 관련 설정:

| 항목 | 기본값 | 설명 |
|---|---|---|
| `anonymous_enable=YES` | NO | 익명 사용자 접속 허용 |
| `no_anon_password=YES` | NO | 익명 로그인 시 비밀번호 입력 없이 접속 허용 |
| `anon_root=/var/ftp` | - | 익명 사용자의 홈 디렉터리 지정 |
| `anon_upload_enable=YES` | NO | 익명 사용자의 파일 업로드 허용 |
| `anon_mkdir_write_enable=YES` | NO | 익명 사용자의 디렉터리 생성 허용 |
| `anon_other_write_enable=YES` | NO | 익명 사용자의 삭제·이름변경 허용 |
| `anon_world_readable_only=YES` | YES | 익명 사용자는 읽기 전용 디렉터리만 접근 가능 |

일반 사용자(local user) 관련 설정:

| 항목 | 기본값 | 설명 |
|---|---|---|
| `local_enable=YES` | YES | 시스템 계정(일반 사용자) FTP 접속 허용 |
| `write_enable=YES` | NO | 파일 쓰기(업로드·삭제·수정) 허용 |
| `local_umask=022` | 022 | 업로드 파일의 기본 권한 마스크 |

`write_enable=NO`로 두면 업로드는 `550 Permission denied`로 차단되고 다운로드(`get`)만 가능하다. 이 값은 리눅스 파일 권한과 별개로 동작하므로, 디렉터리 권한이 충분해도 `write_enable=NO`이면 업로드가 불가능하다.

`/etc/vsftpd/user_list`는 접근 제어 목록이며 `userlist_enable=YES`일 때 활성화된다. `userlist_deny` 값에 따라 목록의 의미가 정반대로 바뀐다.

```text
userlist_deny=YES (기본값)   : user_list에 등록된 사용자를 차단
userlist_deny=NO             : user_list에 등록된 사용자만 허용
```

기본값(`userlist_deny=YES`)은 설정 파일에 명시되어 있지 않아도 적용되며, 목록에 등록된 계정은 비밀번호 확인 단계조차 거치지 않고 즉시 거부된다.

```text
Name (192.168.10.100:none): guest
530 Permission denied.
```

전체 정책은 제한적으로 두고 특정 계정에만 예외를 주고 싶을 때 사용한다. `user_config_dir`에 지정한 디렉터리 안에 계정명과 동일한 파일을 만들면 그 사용자에게만 해당 설정이 적용된다.

```text
vsftpd.conf                             : write_enable=NO       # 전체 업로드 차단
/etc/vsftpd/userconfig/guest            : write_enable=YES      # guest만 업로드 허용
```

chroot(Change Root)는 특정 사용자 또는 프로세스를 가짜 루트(`/`) 안에 가두는 기술이다. 원래 모든 Linux 계정은 `/`를 기준으로 전체 파일 시스템을 탐색할 수 있지만, chroot를 적용하면 사용자가 보게 되는 최상위 디렉터리가 바뀐다.

```text
설정 전 접속 : /home/guest
설정 후 접속 : guest는 /home/guest를 '/'라고 인식
```

사용자는 자신의 chroot 공간 밖으로 벗어날 수 없으며, `/etc/passwd`, `/var/log` 같은 시스템 전체 구성 파일에 접근할 수 없어 보안적으로 매우 유리한 격리 환경을 만든다. 로컬 계정(guest, user1 등)을 FTP 접속용으로 사용할 경우 사용자가 의도치 않게 시스템 파일을 열어보거나 다른 사용자의 파일을 열어보는 사고가 발생할 수 있기 때문에, FTP 계정이 접속하면 자신의 홈 디렉터리 안에만 가두는 것이 매우 중요하다.

vsftpd는 보안상의 이유로 chroot 최상위 디렉터리에 쓰기 권한이 있으면 로그인을 거부한다. 사용자 홈 디렉터리는 보통 소유자에게 쓰기 권한이 있으므로, chroot 적용 시 다음 오류가 발생할 수 있다.

```text
500 OOPS: vsftpd: refusing to run with writable root inside chroot()
```

`allow_writeable_chroot=YES`를 추가하면 접속이 가능해진다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 설치 및 기동

```bash
rpm -qa | grep ftp                         # 설치 여부 확인
dnf install -y vsftpd                      # vsftpd 설치
rpm -qa | grep ftp
# vsftpd-3.0.5-8.el9.x86_64

systemctl status vsftpd                    # 기동 전 상태
systemctl start vsftpd                     # 서비스 시작
systemctl enable vsftpd                    # 부팅 시 자동 시작
systemctl status vsftpd                    # 상태 확인
```

---

### 2-2. 방화벽 개방

```bash
firewall-cmd --permanent --add-service=ftp
firewall-cmd --permanent --add-port=20/tcp
firewall-cmd --permanent --add-port=21/tcp
firewall-cmd --reload
firewall-cmd --list-port
firewall-cmd --list-services
```

방화벽을 열기 전에는 클라이언트에서 연결 시간 초과가 발생한다.

```text
C:\Users\aaa> ftp 192.168.10.100
ftp: connect :연결 시간 초과
```

---

### 2-3. 설정 파일 백업 및 익명 접속 제어

```bash
cp /etc/vsftpd/vsftpd.conf /backup         # 변경 전 원본 백업
```

익명 차단(권장):

```text
anonymous_enable=NO          # 익명 사용자 차단
local_enable=YES              # 일반 사용자 허용
```

익명 허용(실습·공개 배포 목적):

```text
#anonymous_enable=NO
anonymous_enable=YES          # 익명 사용자 허용
local_enable=YES
```

```bash
systemctl restart vsftpd
```

익명 차단 상태에서 anonymous로 접속하면 비밀번호 확인 없이 거부된다.

```text
사용자(192.168.10.100:(none)): anonymous
331 Please specify the password.
530 Login incorrect.
```

---

### 2-4. 업로드 제어

```text
write_enable=NO               # 모든 계정 업로드 차단(다운로드만 허용)
local_umask=022
```

```bash
systemctl restart vsftpd
```

```text
ftp> put b.txt
550 Permission denied.
ftp> get aliases
226 Transfer complete.                     # 다운로드는 정상
```

---

### 2-5. 전송 로그 활성화 (xferlog)

```text
dirmessage_enable=YES
xferlog_enable=YES             # 업로드·다운로드 로그
connect_from_port_20=YES
```

```bash
systemctl restart vsftpd
cat /var/log/xferlog
```

```text
Thu Jul 16 17:48:15 2026 1 ::ffff:192.168.10.131 58 /home/guest/abc.txt a _ i r guest ftp 0 * c
```

| 필드 | 의미 |
|---|---|
| 전송 시각 | 로그가 기록된 시각 |
| `1` | 전송에 걸린 시간(초) |
| `192.168.10.131` | 업/다운로드한 클라이언트 IP |
| `58` | 전송한 파일 크기(byte) |
| `/home/guest/abc.txt` | 대상 경로·파일명 |
| `i` / `o` | incoming(업로드) / outgoing(다운로드) |
| `r` / `a` | real(일반 사용자) / anonymous(익명) |
| `guest` | 사용자 계정 |
| `c` / `i` | complete(정상) / incomplete(비정상) |

---

### 2-6. 특정 사용자 차단 (user_list, 기본 정책)

```bash
vi /etc/vsftpd/user_list
```

목록 끝에 차단할 계정 추가:

```text
root
bin
daemon
...
nobody
guest                          # 차단할 계정 추가
```

```text
pam_service_name=vsftpd
userlist_enable=YES
# userlist_deny=YES            # 명시하지 않아도 기본값 적용
```

```bash
systemctl restart vsftpd
```

---

### 2-7. 계정별 개별 설정 (user_config_dir)

```bash
mkdir /etc/vsftpd/userconfig            # 계정별 설정 보관 디렉터리
ls -l /etc/vsftpd
```

```text
write_enable=NO                          # 전체 업로드 차단(메인 설정)
```

```bash
vi /etc/vsftpd/userconfig/guest
```

```text
write_enable=YES                         # guest만 업로드 허용
```

메인 설정에 반영:

```text
pam_service_name=vsftpd
userlist_enable=YES
user_config_dir=/etc/vsftpd/userconfig    # 설정 적용
```

```bash
systemctl restart vsftpd
```

결과 확인:

```text
user1 접속 → put user1.txt → 550 Permission denied
guest 접속 → put guest.txt → 226 Transfer complete
```

user1도 허용하려면 동일한 방식으로 파일을 만든다.

```bash
vi /etc/vsftpd/userconfig/user1          # write_enable=YES
systemctl restart vsftpd
```

---

### 2-8. chroot 적용

```text
chroot_local_user=YES                    # 모든 로컬 사용자에게 chroot 적용
allow_writeable_chroot=YES                # 쓰기 가능한 홈 허용
#chroot_list_enable=YES
#chroot_list_file=/etc/vsftpd/chroot_list
```

```bash
systemctl restart vsftpd
```

적용 후 클라이언트 동작:

```text
ftp> pwd
257 "/" is the current directory          # 실제로는 /home/guest
ftp> cd /etc/
550 Failed to change directory.           # 상위 디렉터리 접근 차단
ftp> ls -l
150 Here comes the directory listing.
-rw-r--r--  1 0     0     198676 Jul 10 03:13 ALL.tar.bz2   # guest 홈 내부 파일만 확인됨
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 접근 제어 동작 확인

```bash
grep -E 'userlist|chroot|write_enable|user_config' /etc/vsftpd/vsftpd.conf
cat /etc/vsftpd/user_list
ls -l /etc/vsftpd/userconfig
systemctl status vsftpd
ss -tlnp | grep :21
```

### 3-2. FTP 응답 코드

| 코드 | 의미 |
|---|---|
| 220 | 서비스 준비 완료(배너 출력) |
| 230 | 로그인 성공 |
| 331 | 사용자명 확인, 비밀번호 요구 |
| 530 | 로그인 실패 또는 접근 거부 |
| 550 | 권한 없음·파일 없음(쓰기 차단 포함) |
| 226 | 전송 완료 |

### 3-3. 트러블슈팅 시나리오

#### 🚨 시나리오 1. FTP 접속 자체가 시간 초과됨

- **원인:** 방화벽 미개방 또는 vsftpd 미동작.
- **해결:** `systemctl status vsftpd`, `firewall-cmd --list-all`로 확인 후 20/21번 포트와 ftp 서비스를 개방한다.

#### 🚨 시나리오 2. write_enable=YES인데도 업로드가 550으로 거부됨

- **원인 후보:** 설정 변경 후 데몬 미재시작, `user_config_dir` 계정 파일이 우선 적용, 디렉터리 실제 쓰기 권한 부족, chroot·SELinux 차단.
- **해결 절차:**

```bash
grep -vE '^\s*#|^$' /etc/vsftpd/vsftpd.conf   # 유효 설정만 확인
ls -l /etc/vsftpd/userconfig                   # 계정별 설정 확인
ls -ld 대상디렉터리                             # 쓰기 권한 확인
getsebool -a | grep ftp                        # SELinux ftp 관련 boolean 확인
systemctl restart vsftpd
```

#### 🚨 시나리오 3. 비밀번호도 묻지 않고 530으로 거부됨

- **원인:** `user_list`에 등록된 계정이며 `userlist_deny=YES`(기본값) 상태.
- **해결:** 접속을 허용하려면 `user_list`에서 해당 계정을 제거하거나, 허용 목록 방식(`userlist_deny=NO`)으로 전환하고 목록을 허용 계정으로 재작성한다.

#### 🚨 시나리오 4. chroot 적용 후 500 OOPS 오류로 로그인 실패

- **원인:** `chroot_local_user=YES`인데 홈 디렉터리에 쓰기 권한이 있어 vsftpd가 보안상 거부.
- **해결:** `allow_writeable_chroot=YES` 추가 후 재시작.

> 📌 **핵심 요약**
> - FTP는 제어 TCP 21, 데이터 TCP 20 사용, 평문 전송
> - `write_enable=NO`면 다운로드만 가능
> - `userlist_deny=YES`는 차단 목록(기본값), `NO`는 허용 목록
> - `user_config_dir`로 계정별 예외 설정 가능
> - `chroot_local_user=YES` + 쓰기 가능 홈은 `allow_writeable_chroot=YES` 필요
> - 관련: 🔒 SFTP 파일 전송 · 📤 SCP 파일 전송 (Linux·Windows) · 🚨 트러블슈팅 치트시트 (SSH·vsFTP·SFTP·DHCP·DNS)
