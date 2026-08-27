# 종합실습 부서별 계정·홈디렉터리·Skel 표준화 시나리오

## 개요 (Overview)

부서별로 `/etc/skel-a`, `/etc/skel-b` 처럼 skel 디렉터리를 분리 운영하는 이유는, 신규 입사자 계정을 만들 때 **부서 표준 초기 파일(매뉴얼, dotfile, VPN 설정 스크립트 등)** 을 자동 배포해 **온보딩 시간을 단축**하기 위해서다. `useradd -mk /etc/skel-a` 처럼 `-k` 옵션으로 참조 skel을 지정하면 계정별 초기환경이 자동 프로비저닝되며, 조직 표준을 아예 `/etc/default/useradd` 의 `SKEL=` 로 강제하면 실수를 방지할 수 있다.

홈 디렉터리를 `/home` 이 아닌 `/gitA`, `/gitB` 처럼 프로젝트별로 분리하는 실무 이점은, **디스크 쿼터/스냅샷/백업 정책을 부서 단위로 분리 적용**할 수 있고, LVM/ZFS 파티션을 부서별로 잘라 **I/O 격리**까지 가능하기 때문이다. 부서 폐지 시 `/gitA` 만 tar/삭제하면 정리가 완료되어 운영 부담이 최소화되고, 백업 스케줄을 부서 SLA에 맞춰 다르게 적용할 수 있다(HQ=매일, A=주간 등).

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 사전 준비 — root 직접 로그인 차단 + guest에 sudo 위임

먼저 root 계정에서 `vi` 로 SSH 설정 파일을 열어 root 직접 로그인을 차단합니다.

```bash
# root 계정에서 수행
[root@localhost ~]# vi /etc/ssh/sshd_config
```

편집 화면에서 `PermitRootLogin` 항목을 찾아 **주석(`#`)을 제거하고 값을 `no` 로 변경**합니다.

```text
     35 # Authentication:
     36
     37 #LoginGraceTime 2m
     38 #PermitRootLogin yes
     39 PermitRootLogin no        <---- '#' 주석 제거 후 값을 yes → no 로 변경
     40 #StrictModes yes
     41 #MaxAuthTries 6
     42 #MaxSessions 10
: wq
```

편집을 마쳤으면 저장(`:wq`) 후 SSH 데몬을 재시작해 설정을 반영합니다.

```bash
[root@localhost ~]# systemctl restart sshd
```

이어서 guest 계정을 `wheel` 그룹에 추가해 sudo 권한을 위임합니다.

```bash
[root@localhost ~]# usermod -aG wheel guest       # guest 계정에 sudo 권한 위임
[root@localhost ~]# id guest
uid=1000(guest) gid=1000(guest) groups=1000(guest),10(wheel)
```
> **참고:** ⚠️ 로그의 주석("yes를 no로 변경")은 두 동작이 합쳐진 표현입니다. 정확한 목표는 **"`#PermitRootLogin yes` 의 `#` 주석을 제거하고, 값을 `no` 로 설정"** 이며, 최종 파일 상태는 `PermitRootLogin no` 입니다.
### Step 2. 프로젝트 홈 루트 3개 생성

```bash
# 이후 모든 작업은 guest 계정 + sudo 로 수행 (감사로그 확보 목적)
sudo mkdir /gitHQ /gitA /gitB
ls -ld /gitHQ /gitA /gitB
```

### Step 3. useradd 조직 표준값 세팅 — `/etc/default/useradd`

```properties
# sudo vi /etc/default/useradd
GROUP=100
HOME=/gitHQ           # 신규 계정 홈 루트를 프로젝트 디렉터리로 강제
INACTIVE=-1
EXPIRE=
SHELL=/bin/tcsh       # 조직 표준 셸 통일
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
```

```bash
sudo useradd -D       # 반영 확인
```

### Step 4. 부서별 skel 디렉터리 생성

```bash
sudo mkdir /etc/skel-a /etc/skel-b

# 원본 skel(숨김파일 포함) 복사 → -a 옵션 필수 (권한/속성 보존)
sudo cp -a /etc/skel/. /etc/skel-a
sudo cp -a /etc/skel/. /etc/skel-b

# 부서 표준 초기파일 배포 (매뉴얼/디렉터리 등)
# ※ 문제 지문의 '/etc/skel-g' 는 '/etc/skel-b' 의 오타
sudo mkdir /etc/skel/GIT-HQ  /etc/skel-a/GIT-A  /etc/skel-b/GIT-B
sudo touch /etc/skel/GIT-HQ-Manual  /etc/skel-a/GIT-A-Manual  /etc/skel-b/GIT-B-Manual
```

### Step 5. 계정 프로비저닝 (UID 대역을 부서별로 분리)

```bash
# HQ 부서 (UID 1100번대) — 기본 skel + Step3 기본값(HOME=/gitHQ) 상속
sudo useradd -u 1100 -c "HQ-User1" -s /bin/bash huser1
sudo useradd -u 1101 -c "HQ-User2"               huser2   # 기본 shell(tcsh) 상속
sudo useradd -u 1102 -c "HQ-User3" -s /bin/csh  huser3

# A팀 (UID 1200번대) — skel-a 사용
sudo useradd -u 1200 -c "gitA_User1" -s /bin/tcsh -mk /etc/skel-a -d /gitA/gitA_User1 auser1
sudo useradd -u 1201 -c "gitA_User2" -s /bin/csh  -mk /etc/skel-a -d /gitA/gitA_User2 auser2
sudo useradd -u 1202 -c "gitA_User3" -s /bin/bash -mk /etc/skel-a -d /gitA/gitA_User3 auser3

# B팀 (UID 1300번대) — skel-b 사용
sudo useradd -u 1300 -c "gitB_User1" -s /bin/tcsh -mk /etc/skel-b -d /gitB/gitB_User1 buser1
sudo useradd -u 1301 -c "gitB_User2" -s /bin/bash -mk /etc/skel-b -d /gitB/gitB_User2 buser2
sudo useradd -u 1302 -c "gitB_User3" -s /bin/csh  -mk /etc/skel-b -d /gitB/gitB_User3 buser3
```

> **참고:** 💡 huser1~3 은 Step 3에서 `HOME=/gitHQ`, `SKEL=/etc/skel` 기본값이 설정된 상태이므로 `-d`, `-k` 옵션 없이도 `/gitHQ/userXXX` 에 기본 skel이 적용됩니다. 반면 auser/buser 는 홈 경로와 skel이 기본값과 다르므로 `-mk`, `-d` 를 명시해야 합니다.

### Step 6. 부서 그룹 생성 & Primary/Secondary 할당

```bash
# 부서 그룹 (GID 예약)
sudo groupadd -g 1110 gitHQ
sudo groupadd -g 1220 gitA
sudo groupadd -g 1330 gitB

# Primary Group 정렬 → 홈디렉터리 소유그룹이 부서 그룹으로 통일
for u in huser1 huser2 huser3; do sudo usermod -g gitHQ $u; done
for u in auser1 auser2 auser3; do sudo usermod -g gitA  $u; done
for u in buser1 buser2 buser3; do sudo usermod -g gitB  $u; done

# 협업용 보조 그룹 부여 (기존 그룹 유지 → -aG 필수)
sudo groupadd groupHQ groupA groupB
sudo usermod -aG groupA           huser1     # HQ + A 프로젝트 협업
sudo usermod -aG groupB           auser1     # A  + B 프로젝트 협업
sudo usermod -aG groupHQ          buser1     # B  + HQ 협업
```

> **참고:** 📌 로그 확인 결과, 보조 그룹은 `groupHQ`, `groupA`, `groupB` 순서로 생성되어 각각 GID `1331`, `1332`, `1333` 이 자동 할당되었습니다(`id` 출력의 `groupA=1331`, `groupB=1332`, `groupHQ=1334` 참고 — 중간에 다른 GID 소비가 있었던 것으로 보이므로 실제 값은 `id` 명령으로 반드시 확인).

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 최종 검증 체크리스트

```bash
# [1] 홈디렉터리 소유권/그룹이 부서 그룹으로 정렬됐는지
ls -l /gitHQ /gitA /gitB

# [2] 계정별 UID/GID/보조그룹 매핑 확인
for u in huser1 auser1 buser1; do id $u; done

# [3] skel 초기파일이 홈에 정상 배포됐는지
sudo ls -la /gitA/gitA_User1 | grep -E 'GIT-A|manual'

# [4] shell이 요구사항대로 설정됐는지
grep -E '^(huser|auser|buser)' /etc/passwd

# [5] guest → sudo 위임이 유효한지
sudo -l -U guest
```

기대 결과 예시(로그 기준):

```properties
uid=1100(huser1) gid=1110(gitHQ) groups=1110(gitHQ),<GID>(groupA)
uid=1200(auser1) gid=1220(gitA)  groups=1220(gitA),<GID>(groupB)
uid=1300(buser1) gid=1330(gitB)  groups=1330(gitB),<GID>(groupHQ)
```

### 3-2. 대표 트러블슈팅

#### 🚨 시나리오 1. `useradd` 시 `-d /gitA/xxx` 로 지정했는데 `/home/xxx` 에 생성되거나 홈이 안 만들어짐

- **원인:** `-m` 없이 `-d` 만 지정하면 홈이 실제 생성되지 않을 수 있고, skel도 복사되지 않습니다.
    
- **해결:** 항상 **`-md <경로>`** 또는 **`-mk <skel> -d <경로>`** 를 **세트로** 사용.
    
    ```bash
    sudo userdel -r auser1                  # 잘못 생성된 계정/홈 정리
    sudo useradd -u 1200 -c "gitA_User1" -s /bin/tcsh -mk /etc/skel-a -d /gitA/gitA_User1 auser1
    ```
    

#### 🚨 시나리오 2. Primary Group 변경 후, 홈 디렉터리 내부의 **기존 파일** 소유 그룹이 이전 그룹으로 남음

- **원인:** `usermod -g` 는 이후 생성되는 파일에만 반영되고, **홈 디렉터리 내부에 이미 존재하던 파일의 소유 그룹은 자동 변경되지 않습니다.**
    
- **참고:** 이번 실습 로그에서는 `useradd` 직후 곧바로 `usermod -g` 를 수행해 `ls -l /gitHQ` 등의 상위 홈 디렉터리 그룹이 부서 그룹으로 정상 표시되었습니다. 다만 홈 **내부** dotfile 등의 소유 그룹은 별도 확인이 필요합니다.
    
- **해결:**
    
    ```bash
    sudo chgrp -R gitA /gitA/gitA_User1     # 기존 홈 내부 파일 소유그룹 일괄 변경
    sudo find /gitA/gitA_User1 -type d -exec chmod g+s {} \;   # (선택) SGID로 이후 파일도 그룹 상속
    ```
    

#### 🚨 시나리오 3. sudoers에 등록했는데도 `sudo` 실패

- **원인/해결:** `visudo -c` 로 문법 검증 → `/etc/sudoers` 의 `%wheel` 라인 주석 여부 확인 → 사용자 **재로그인** 으로 세션 그룹 갱신.

---

> 📌 **핵심 요약**
> - skel 템플릿(`/etc/skel`)에 공통 파일 배치 → `useradd -m` 시 홈에 자동 복사
> - 부서별 그룹 먼저 생성 → 계정 생성 시 `-g` (primary) + `-G` (secondary) 지정
> - 홈 디렉터리 권한: 개인 계정 `700`, 부서 공유 `2770`(SGID) 조합이 표준
> - 관련: 5-1. 👤 리눅스 사용자 계정 관리 (useradd & usermod & userdel) · 5-2. 👥 리눅스 그룹 관리 & UPG 모델(groupadd &usermod & gpasswd) · 5-3. 🛡️ Root 접속 통제 & Sudo 권한 위임 · 5-4. 🧩 사용자·그룹·권한 통합 정리
