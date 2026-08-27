# ⚡ 사용자·그룹·권한 명령어 퀵 레퍼런스

> **Tag:** #Linux #QuickReference #useradd #usermod #passwd #group #sudo #CheatSheet  
> **핵심 요약:** useradd·usermod·userdel·passwd·group·su·sudo 문법을 빠르게 조회하는 암기 카드. 이해가 아니라 "조회·복붙"이 목적.

---

## 1. 🛠️ 명령어 문법 (Configuration)

### 1-1. useradd (계정 생성)

```bash
useradd user1                           # 기본 설정으로 user1 생성
useradd -u 1100 user1                   # UID를 1100으로 지정
useradd -c "HR-Kim" user1               # Comment에 실명·부서 정보 지정
useradd -s /bin/bash user1              # 로그인 셸을 /bin/bash로 지정
useradd -m -d /home/user1 user1         # 홈 디렉터리 생성 및 경로 지정
useradd -m -k /etc/skel-a -d /gitA/u1 u1 # 사용자별 Skel과 홈 경로 지정
useradd -D                              # useradd 기본값 조회
useradd -D -s /bin/tcsh                 # 이후 생성 계정의 기본 셸 변경
```

### 1-2. usermod (계정 수정)

```bash
usermod -s /bin/tcsh user1              # 로그인 셸 변경
usermod -c "New" user1                  # Comment 필드 변경
usermod -m -d /new/path user1           # 홈 경로 변경 및 기존 데이터 이동
usermod -u 2000 user1                   # UID 변경
usermod -g gitA user1                   # Primary Group을 gitA로 변경
usermod -aG wheel user1                 # 기존 보조 그룹을 유지하며 wheel 추가
```

UID 변경 후 확인:

```bash
find / -xdev -uid <이전UID> -ls 2>/dev/null # 이전 UID 소유 파일 검색
```

### 1-3. userdel / passwd

```bash
userdel user1                            # 계정만 삭제하고 홈·메일은 유지
userdel -r user1                         # 계정과 홈·메일 스풀 삭제
userdel -f -r user1                      # 강제 삭제 시도: 운영 환경 주의

passwd user1                             # 비밀번호 설정 또는 변경
passwd -S user1                          # 비밀번호 상태 확인
passwd -l user1                          # 비밀번호 인증 잠금
passwd -u user1                          # 비밀번호 인증 잠금 해제
passwd -d user1                          # 비밀번호 삭제: 운영 환경 사용 주의

chage -M 90 user1                        # 비밀번호 최대 사용 기간 90일
chage -E 2026-12-31 user1                # 계정 만료일 설정
chage -l user1                           # 계정·비밀번호 만료 정책 조회
```

> `passwd -l`은 비밀번호 인증을 잠근다. SSH 공개키, sudo 정책, 실행 중인 세션 등 다른 접근 경로까지 모두 차단한다고 가정하지 않는다.

### 1-4. group / gpasswd

```bash
groupadd GroupA                          # GroupA 그룹 생성
groupadd -g 2040 GroupD                  # GID 2040으로 GroupD 생성
groupdel GroupD                          # GroupD 삭제
gpasswd -a user1 GroupD                  # user1을 GroupD에 추가
gpasswd -d user1 GroupD                  # user1을 GroupD에서 제거
getent group GroupD                      # NSS 기반 그룹 정보 조회
```

### 1-5. su / sudo

```bash
su                                       # root로 전환하되 현재 환경 일부 유지
su -                                     # root 로그인 셸로 전환
su - user1                               # user1 로그인 환경으로 전환
su -c 'systemctl restart nginx' -        # root로 지정 명령 하나 실행

sudo -i                                  # sudo를 이용한 root 로그인 셸
sudo -l                                  # 현재 사용자의 sudo 권한 조회
sudo -l -U user1                         # user1의 sudo 권한 조회
visudo                                   # sudoers를 잠금·검증하며 편집
visudo -c                                # sudoers 전체 문법 검사
```

---

## 2. 🔢 빠른 조회표 (Configuration)

### 2-1. `/etc/passwd` 7필드

```properties
계정명 : x : UID : GID : Comment : 홈디렉터리 : 로그인셸
guest  : x :1000 :1000 : guest   : /home/guest : /bin/bash
#  ①      ②   ③     ④      ⑤          ⑥            ⑦
```

### 2-2. `passwd -S` 상태값

| 코드 | 의미 |
|---|---|
| PS | 비밀번호 설정됨 |
| LK | 비밀번호 잠금 |
| NP | 비밀번호 없음 |

### 2-3. UID 대역 (RHEL 계열 기본값)

| 대역 | 용도 |
|---|---|
| 0 | root |
| 1~999 | 일반적인 시스템 계정 범위 |
| 1000~ | 일반 사용자 기본 범위 |

> 실제 범위는 `/etc/login.defs`의 `SYS_UID_MIN`, `SYS_UID_MAX`, `UID_MIN`, `UID_MAX` 설정에 따라 달라질 수 있다.

### 2-4. `-g` vs `-aG`

```properties
-g   → Primary Group 교체
-aG  → Secondary Group 추가
-G   → 지정 목록으로 보조 그룹 교체
```

> `usermod -G`를 `-a` 없이 사용하면 기존 보조 그룹이 제거될 수 있다.

---

## 3. 🔍 검증 명령어 모음 (Verification)

```bash
id <user>                               # UID·GID·보조 그룹 확인
getent passwd <user>                    # NSS 기반 계정 정보 조회
getent group <group>                    # NSS 기반 그룹 정보 조회
passwd -S <user>                        # 비밀번호 상태 확인
chage -l <user>                         # 계정·비밀번호 만료 정책 확인
sudo -l -U <user>                       # sudo 허용 명령 조회

last <user>                             # 로그인 기록 확인
lastlog -u <user>                       # 마지막 로그인 정보 확인
awk -F: '$3==0 {print $1}' /etc/passwd  # UID 0을 사용하는 계정 감사
find / -xdev -uid <UID> -ls 2>/dev/null # 특정 UID 소유 파일 검색
```

> 📌 **핵심 요약**
> - 생성: `useradd -m -d`
> - 수정: `usermod -aG`
> - 삭제: `userdel -r`
> - 비밀번호 인증 잠금: `passwd -l`
> - Primary Group: `-g`
> - Secondary Group 추가: `-aG`
> - sudoers는 `visudo`로 편집
> - 관련: 5-1. 👤 리눅스 사용자 계정 관리 (useradd & usermod & userdel) · 5-2. 👥 리눅스 그룹 관리 & UPG 모델(groupadd &usermod & gpasswd) · 5-3. 🛡️ Root 접속 통제 & Sudo 권한 위임 · 5-7. 🏗️ 종합실습 부서별 계정·홈디렉터리·Skel 표준화 시나리오
