# 🧩 사용자·그룹·권한 통합 정리 — 계정 라이프사이클 한눈에

> **Tag:** #Linux #UserManagement #Group #Sudo #UPG #Summary
> **핵심 요약:** 계정 관리의 근간은 **UID(숫자 식별자)** 이며, 생성은 5곳(`/etc/passwd`·`shadow`·`group`·홈·메일박스)에 동시 기록된다. `useradd→usermod→userdel`로 라이프사이클을, `passwd -l`로 안전한 차단을, `wheel`/`sudoers`로 권한 위임을 관리한다. 이 문서는 5-1~5-3을 한 장으로 닫는 색인이다.

---

## 1. 📖 개요 (Overview)

계정·그룹·권한을 관통하는 단 하나의 원리는, 전부 **숫자 식별자(UID/GID)** 로 판정된다는 것이다. 이름(root, wheel)은 `/etc/passwd`·`/etc/group`의 매핑 별칭일 뿐, 커널은 **UID=0이면 root, GID로 그룹 권한**을 판정한다. UID=0 계정은 이름과 무관하게 전권을 가지므로 감사 시 `awk -F: '$3==0 {print $1}' /etc/passwd` 로 확인하며, 시스템 계정은 0~999, 일반 사용자는 1000~ 로 구분된다(`/etc/login.defs`).

"삭제(userdel)"보다 "잠금(passwd -l)"이 실무 기본인 이유는, 삭제는 **감사 추적성이 끊기고 파일 소유자가 UID 숫자로 남아** 재사용 시 데이터 유출 위험이 있기 때문이다. 잠금은 계정·데이터를 유지한 채 로그인만 차단해 복원·감사 여지를 남긴다. `passwd -l`(해시 앞 `!`)은 `passwd -u`로 해제하며, `passwd -d`(무비번)는 운영 금지이다. 완전 삭제는 백업 + 유예기간 후 `userdel -r`을 사용한다.

---

## 2. 🛠️ 표준 개념 정리 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 계정 관리 3대 명령 흐름
```properties
useradd  → 생성 (5곳 기록: passwd·shadow·group·홈·메일박스)
usermod  → 수정 (셸·홈·UID·그룹)
userdel  → 삭제 (-r 로 홈·메일박스까지 완전 제거)
passwd   → 비밀번호·잠금 정책
su -     → 관리자 권한 승격 (환경까지 root 로 전환)
```

### Step 2. 참조·기록 위치 총정리
| 구분 | 위치 | 역할 |
|---|---|---|
| 기록 | `/etc/passwd` | 계정 기본정보(UID/GID/홈/셸) |
| 기록 | `/etc/shadow` | 비밀번호 해시·만료 |
| 기록 | `/etc/group` | 그룹·보조그룹 구성원 |
| 기록 | 홈 디렉터리 | skel 복사 |
| 기록 | `/var/spool/mail/` | 메일박스 |
| 참조 | `/etc/default/useradd` | 기본 옵션 |
| 참조 | `/etc/login.defs` | UID/비밀번호 정책 |
| 참조 | `/etc/skel/` | 초기 환경 템플릿 |

### Step 3. 그룹 모델(UPG)과 -g / -aG 구분 ★
```bash
# UPG: useradd 시 계정과 동일 이름·GID의 전용 그룹 자동 생성
useradd user1                  # → group user1 자동 생성

usermod -g  gitA  user1        # Primary Group 교체 (홈 소유그룹 정렬용)
usermod -aG wheel user1        # Secondary Group 추가 (기존 유지 append)

# ⚠️ -a 없는 -G 는 기존 보조그룹 전체 소실 → 운영 금지
```

### Step 4. 권한 위임 2방식
```bash
# ① wheel 그룹 (RHEL 표준)
usermod -aG wheel opsuser       # sudoers 의 %wheel ALL=(ALL) ALL 라인 전제

# ② sudoers 개별 등록 (최소 권한)
visudo
#   opsuser ALL=(root) NOPASSWD: /bin/systemctl restart nginx
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 계정 생성 5-Point 검증 (+ 실제 출력)
```bash
$ id user9
uid=1009(user9) gid=1009(user9) groups=1009(user9)

$ getent passwd user9
user9:x:1009:1009:user9:/home/user9:/bin/bash

$ passwd -S user9
user9 PS 2026-07-08 0 99999 7 -1

$ ls -ld /home/user9
drwx------ 3 user9 user9 131  7월  9 10:42 /home/user9

$ ls -l /var/spool/mail/user9
-rw-rw---- 1 user9 mail 0  7월  9 10:42 /var/spool/mail/user9
```

### 3-2. 대표 함정 요약
| 함정 | 결과 | 정답 |
|---|---|---|
| `userdel`(-r 없이) | 홈·메일 잔존, 소유자 UID 숫자 | `userdel -r` |
| `usermod -d`(m 없이) | passwd만 바뀌고 홈 이동 안 됨 | `usermod -md` |
| `usermod -G`(a 없이) | 기존 보조그룹 소실 | `usermod -aG` |
| `passwd -d` | 무비번 로그인 | `passwd -l` |
| root 직접 로그인 | 감사 추적 불가 | `su -` / `sudo` |

> 📌 **핵심 요약**
> - 모든 판정은 **UID/GID 숫자**
> - 삭제보다 **잠금(passwd -l)** 이 기본
> - 그룹 추가는 **-aG**, Primary 교체만 -g
> - 관련: 5-1. 👤 리눅스 사용자 계정 관리 (useradd & usermod & userdel) · 5-2. 👥 리눅스 그룹 관리 & UPG 모델(groupadd &usermod & gpasswd) · 5-3. 🛡️ Root 접속 통제 & Sudo 권한 위임 
