# 리눅스 그룹 관리 & UPG 모델 (groupadd & usermod & gpasswd)

RHEL 계열의 **UPG(User Private Group)** 모델을 이해하고, `groupadd` / `usermod -g` / `usermod -aG` 를 실무 표준에 맞게 구분해서 사용하는 법 정리.

---

## 1. 개요 (Overview)

RHEL 계열이 UPG(User Private Group) 방식을 표준으로 채택한 이유는, 사용자마다 **1:1 전용 그룹**이 자동 생성되므로 `umask 002` 를 써도 타인에게 파일이 노출되지 않고, 협업이 필요할 때만 **공용 그룹 + SGID** 로 안전하게 확장할 수 있기 때문이다. `useradd user1` 을 실행하면 `user1` 계정과 **동일 이름의 그룹(GID 동일)** 이 자동 생성된다. 다른 배포판은 과거에 `users(GID=100)` 공용 그룹에 모두 소속시켰으나, 지금은 대부분 UPG로 통일되어 있다.

`usermod -g` 와 `usermod -aG` 의 차이는 운영 서버에서 사고로 이어질 수 있는데, `-g` 는 **Primary Group 교체**, `-aG` 는 **Secondary Group 추가**이다. `-a` 없이 `-G` 만 쓰면 **기존 보조 그룹 전체가 날아가** `sudo`, `docker` 권한이 통째로 사라지는 사고가 자주 발생한다. 따라서 운영 서버 표준은 **무조건 `-aG`** 이며, 리뷰 시 `usermod -G` (append 없음) 패턴은 grep으로 차단해야 한다. `-g` 는 홈 디렉터리 소유 그룹 정렬 목적으로만 제한적으로 사용한다.

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 그룹 생성 / 삭제 기본

```bash
# 그룹 생성 (GID 자동 할당)
groupadd GroupA
groupadd GroupB
groupadd GroupC

# GID를 직접 지정 (다중 서버 GID sync가 필요할 때 표준)
groupadd -g 2040 GroupD

# 그룹 삭제 ( 해당 그룹을 Primary로 쓰는 계정이 하나라도 있으면 삭제 불가)
groupdel GroupD
```
### Step 2. Primary Group 변경 (-g)

```bash
# 홈디렉터리 소유그룹을 부서 그룹으로 정렬할 때 사용
usermod -g GroupA userA1       # 그룹명으로 지정
usermod -g 1022   userA2       # GID로 지정 (동일 결과)

# 확인
id userA1
# uid=1013(userA1) gid=1022(GroupA) groups=1022(GroupA)
```

### Step 3. Secondary Group 추가 (-aG) — 운영 표준

```bash
# 기존 그룹 유지 + 신규 그룹 추가 (append)
usermod -aG GroupD userD2

# 여러 그룹 동시 추가는 콤마로 구분 (공백 X)
usermod -aG GroupB,GroupD userD3

# 그룹에서 특정 사용자만 제거
gpasswd -d userD2 GroupD
```

### Step 4. 그룹/계정 정보 확인 3종 세트

```bash
id userD3                                  # 통합 정보
getent group GroupD                        # NSS 기반 (LDAP 연동 시에도 정확)
grep GroupD /etc/group
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
id <user>                                  # gid / groups 동시 확인
getent group  <group>
getent passwd <user>
awk -F: '$4==0 {print}' /etc/passwd        # GID=0(root그룹) 소속 계정 감사
ls -ld /homeA/*                            # 홈 디렉터리 소유 그룹 검증
```

---

### 3-2. 트러블슈팅 시나리오

#### 시나리오 1. `groupdel: '<user>' 사용자의 주요 그룹을 제거할 수 없습니다`

- **원인:** 삭제하려는 그룹을 **Primary Group으로 사용 중인 계정**이 존재. UPG 모델에서는 계정과 동일 이름의 그룹이 Primary로 묶여 있어 발생.
- **해결 절차:**
    1. `getent passwd | awk -F: -v g=$(getent group <그룹> | cut -d: -f3) '$4==g {print $1}'` 로 사용 중인 계정 색출.
    2. 해당 계정의 Primary Group을 다른 그룹으로 이동: `usermod -g <다른그룹> <user>`
    3. 다시 `groupdel <그룹>` 실행.

#### 시나리오 2. `-G` 실수로 wheel/docker 그룹이 사라져 sudo·컨테이너 접근 불가

- **원인:** `usermod -G newgroup user1` 이 **덮어쓰기**로 동작 → 기존 보조 그룹 소실.
- **해결 절차:**
    1. `/var/log/secure` 로 사고 시점 확인.
    2. `usermod -aG wheel,docker,<원래그룹들> user1` 로 append 복구.
    3. 사용자 **재로그인**으로 세션 그룹 갱신.
    4. **재발 방지:** Ansible/Shell 표준 템플릿에서 `-aG` 만 허용하도록 코드리뷰 정책화.

>  **핵심 요약**
> - 그룹 생성: `groupadd -g GID groupname` — GID 중복 주의
> - 보조 그룹 추가: `usermod -aG group user` — `-a` 없으면 기존 그룹 전체 덮어씀
> - 그룹 멤버 관리: `gpasswd -a user group` (추가) / `gpasswd -d user group` (제거)
> - 그룹 변경 후 **재로그인** 필수 — `newgrp group` 으로 세션 내 즉시 전환 가능
> - 확인: `id user`, `groups user`, `getent group groupname`
> - 관련: 5-1.  리눅스 사용자 계정 관리 (useradd & usermod & userdel) · 5-3.  Root 접속 통제 & Sudo 권한 위임 · 5-4.  사용자·그룹·권한 통합 정리 · 5-5.  사용자·그룹·권한 트러블슈팅 치트시트 · 5-6.  사용자·그룹·권한 명령어 퀵 레퍼런스
