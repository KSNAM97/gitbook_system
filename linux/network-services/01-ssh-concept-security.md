# 🖥️ SSH 개념 & 프로세스·보안 설정 (Telnet·Daemon·sshd_config)

> **Tag:** #Linux #SSH #Telnet #Process #Daemon #sshd_config #PermitRootLogin #Banner
> **핵심 요약:** Telnet은 TCP 23번을 사용하는 평문 원격 접속 프로토콜로 현재는 사용이 금지되며, SSH는 TCP 22번을 사용해 암호화 통신과 키 기반 인증을 제공하는 원격 접속 표준이다. 프로그램은 디스크의 실행 파일, 프로세스는 메모리에서 실행 중인 상태이며, 데몬은 백그라운드에 상주하며 요청을 기다리는 프로세스로 이름에 보통 `d`가 붙는다. SSH 접속 정책은 `/etc/ssh/sshd_config`에서 `PermitRootLogin`·`Port`·`Banner`·`MaxAuthTries`·`LoginGraceTime`·`AllowUsers`·`AllowGroups`로 제어하며, 설정 변경 후에는 반드시 데몬을 재시작해야 반영된다.

---

## 1. 📖 개요 (Overview)

Telnet은 TCP 23번 포트를 사용하는 원격 접속 프로토콜이다. GUI 기능은 없고 텍스트 기반 원격 터미널 접속만 가능하며, 평문(암호화 없음)으로 데이터를 송수신하기 때문에 패킷을 스니핑하면 계정·비밀번호·명령어가 그대로 노출된다. 보안이 취약해 보안 환경에서는 절대 사용하지 않는다.

SSH(Secure Shell)는 TCP 22번 포트를 사용하는 보안 원격 접속 프로토콜이다. Telnet과 기능은 거의 동일하지만 암호화된 통신, 사용자 인증 강화, 공개키·개인키 기반 인증(Public/Private Key)을 추가로 제공한다. 동일한 텍스트 기반 원격 터미널 접속이지만 보안성이 매우 높아 현재 모든 리눅스·서버 관리에서 표준으로 사용된다.

| 구분 | Telnet | SSH |
|---|---|---|
| 포트 | TCP 23 | TCP 22 |
| 암호화 | 없음(평문) | 있음 |
| 인증 | 비밀번호 | 비밀번호·공개키 |
| 현재 사용 | 사실상 금지 | 표준 |

Program(프로그램)은 파일 시스템(하드디스크)에 저장되어 있는 실행 가능한 파일이다. `/usr/bin/ls`, `/usr/bin/nginx`, `/usr/bin/python` 등이 해당한다. Process(프로세스)는 프로그램을 실행했을 때 그 명령어가 메모리(RAM)에 로딩되고 CPU를 사용하는 상태를 말한다. 하나의 프로그램으로 여러 개의 프로세스가 생성될 수도 있다(Apache, Nginx, Chrome 등).

```text
Program : 디스크에 저장된 실행 파일 (정적)
Process : 메모리에 올라가 실행 중인 상태 (동적)
```

Foreground Process는 사용자가 화면에서 직접 확인하면서 실행되는 프로세스로, 터미널에서 실행 중인 `vi`가 대표적이다. Background Process는 실행 중이지만 사용자 화면에 나타나지 않는 프로세스로, 웹 서버(Nginx, Apache), 데이터베이스(MySQL, MariaDB), Cron 서비스, SSH 데몬 등이 해당한다.

데몬(Daemon)은 백그라운드에서 지속적으로 실행되며 특정 요청을 처리하기 위해 대기하는 프로세스이다. 특정 조건이 발생하면 바로 동작하도록 메모리에 상주하며 기다린다. 일반 프로세스는 실행 → 작업 수행 → 종료 순으로 동작하지만, 데몬 프로세스는 계속 살아 있으며 필요한 순간에 다시 동작한다. 서버 역할을 하는 대부분의 서비스가 데몬 형태로 동작하며 일반적으로 프로그램 이름 뒤에 `d`가 붙는다.

| 데몬 | 역할 |
|---|---|
| `sshd` | SSH 서버 데몬 |
| `httpd` | Apache 웹서버 데몬 |
| `crond` | 스케줄러 데몬 |

데몬은 메모리에 상주하며 설정 파일을 캐시처럼 사용한다. 따라서 설정 파일을 수정해도 이미 실행 중인 데몬이 새 설정을 자동으로 읽지 않는다. Linux에서 데몬은 설정을 바로 인식하지 못하기 때문에 시스템을 재부팅하거나 데몬을 재시작해야 한다.

```bash
systemctl restart sshd                     # 설정 반영을 위한 재시작
```

원격 작업 중 SSH 설정을 바꾼 뒤 재시작하기 전에는 기존 세션을 유지한 채로 두어야 한다. 설정에 오류가 있어 sshd가 죽으면 콘솔 접근 없이는 복구할 수 없다.

root는 모든 시스템에 존재하는 계정명이므로 공격자가 계정명을 추측할 필요가 없다. 비밀번호만 대입하면 되기 때문에 무차별 대입 공격의 1순위 대상이 된다. `PermitRootLogin no`로 차단하면 root로 접속 시 비밀번호가 맞아도 거부된다.

```text
login as: root
root@192.168.10.100's password:
Access denied
```

일반 계정으로 접속한 뒤 `su -`나 `sudo`로 권한을 상승시키면 계정명 추측이라는 장벽이 하나 더 생기고, 누가 언제 관리자 권한을 사용했는지 감사 로그로 추적할 수 있다.

`AllowUsers`를 한 번이라도 설정하면 목록에 없는 모든 계정은 자동으로 차단되는 화이트리스트 방식이다. 계정명, 네트워크 대역(`@192.168.112.`), 계정+대역 조합까지 지정할 수 있다. `AllowGroups`는 특정 그룹 소속 계정만 허용한다.

```text
AllowUsers  guest1 guest2              # 이 두 계정만 허용
AllowUsers  @192.168.112.              # 192.168.112.0/24 대역만 허용
AllowGroups sshGroup                   # sshGroup 소속만 허용
```

가장 흔한 실수는 현재 접속에 사용 중인 계정을 목록에서 빠뜨리는 것이다. 재시작 즉시 자기 자신이 접속 불가능한 상태가 되며, 콘솔 접근 없이는 복구가 어렵다. 반드시 현재 계정을 포함시키고, 기존 세션을 유지한 채 새 세션으로 접속을 검증한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. root 직접 로그인 허용/차단

```bash
vi /etc/ssh/sshd_config
```

```text
#LoginGraceTime 2m
PermitRootLogin yes                    # root SSH 접속 허용
#StrictModes yes
```

```text
#LoginGraceTime 2m
PermitRootLogin no                     # root SSH 접속 차단(권장)
#StrictModes yes
```

```bash
systemctl restart sshd                 # 설정 변경 시 반드시 재시작
systemctl start sshd                   # 패키지 설치 후 최초 실행
systemctl enable sshd                  # 재부팅 시 자동 실행
systemctl status sshd                  # 프로세스 동작 상태 확인
```

---

### Step 2. 접속 경고 배너(Banner) 설정

```bash
vi /etc/ssh/ssh-banner
```

```text
#######################################################################
해당 시스템에 무단으로 접속시 법적 제재를 받을수 있습니다.
admin   : ryu changwan
phone   : 010-5555-1234
mail    : admin@soldesk.com
fax     : 02-555-1234
#######################################################################
```

```bash
vi /etc/ssh/sshd_config
```

```text
# no default banner path
#Banner none
Banner /etc/ssh/ssh-banner              # Banner 경로 및 파일명 설정
```

```bash
systemctl restart sshd
```

배너는 인증 전에 출력되며, 설정 전에는 표시되지 않고 로그인 화면이 바로 나온다.

```text
login as: guest
Pre-authentication banner message from server:
| #######################################################################
| 해당 시스템에 무단으로 접속시 법적 제재를 받을수 있습니다.
| ...
End of banner message from server
guest@192.168.10.100's password:
```

---

### Step 3. 포트 변경 (22 → 2002)

```bash
vi /etc/ssh/sshd_config
```

```text
#Port 22
Port 2002                               # TCP 2002번으로 설정
#AddressFamily any
```

```bash
systemctl restart sshd
```

데몬만 재시작하면 방화벽에 막혀 접속되지 않는다. 방화벽 개방이 반드시 함께 필요하다.

```bash
firewall-cmd --permanent --add-port=2002/tcp
firewall-cmd --reload
firewall-cmd --list-port
```

클라이언트 접속 시 변경 전 포트(22)로는 접속이 실패하고, 새 포트(2002)로 정상 접속된다.

```text
login as: guest
guest@192.168.10.100's password:
Last login: Thu Jul 16 11:07:48 2026 from 192.168.10.1
[guest@Server-A ~]$
```

---

### Step 4. 인증 시도·대기 시간 제한

```bash
vi /etc/ssh/sshd_config
```

```text
#LoginGraceTime 2m
PermitRootLogin yes
#StrictModes yes
#MaxAuthTries 6
MaxAuthTries 3                          # 인증 3회 실패 시 세션 종료
```

```text
#LoginGraceTime 2m
LoginGraceTime 30                       # 30초 안에 인증 성공하지 못하면 세션 종료
PermitRootLogin yes
```

```bash
systemctl restart sshd
```

3회 실패하면 다음처럼 세션이 끊기며, 30초 안에 로그인하지 않아도 세션이 종료된다.

```text
guest@192.168.10.100's password:
Access denied
guest@192.168.10.100's password:
Access denied
guest@192.168.10.100's password:
```

---

### Step 5. 특정 계정·네트워크·그룹만 접속 허용

특정 계정만 허용:

```text
AllowUsers  guest1 guest2
```

```bash
cat /etc/passwd | grep guest
# guest1:x:1009:1009:yaja:/home/guest1:/bin/bash
# guest2:x:1010:1010::/solhome/guest2:/bin/tcsh
systemctl restart sshd
```

특정 네트워크 대역만 허용:

```text
AllowUsers  *@192.168.112.*             # 192.168.112.0/24 대역만 허용
```

현재 접속 네트워크가 192.168.10.0/24라면 다음처럼 거부된다.

```text
guest@192.168.10.100's password:
Access denied
```

특정 그룹만 허용:

```bash
groupadd sshGroup                       # sshGroup 그룹 생성
useradd  sshUser1                       # sshUser1 계정 생성
usermod  -aG  sshGroup  sshUser1        # sshUser1을 sshGroup에 추가
id sshUser1
# uid=1326(sshUser1) gid=1326(sshUser1) groups=1326(sshUser1),1338(sshGroup)
```

```text
AllowGroups  sshGroup
```

```bash
systemctl restart sshd
```

sshGroup에 속하지 않은 guest는 접속이 거부되고, sshUser1은 정상 접속된다.

> **주의:** 실습 종료 후에는 `AllowUsers`·`AllowGroups` 라인을 삭제하거나 주석 처리해 원복한다. 설정을 남겨두면 이후 추가되는 계정이 의도치 않게 차단될 수 있다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
systemctl status sshd                       # 데몬 동작 상태
ss -tlnp | grep :22                         # 리스닝 포트 확인
sshd -t                                     # 문법 검사(재시작 전 필수)
sshd -T | grep -iE 'port|permitroot|allow'  # 최종 적용값 확인
tail -f /var/log/secure                     # 인증 성공·실패 로그
journalctl -u sshd -n 50                    # 서비스 로그
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. root 계정으로 SSH 접속이 안 됨

- **원인:** `PermitRootLogin no` 설정으로 root 직접 접속이 차단된 상태.
- **해결:** 일반 계정으로 접속 후 `su -` 또는 `sudo`로 권한을 상승시킨다. root 직접 접속이 반드시 필요하면 `PermitRootLogin yes`로 되돌리되, 운영 서버에서는 권장하지 않는다.

#### 🚨 시나리오 2. 포트를 2002로 바꿨는데 접속이 안 됨

- **원인:** 데몬만 재시작하고 방화벽에서 2002번 포트를 열지 않음.
- **해결 절차:**

```bash
grep ^Port /etc/ssh/sshd_config          # 1) 설정값 확인
firewall-cmd --list-port                 # 2) 방화벽 확인
firewall-cmd --permanent --add-port=2002/tcp
firewall-cmd --reload
sshd -t && systemctl restart sshd
```

#### 🚨 시나리오 3. AllowUsers 설정 후 자기 자신도 접속이 안 됨

- **원인:** `AllowUsers`에 현재 사용 중인 계정을 포함시키지 않음.
- **예방:** 원격에서 `AllowUsers`·`AllowGroups`를 설정할 때는 반드시 현재 계정을 포함시키고, 기존 세션을 열어둔 채 새 세션으로 접속을 검증한다.

#### 🚨 시나리오 4. 배너를 설정했는데 출력되지 않음

- **원인:** `vi`로 `Banner` 라인을 수정한 뒤 데몬을 재시작하지 않음.
- **해결:** `systemctl restart sshd` 후 재접속하여 확인. 데몬은 설정 파일을 메모리에 캐시하므로 재시작 없이는 절대 반영되지 않는다.

> 📌 **핵심 요약**
> - Telnet(23)은 평문, SSH(22)는 암호화 원격 접속
> - Daemon은 설정을 메모리에 유지하므로 변경 후 재시작 필수
> - `PermitRootLogin no`로 root 직접 로그인 차단이 기본 원칙
> - 포트 변경 시 방화벽 개방을 반드시 함께 처리
> - `AllowUsers`·`AllowGroups`는 화이트리스트, 자기 계정 누락 주의
> - 관련: 📤 SCP 파일 전송 (Linux·Windows) · 🔒 SFTP 파일 전송 · 🚨 트러블슈팅 치트시트 (SSH·vsFTP·SFTP·DHCP·DNS)

---
