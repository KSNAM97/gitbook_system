# 🚑 사용자·그룹·권한 트러블슈팅 치트시트

> **Tag:** #Linux #Troubleshooting #UserManagement #Sudo #Group #CheatSheet
> **핵심 요약:** 계정·그룹·sudo 관련 장애를 "증상 → 원인 → 명령어"로 즉시 대응하는 조회용 문서. 반복 사고의 대부분은 **-m/-r 옵션 누락**, **-aG 대신 -G**, **그룹 변경 후 미재로그인** 세 가지에서 나온다.

---

## 1. 📖 개요 (Overview)

계정 장애에서 가장 먼저 봐야 할 세 가지는 ① **비밀번호 상태(`passwd -S`)**, ② **로그인 셸(`/etc/passwd`)**, ③ **그룹 세션 반영(`id` + 재로그인)** 이다. 로그인 불가의 90%가 이 셋 중 하나에서 비롯된다. "그룹 줬는데 안 된다"는 문제가 흔한 이유는 그룹 정보가 **로그인 시점에 로드**되기 때문이다. `usermod -aG` 직후 기존 세션엔 반영되지 않아 재로그인 또는 `newgrp`이 필요하다.

---

## 2. 🛠️ 증상별 즉시 대응표 (Configuration)

### 1. 계정 / 로그인
| 증상 | 원인 | 조치 |
|---|---|---|
| 계정 생성했는데 SSH 로그인 불가 | 비번 미설정(`!!`) / 셸 nologin | `passwd <u>` / `usermod -s /bin/bash <u>` |
| 홈 경로 바꿨더니 "No such directory" | `-d`만 사용(홈 미이동) | `usermod -md` 또는 `mv` 후 `-d` |
| 삭제 후 소유자가 숫자로 표시 | `-r` 누락, 홈 잔존 | `userdel -r` / `find / -uid <UID>` |
| 무비번 로그인됨 | `passwd -d` 사용 | `passwd -l` 후 `passwd` 재설정 |

### 2. 그룹 / 권한
| 증상 | 원인 | 조치 |
|---|---|---|
| 그룹 추가했는데 접근 불가 | 세션 미반영 | 재로그인 / `newgrp <g>` |
| sudo·docker 권한 사라짐 | `-G`(append 없음) | `usermod -aG <원래그룹들>` 복구 |
| `groupdel` 거부 | Primary로 사용 중 | `usermod -g <다른그룹>` 후 삭제 |
| "sudoers에 없습니다" | %wheel 주석 / 문법오류 / 미재로그인 | `visudo -c`, 주석해제, 재로그인 |

### 3. 핵심 진단 명령어
```bash
id <user>                      # UID/GID/보조그룹 (판정 기준)
passwd -S <user>               # PS(정상)/LK(잠금)/NP(비번없음)
getent passwd <user>           # NSS 기반 조회 (LDAP도 정확)
grep <user> /etc/passwd        # 셸·홈 경로 확인
sudo -l -U <user>              # 사용 가능한 sudo 명령
visudo -c                      # sudoers 문법 검증
find / -uid <UID> 2>/dev/null  # 삭제 계정 잔재 색출
journalctl _COMM=sudo -e       # sudo 실행 이력
```

---

## 3. 🔍 트러블슈팅 시나리오 (Verification & Troubleshooting)

### 🚨 시나리오 1. `usermod -d`로 홈 바꿨더니 로그인 시 디렉터리 없음
```bash
$ grep user1 /etc/passwd
user1:x:1001:1001::/new/path:/bin/bash    # passwd만 바뀜
$ ls -ld /new/path
ls: '/new/path'에 접근할 수 없음: 그런 파일이나 디렉터리 없음
```
- **해결:**
  ```bash
  mv /home/user1 /new/path        # 이미 -d만 했다면 수동 이동
  # 또는 처음부터
  usermod -md /new/path user1
  ```

### 🚨 시나리오 2. `-G` 실수로 wheel/docker 그룹 소실
```bash
$ usermod -G project user1        # ⚠️ 기존 보조그룹 덮어씀
$ id user1
uid=1001(user1) gid=1001(user1) groups=1001(user1),1500(project)  # wheel 사라짐!
```
- **해결:**
  ```bash
  usermod -aG wheel,docker,project user1   # append 복구
  # user1 재로그인
  ```

### 🚨 시나리오 3. wheel 넣었는데 "sudoers에 없습니다"
- **원인 3가지:** %wheel 주석 / 세션 미반영 / vi 편집 문법오류.
- **해결:**
  ```bash
  grep -E '^\s*%wheel' /etc/sudoers    # 주석이면 visudo로 해제
  id user1                              # wheel 소속 확인 → 없으면 usermod -aG wheel 후 재로그인
  visudo -c                             # 문법 검증
  ```

### 🚨 시나리오 4. 퇴사자를 `userdel -r`로 지웠는데 감사 데이터 요청
- **해결/예방 (퇴사자 표준 절차):**
  ```bash
  passwd -l <user>                      # ① 로그인 차단
  usermod -s /sbin/nologin <user>       # ② 셸 차단(키 우회 방지)
  # ③ 유예기간 데이터 보존·인수인계
  tar -czf /backup/leaver-<user>-$(date +%F).tar.gz /home/<user>
  userdel -r <user>                     # ④ 백업 후 완전 삭제
  ```

### 🚨 시나리오 5. NFS 환경에서 같은 사용자인데 서버마다 소유자 다름
- **원인:** 서버별 UID 상이.
- **해결:**
  ```bash
  usermod -u 1001 user1
  find / -uid <옛UID> -exec chown user1 {} \;
  # 근본: useradd -u 로 UID 명시 또는 LDAP/FreeIPA 중앙관리
  ```

> 📌 **핵심 요약**
> - 로그인 불가 3대 원인: **비번 / 셸 / 그룹 세션**
> - 홈 이동=**-md**, 그룹 추가=**-aG**, 삭제=**-r**
> - 그룹 변경 후 **재로그인** 필수
> - 관련: 5-1. 👤 리눅스 사용자 계정 관리 (useradd & usermod & userdel) · 5-2. 👥 리눅스 그룹 관리 & UPG 모델(groupadd &usermod & gpasswd) · 5-3. 🛡️ Root 접속 통제 & Sudo 권한 위임
