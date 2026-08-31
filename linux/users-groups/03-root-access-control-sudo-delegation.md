# Root 접속 통제 & Sudo 권한 위임 (SSH Hardening / wheel / sudoers)

root 직접 접속을 차단하고, `wheel` 그룹 또는 `/etc/sudoers` 를 통해 **최소 권한 원칙(Least Privilege)** 으로 관리자 권한을 위임하는 실무 표준.

---

## 1. 개요 (Overview)

`PermitRootLogin no` 를 모든 운영 서버의 기본값으로 강제해야 하는 이유는 **감사 추적성(Audit Trail)** 확보와 **폭발 반경 최소화** 때문이다. root 직접 로그인은 "누가" 명령을 실행했는지 추적이 불가능하고, 비밀번호 하나 유출로 시스템 전체를 잃게 된다. `root` 는 이름이 아니라 **UID=0** 으로 정의되며, `/etc/passwd` 편집으로 다른 계정을 UID 0으로 바꾸면 그 계정도 root가 되므로, 감사 시 `awk -F: '$3==0 {print $1}' /etc/passwd` 로 UID=0 계정을 반드시 점검해야 한다. 개인 계정으로 로그인 후 `sudo` 를 쓰면 `/var/log/secure` 에 **"누가 어떤 명령을 실행했는지"** 남아 사고 추적이 가능하다.

`su` 와 `su -` 의 차이가 실무에서 중요한 이유는, `su` 는 계정만 바뀌고 **환경변수(PATH, HOME 등)** 는 기존 유저 그대로라서 `sbin` 경로 명령이 안 먹거나 스크립트가 오작동하기 때문이다. `su -` 는 **로그인 셸을 새로 열어** root로 직접 로그인한 것과 동일한 환경을 제공한다. 관리 작업(패키지 설치, 서비스 제어)은 반드시 `su -` 또는 `sudo -i` 를 사용해야 하며, 스크립트 자동화에서는 `sudo` 를 우선 사용해 감사 로그를 남기는 것이 표준이다.

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. SSH root 직접 로그인 차단

```properties
# /etc/ssh/sshd_config
# ------------------------------------------------------------
PermitRootLogin no          # root 직접 SSH 로그인 차단
MaxAuthTries 3              # 인증 실패 허용 3회 → brute force 방어
# PasswordAuthentication no # (선택) 키 기반 인증만 허용
# ------------------------------------------------------------
# 설정 반영 systemctl restart sshd
```
### Step 2. `su` 명령을 wheel 그룹으로 제한 (PAM)

```bash
# /etc/pam.d/su 에서 아래 줄의 주석 해제
auth required pam_wheel.so use_uid
```

- `use_uid` 옵션: 실제 UID(root로 전환 전 원래 사용자)를 기준으로 wheel 소속 여부 판단
- 설정 후 wheel 비소속 계정은 `su -` 실행 시 `Permission denied` 반환
- `/etc/sudoers` 와 별개로 동작하므로 sudo 권한 없이 su만 제한할 때 유용

```bash
# 적용 확인 (wheel 비소속 계정으로 테스트)
su - root   # → Authentication failure 확인
```

---

### Step 4. sudo 권한 위임 — 방식 ① wheel 그룹 (RHEL 표준)

```bash
# [1] 대상 계정을 wheel 그룹에 '보조 그룹'으로 추가
usermod -aG wheel subroot

# [2] /etc/sudoers 에 아래 라인이 활성화돼 있는지 확인 (RHEL 기본값)
# %wheel ALL=(ALL) ALL

# [3] 권한 회수
gpasswd -d subroot wheel
```

### Step 5. sudo 권한 위임 — 방식 ② sudoers 개별 등록

```bash
# 반드시 visudo 로 편집 (문법 오류 시 sudo 자체 잠금 사고 방지)
visudo
```

```properties
# /etc/sudoers
# 형식 : 사용자 호스트=(실행권한자) 허용명령
root     ALL=(ALL)   ALL
rootsub  ALL=(ALL)   ALL                # 풀 권한 위임

# [실무 권장] 최소 권한 원칙 - 특정 명령만 허용
# opsuser ALL=(root) NOPASSWD: /bin/systemctl restart nginx, /usr/bin/tail -f /var/log/*
```

### Step 6. 비밀번호 잠금/해제 (퇴사자·휴직자 대응)

```bash
passwd -S <user>          # 상태 조회 (PS/LK/NP)
passwd -l <user>          # 잠금 (계정 유지 + 로그인만 차단)  실무 표준
passwd -u <user>          # 잠금 해제
# passwd -d <user> # 비밀번호 삭제 → 무비번 로그인 위험, 실무 사용 금지
chage -l <user>           # 만료일/최근 변경일 확인
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
sshd -T | grep -Ei 'permitrootlogin|maxauthtries'   # 실효 정책 확인
sudo -l -U <user>                                    # 해당 유저가 쓸 수 있는 sudo 명령 조회
visudo -c                                            # sudoers 문법 검증
awk -F: '$3==0 {print $1}' /etc/passwd               # UID=0 계정 감사
journalctl _COMM=sudo -e                             # sudo 실행 로그 (누가/무엇을)
```

---

### 3-2. 트러블슈팅 시나리오

#### 시나리오 1. wheel 그룹에 넣었는데도 `"<user>은(는) sudoers 파일에 없습니다"`

- **원인 후보:**
    1. `/etc/sudoers` 의 `%wheel ALL=(ALL) ALL` 라인이 주석 처리돼 있음.
    2. 그룹 추가 후 **재로그인을 안 함** → 현재 세션에는 그룹이 반영되지 않음.
    3. `visudo` 대신 `vi` 로 편집해 문법 오류 발생 → sudoers 전체 무효화.
- **해결 절차:**
    1. `grep -E '^\s*%wheel' /etc/sudoers` 로 주석 여부 확인 → 주석이면 `visudo` 로 해제.
    2. `id <user>` 로 wheel 소속 확인 → 없으면 `usermod -aG wheel <user>` 후 **재로그인**.
    3. `visudo -c` 로 문법 검증. 오류 시 백업본으로 복구.
    4. 여전히 실패 시 `/var/log/secure` 로 거부 사유 확인.

#### 시나리오 2. `PermitRootLogin no` 를 반영했더니 자동화 스크립트가 전부 실패

- **원인:** CI/배포 스크립트가 root SSH 로그인을 전제로 작성돼 있음.
- **해결 절차:**
    1. 배포 계정(예: `deploy`)을 신규 생성 → `usermod -aG wheel deploy`.
    2. SSH **키 기반 인증** + `sudo NOPASSWD` 로 제한적 명령만 허용.
        
        ```properties
        deploy ALL=(root) NOPASSWD: /bin/systemctl restart myapp, /usr/bin/rsync
        ```
        
    3. 스크립트 내 `ssh root@host` → `ssh deploy@host "sudo ..."` 로 교체.
    4. 감사 로그(`journalctl _COMM=sudo`)에 배포 이력이 남는지 확인.

>  **핵심 요약**
> - 직접 root 로그인 차단: `/etc/ssh/sshd_config` → `PermitRootLogin no` + `sshd` 재시작
> - `/etc/pam.d/su` → `auth required pam_wheel.so` 로 `su` 도 wheel 그룹만 허용
> - sudo 권한: `usermod -aG wheel user` 또는 `visudo` 에서 사용자·명령 단위 제한 설정
> - sudo 로그 확인: `journalctl _COMM=sudo`, `grep sudo /var/log/secure`
> - 관련: 5-1.  리눅스 사용자 계정 관리 (useradd & usermod & userdel) · 5-2.  리눅스 그룹 관리 & UPG 모델(groupadd &usermod & gpasswd) · 5-4.  사용자·그룹·권한 통합 정리 · 5-5.  사용자·그룹·권한 트러블슈팅 치트시트 · 5-6.  사용자·그룹·권한 명령어 퀵 레퍼런스
