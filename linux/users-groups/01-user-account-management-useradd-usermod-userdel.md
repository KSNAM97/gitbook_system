 # 👤 리눅스 사용자 계정 관리 (useradd / usermod / userdel / passwd / su)

> **Tag:** #Linux #UserManagement #useradd #usermod #userdel #passwd #su #UID #GID
> **핵심 요약:** 리눅스 계정 시스템의 근간은 **UID(숫자 식별자)** 이며, 계정 생성은 `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/home`, `/var/spool/mail` 5개 위치에 **동시적으로** 정보를 기록한다. `useradd`/`usermod`/`userdel` 로 계정 라이프사이클을, `passwd` 로 비밀번호 정책을, `su -` 로 관리자 권한 승격을 관리하는 것이 실무 표준.

---

## 1. 📖 개요 (Overview)

리눅스가 계정을 이름이 아닌 UID(숫자)로 관리하는 이유는 **커널·파일시스템 레벨에서는 문자열 처리 비용이 너무 크고**, 파일 소유권·프로세스 권한 검사에 초당 수백만 번 참조되므로 **정수 비교(UID)** 로 처리하는 것이 표준이기 때문이다. 이름(`root`, `guest`)은 어디까지나 `/etc/passwd` 라는 매핑 테이블을 통해 관리자 편의를 위해 붙인 별칭이다. **UID=0 인 계정은 무조건 root 권한**을 가지며, 이름이 `root` 가 아니어도 UID 0이면 시스템 전권을 가지므로 보안 감사 시 `awk -F: '$3==0 {print $1}' /etc/passwd` 로 반드시 확인해야 한다. RHEL 7 이상은 **시스템 계정 UID 0~999**, **일반 사용자 UID 1000~** 로 분리되어 있으며(`/etc/login.defs` 의 `UID_MIN`/`UID_MAX` 정책), 다중 서버 UID sync 시 이 경계값이 다르면 NFS 소유권 오류가 발생한다.

`useradd` 를 한 번 실행하면 **5개 위치에 원자적으로 정보를 기록** 한다. `/etc/passwd`(계정 기본정보), `/etc/shadow`(비밀번호 해시), `/etc/group`(UPG 자동그룹), `홈 디렉터리`(스켈레톤 복사), `/var/spool/mail/`(메일박스)이며, 이 중 하나만 어긋나도 로그인 실패나 권한 사고가 난다. 참조 파일은 3개로, **`/etc/default/useradd`**(기본 옵션), **`/etc/login.defs`**(UID/비밀번호 정책), **`/etc/skel/`**(초기 환경파일)이 있으며, 조직 표준을 잡으려면 이 세 파일을 함께 관리해야 한다.

`userdel` 만 실행하면 홈 디렉터리와 메일박스가 그대로 남는데, 이는 `userdel` 이 **기본적으로 `/etc/passwd`, `/etc/shadow`, `/etc/group` 3개 파일에서만 항목을 제거** 하기 때문이다. 홈/메일박스는 사용자의 **데이터 자산** 으로 간주되어 보존되며, 완전 제거는 **`userdel -r`** 이 표준이다. `-r` 없이 삭제 후 같은 UID 로 새 계정을 생성하면, 옛 사용자의 홈/메일이 새 계정 소유로 잘못 매핑되어 데이터 유출 사고가 발생할 수 있으므로, 삭제 전 반드시 `find / -uid <UID> 2>/dev/null` 로 시스템 전역 잔재 파일을 확인해야 한다.

퇴사자·휴직자 계정을 처리할 때 `userdel` 대신 `passwd -l` 을 쓰는 이유는, **계정을 삭제하면 감사 로그 추적성이 끊기고 파일 소유자가 UID 숫자로 남아** 문제가 되기 때문이다. 반면 **`passwd -l` 은 계정과 데이터는 유지한 채 로그인만 차단** 하므로, 인수인계·법적 감사 대응·데이터 복원 여지를 남긴다. 완전 삭제는 유예 기간 후에 진행한다. `passwd -l` 은 shadow 파일의 비밀번호 해시 앞에 `!` 를 붙여 로그인 불가 상태로 만들며, 해제는 `passwd -u` 로 한다. `passwd -d` 는 비밀번호 자체를 삭제하는데 이는 **무비밀번호 로그인 위험**이 있어 운영 서버에서는 사용을 금지한다.

관리자 작업 시 `root` 로 직접 로그인하는 대신 `su -` 를 쓰는 이유는 **감사 추적성(Audit Trail)** 확보와 **폭발 반경 최소화** 때문이다. root 직접 로그인은 "누가 명령을 실행했는지" 추적이 불가능하고, 비밀번호 하나 유출로 시스템 전체를 잃게 된다. `su -` 로 승격하면 `/var/log/secure` 에 **원래 사용자 → root 전환 이력**이 남는다. `su` 와 `su -` 의 차이는, 후자는 **로그인 셸 환경(PATH, HOME, umask 등)까지 완전히 root 로 갱신**한다는 점이며, 관리 작업은 무조건 `su -` 또는 `sudo -i` 를 사용해야 한다. 더 세밀한 감사는 `sudo` 가 표준인데, `su` 는 root 비밀번호를 알아야 하지만 `sudo` 는 자기 비밀번호로 인증하고 명령 단위로 로그가 남는다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 계정 관리 관련 파일 7종 (필수 암기)

| 파일 / 디렉터리 | 역할 | 접근 |
|-----------------|------|------|
| `/etc/passwd` | 계정 기본정보 (ID/UID/GID/홈/셸) | 전체 읽기 |
| `/etc/shadow` | 비밀번호 해시, 만료 정책 | root 만 |
| `/etc/group` | 그룹 정의, 보조 그룹 구성원 | 전체 읽기 |
| `/etc/default/useradd` | useradd 기본 옵션값 | root 만 |
| `/etc/login.defs` | UID/GID 범위, 비밀번호 정책 | root 만 |
| `/etc/skel/` | 신규 계정 초기 환경 파일 템플릿 | 전체 읽기 |
| `/var/spool/mail/` | 사용자별 메일박스 자동 생성 | 각 사용자 |

### Step 2. `/etc/passwd` 필드 해석 (7개 필드)

```bash
# 형식 : 계정명:비밀번호:UID:GID:Comment:홈디렉터리:로그인셸
guest:x:1000:1000:guest:/home/guest:/bin/bash
#  ①    ②  ③    ④    ⑤       ⑥          ⑦
① guest         계정명 (로그인 ID)
② x             비밀번호 자리표시자 (실제는 /etc/shadow 에)
③ 1000          UID (시스템: 0~999 / 사용자: 1000~)
④ 1000          기본 그룹 GID (UPG: 계정과 동일)
⑤ guest         Comment (닉네임/실명/부서 - SSH 관리용 메타)
⑥ /home/guest   홈 디렉터리 절대경로
⑦ /bin/bash     로그인 셸
```
### Step 3. useradd 기본 옵션 조회 / 수정

```bash
# 현재 기본값 조회 (2가지 방법 - 결과 동일)
useradd -D
cat /etc/default/useradd
# GROUP=100
# HOME=/home
# INACTIVE=-1
# EXPIRE=
# SHELL=/bin/bash
# SKEL=/etc/skel
# CREATE_MAIL_SPOOL=yes

# 옵션으로 즉시 수정 (파일 자동 반영)
useradd -D -b /solhome           # 기본 홈 디렉터리 부모(base) 변경
useradd -D -s /bin/tcsh          # 기본 셸 변경
useradd -D -f 30                 # 비밀번호 만료 후 계정 비활성 대기일

# SKEL 은 옵션으로 못 바꿈 → vi 로 직접 편집
vi /etc/default/useradd
```

### Step 4. useradd 옵션 6종 (실무 필수)

```bash
# -c : Comment (실명/부서/닉네임)  → SSH 로그 추적 시 필수
# -d : 홈 디렉터리 절대경로 지정
# -m : 홈 디렉터리 자동 생성 (없으면) - -d 와 세트로 사용 ★
# -k : 참조할 skel 디렉터리 (기본값 /etc/skel 대체)
# -s : 로그인 셸 지정
# -u : UID 직접 지정 (다중 서버 sync 시)
# -g : Primary Group 지정 (사전에 groupadd 필요)
# -G : Secondary Group 지정 (콤마 구분). useradd에서는 -G만 사용; 기존 그룹 유지하며 추가 시 usermod -aG 사용

# [기본 계정 생성]
useradd user1                              # 모든 옵션 기본값 사용

# [실무 표준 조합]
useradd -u 1100 -c "HR-Kim" -s /bin/bash -md /home/hrkim  hrkim

# [부서별 skel 을 적용한 계정 생성]
useradd -c "Sales-User" -mk /etc/skel-sales -d /saleshome/user8  user8
```

### Step 5. 부서별 skel 디렉터리 표준화 (온보딩 자동화)

```bash
# [1] 부서별 skel 디렉터리 생성
mkdir /etc/skel-eng  /etc/skel-insa  /etc/skel-sales

# [2] 기본 skel 을 부서 skel 로 복제 (숨김파일 포함 → cp -a 필수)
cp -a /etc/skel/.  /etc/skel-eng
cp -a /etc/skel/.  /etc/skel-insa
cp -a /etc/skel/.  /etc/skel-sales

# [3] 부서 표준 초기파일 추가
mkdir /etc/skel-eng/Engineer   ; touch /etc/skel-eng/eng-manual
mkdir /etc/skel-insa/Insabu    ; touch /etc/skel-insa/insa-manual
mkdir /etc/skel-sales/Sales    ; touch /etc/skel-sales/sales-manual

# [4] 계정 생성 시 부서별 skel 적용
useradd -c "eng-mango"  -mk /etc/skel-eng   -d /solhome/user6  user6
useradd -c "hr-minji"   -mk /etc/skel-insa                     user7
useradd -c "sales-saja" -mk /etc/skel-sales -d /saleshome/user8 user8
```

### Step 6. passwd — 비밀번호 정책 관리 4대 옵션

```bash
# [설정 / 변경]
passwd                                     # 본인 비밀번호 변경
passwd user1                               # (root) 특정 계정 비밀번호 설정

# [상태 조회 - PS / LK / NP]
passwd -S user1
# user1 PS 2026-07-08 0 99999 7 -1 (비밀번호가 설정되어있습니다, SHA512 암호화.)
#         │       │      │    │     │  │
#         │       │      │    │     │  └─ 만료 후 비활성화까지 일수 (-1 = 사용안함)
#         │       │      │    │     └─ 만료 몇 일 전부터 경고
#         │       │      │    └─ 만료까지 일수 (99999 = 사실상 무기한)
#         │       │      └─ 최소 변경 대기 일수 (0 = 언제든 변경 가능)
#         │       └─ 최근 변경 날짜
#         └─ 상태 : PS(설정됨) / LK(잠금) / NP(비번없음)

# [잠금 / 해제 - 퇴사자·휴직자 대응 표준] ★
passwd -l user1                            # 계정 유지 + 로그인만 차단
passwd -u user1                            # 잠금 해제

# [비밀번호 삭제 - 운영 서버 사용 금지]
passwd -d user1                            # ⚠️ 무비밀번호 로그인 위험

# [만료 정책 - chage 명령]
chage -l user1                             # 정책 조회
chage -M 90 user1                          # 90일마다 비밀번호 변경 강제
chage -E 2026-12-31 user1                  # 계정 만료일 지정 (계약직 등)
```
### Step 7. usermod — 기존 계정 수정 4대 시나리오

```bash
# [셸 변경]
usermod -s /bin/tcsh user1

# [Comment 변경]
usermod -c "HR-Kim-New" hrkim

# [홈 디렉터리 변경 - ★ -md 세트 사용 필수]
# -d 만  : /etc/passwd 의 경로만 바뀌고 실제 디렉터리는 그대로 (사고 유발)
# -md   : passwd 갱신 + 실제 홈 디렉터리를 새 위치로 이동 ★ 실무 표준
usermod -md /saleshome/user1  user1

# [주의] -md 는 "기존 passwd 경로 == 실제 홈 경로" 일 때만 정상 동작
#        → 두 경로가 어긋나 있으면 mv 로 수동 이동 후 -d 만 사용해야 함

# [UID 변경]
usermod -u 2000 user1
chown -R 2000:2000 /home/user1             # 소유권 재설정 필수
```

### Step 8. userdel — 계정 삭제 2가지 모드

```bash
# [기본 삭제 - passwd/shadow/group 항목만 제거]
userdel user8
# → 홈 디렉터리(/home/user8), 메일박스(/var/spool/mail/user8) 잔존
# → ls -l 시 소유자가 UID 숫자로 표시됨 (예: "1008 1008")

# [완전 삭제 - 홈디렉터리 + 메일박스까지 제거] ★ 실무 표준
userdel -r user6

# [강제 삭제 - 로그인 세션이 있어도 강제]
userdel -rf user6                          # ⚠️ 실행 중 프로세스 데이터 손실 위험
```

### Step 9. su — 관리자 권한 승격 (`su` vs `su -`)

```bash
# [su - 환경변수 유지 (권장 X)]
guest$ su
Password: ****
# ▶ UID/EUID 는 root 로 바뀌지만 PATH/HOME 등 환경변수는 guest 그대로
# ▶ sbin 경로가 PATH 에 없어 systemctl, useradd 등이 안 먹을 수 있음
# ▶ pwd 하면 /home/guest (원래 위치 유지)

# [su - 로그인 셸 환경까지 완전 전환] ★ 실무 표준
guest$ su -
Password: ****
# ▶ root 로 실제 로그인한 것과 동일한 환경
# ▶ PATH, HOME, umask, shell rc 파일 모두 root 기준으로 재로딩
# ▶ pwd 하면 /root

# [특정 사용자로 전환]
$ su - user1                               # user1 로 로그인 셸 진입
$ su -c 'systemctl restart nginx' -        # root 로 명령 하나만 실행 후 복귀

# [원래 사용자로 복귀]
# root# exit  또는  Ctrl+D
```

### Step 10. 로그인 실습 워크플로우 (안전 접속 표준)

```bash
# [1] SSH 로는 일반 계정으로만 접속 (root 직접 접속 금지)
$ ssh guest@192.168.10.100

# [2] 관리 작업 필요 시 su - 로 승격
guest$ su -
Password: ****

# [3] 작업 완료 후 즉시 복귀 (root 세션 방치 금지)
root# exit
guest$
```
## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
# [계정 정보 통합 조회]
id <user>                                  # UID/GID/그룹 한번에
getent passwd <user>                       # NSS 기반 (LDAP 연동 시에도 정확)
grep <user> /etc/passwd /etc/group /etc/shadow

# [계정 생성 흔적 5곳 동시 검증]
ls -ld /home/<user>                        # 홈 디렉터리
ls -l  /var/spool/mail/<user>              # 메일박스
tail -1 /etc/passwd                        # 최근 생성 계정
tail -1 /etc/group
tail -1 /etc/shadow

# [보안 감사]
awk -F: '$3==0 {print $1}' /etc/passwd     # UID=0 계정 (root 외 없어야 함)
awk -F: '$2==""'  /etc/shadow              # 비밀번호 없는 계정
awk -F: '$3<1000 && $3>=500' /etc/passwd   # UID 500~999 (구버전 잔재)

# [계정 상태]
passwd -S <user>                           # PS(정상) / LK(잠금) / NP(비번없음)
chage -l <user>                            # 비밀번호 만료 정책
lastlog -u <user>                          # 마지막 로그인 이력
last <user>                                # 로그인 이력 상세
who                                        # 현재 로그인된 사용자
```

### 3-2. 계정 생성 후 5-Point 체크리스트

```bash
# 조직 표준 : 계정 생성 후 반드시 5곳 검증
CHECK_USER=user9

# [1] /etc/passwd 등록 확인
getent passwd $CHECK_USER

# [2] /etc/group 자동 그룹 생성 확인 (UPG)
getent group $CHECK_USER

# [3] /etc/shadow 비밀번호 설정 상태
passwd -S $CHECK_USER

# [4] 홈 디렉터리 존재 + 소유권
ls -ld $(getent passwd $CHECK_USER | cut -d: -f6)

# [5] 메일박스 자동 생성
ls -l /var/spool/mail/$CHECK_USER
```

---

### 3-3. 트러블슈팅 시나리오

#### 🚨 시나리오 1. `useradd` 로 계정을 만들었는데 SSH 로그인이 안 됨

- **원인 후보:**
    1. **비밀번호 미설정** → `/etc/shadow` 에 `!!` 상태 (locked).
    2. 로그인 셸이 `/sbin/nologin` 으로 설정됨.
    3. `PermitRootLogin no` 등 SSH 정책에 걸림 (root 계정인 경우).
- **해결 절차:**
    
    ```bash
    # [1] 비밀번호 상태 확인
    passwd -S user1
    # user1 LK  ...   → 잠긴 상태
    
    # [2] 비밀번호 설정
    passwd user1
    
    # [3] 셸 확인
    grep user1 /etc/passwd
    # → /sbin/nologin 이면 usermod -s /bin/bash user1
    ```
    

#### 🚨 시나리오 2. `usermod -d /new/path user1` 로 홈 경로를 바꿨는데 로그인 시 "No such directory"

- **원인:** `-d` 만 사용해서 **`/etc/passwd` 상의 경로는 바뀌었지만 실제 디렉터리는 이동 안 됨**. 로그인 시 존재하지 않는 경로 진입 시도 → 오류.
- **해결:**
    
    ```bash
    # 방법 ① : 이미 -d 만 실행한 상태라면 → mv 로 실제 이동
    mv /home/user1  /new/path
    ls -ld /new/path
    
    # 방법 ② : 처음부터 -md 사용 (권장)
    usermod -md /new/path user1
    ```
    

#### 🚨 시나리오 3. `userdel user8` 후 `ls -l /home` 하니 소유자가 이름이 아닌 숫자(1008) 로 표시됨

- **원인:** 계정만 삭제되고 홈 디렉터리는 잔존 → 시스템이 UID→이름 매핑에 실패해 숫자로 표시.
- **위험성:** 같은 UID 로 새 계정을 만들면 **옛 사용자의 홈/파일이 자동으로 새 계정 소유로 매핑**되어 데이터 유출 사고.
- **해결/예방:**
    
    ```bash
    # [사후 조치]
    find / -uid 1008 2>/dev/null               # 잔재 파일 전체 색출
    rm -rf /home/user8
    rm -f  /var/spool/mail/user8
    
    # [사전 예방] 계정 삭제는 무조건 -r 옵션
    userdel -r user8
    ```
#### 🚨 시나리오 4. 퇴사자 계정을 `userdel -r` 로 지웠는데 감사팀에서 데이터 요청이 옴

- **원인:** 완전 삭제로 데이터 복구 불가. 인수인계 전에 삭제해버림.
- **해결/예방:**
    
    ```bash
    # [퇴사자 처리 표준 절차]
    # [1] 즉시 로그인 차단 (계정/데이터는 유지)
    passwd -l <user>
    
    # [2] 셸을 nologin 으로 (SSH 키 우회 차단)
    usermod -s /sbin/nologin <user>
    
    # [3] 유예기간(예: 90일) 동안 데이터 보존 + 인수인계 완료
    
    # [4] 백업 후 완전 삭제
    tar -czf /backup/leaver-<user>-$(date +%F).tar.gz /home/<user>
    userdel -r <user>
    ```
    

#### 🚨 시나리오 5. `su -` 로 root 진입 후 세션을 방치했다가 다른 사람이 root 명령 실행

- **원인:** root 승격 상태로 자리를 비움 → 물리적/원격 접근 사고.
- **해결/예방:**
    
    ```bash
    # [1] 즉시 조치 - 세션 강제 종료
    who                                     # root 세션 확인
    pkill -KILL -t pts/1                    # 해당 tty 세션 kill
    
    # [2] 예방 - root 세션 자동 타임아웃
    echo "TMOUT=300" >> /root/.bashrc       # 5분 무입력 시 자동 로그아웃
    echo "readonly TMOUT" >> /root/.bashrc  # 사용자가 못 바꾸도록
    
    # [3] 근본 해결 - sudo 로 전환 (5-2 문서 참조)
    #     su - 대신 sudo -i / sudo 명령 사용 → 감사 로그 자동 기록
    ```
    

#### 🚨 시나리오 6. `passwd -d user1` 실행 후 user1 이 비밀번호 없이 로그인해버림

- **원인:** `-d` 는 shadow 파일의 비밀번호 해시를 빈 값으로 만듦 → **인증 우회 상태**.
- **해결:**
    
    ```bash
    # 즉시 잠금
    passwd -l user1
    
    # 재설정
    passwd user1
    
    # [예방] 팀 표준 문서에 passwd -d 사용 금지 명시
    #        휴직/일시 차단은 무조건 passwd -l 사용
    ```
    

#### 🚨 시나리오 7. 여러 서버에서 같은 사용자인데 파일 소유자가 서버마다 다르게 보임 (NFS 환경)

- **원인:** 서버마다 `useradd` 를 각자 실행 → **UID 가 서버별로 다르게 할당됨**.
- **해결:**
    
    ```bash
    # [단기 조치] UID 강제 정렬
    usermod -u 1001 user1
    find / -uid <옛UID> -exec chown user1 {} \;
    
    # [근본 해결]
    # ① 계정 생성 시 -u 옵션으로 UID 명시적 지정
    useradd -u 1001 -c "..." user1
    
    # ② 중앙 인증(LDAP / FreeIPA / Active Directory) 도입
    #    → 계정/UID 를 중앙에서 단일 관리
    ```

---

> 📌 **핵심 요약**
> - 계정 생성: `useradd -m -s /bin/bash -c "설명" user` — `-m` 홈 자동 생성, `-s` 셸 지정
> - 비밀번호: `passwd user` (root), `echo "pw" | passwd --stdin user` (스크립트용)
> - 계정 수정: `usermod -l 새이름`, `-md 새홈`, `-e YYYY-MM-DD` (만료일)
> - 계정 잠금/해제: `passwd -l user` / `passwd -u user` (삭제 전 잠금이 안전)
> - 계정 삭제: `userdel -r user` (홈+메일 함께 삭제) — `id user` 로 존재 확인 후 실행
> - 관련: 5-2. 👥 리눅스 그룹 관리 & UPG 모델(groupadd &usermod & gpasswd) · 5-3. 🛡️ Root 접속 통제 & Sudo 권한 위임 · 5-4. 🧩 사용자·그룹·권한 통합 정리 · 5-5. 🚑 사용자·그룹·권한 트러블슈팅 치트시트 · 5-6. ⚡ 사용자·그룹·권한 명령어 퀵 레퍼런스
