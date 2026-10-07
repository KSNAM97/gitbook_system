# Umask — 기본 권한 마스크 (User Mask)

> **Tag:** #Linux #Umask #Permission #DefaultPermission #Security  
> **핵심 요약:** `umask`는 새 파일과 디렉터리를 생성할 때 프로그램이 요청한 권한에서 특정 권한 비트를 제거한다. 일반적인 요청 모드는 파일 `0666`, 디렉터리 `0777`이며 최종 권한은 `요청 권한 & ~umask`로 결정된다.

---

## 1. 개요 (Overview)

**umask란?**

`umask`는 현재 프로세스와 그 자식 프로세스가 새 파일이나 디렉터리를 생성할 때 제거할 권한을 정의한다.

```text
u    = user
mask = 제거하거나 가리는 권한 비트
```

현재 값 확인:

```bash
umask
```

심볼릭 표시:

```bash
umask -S
```

**기본 생성 요청 권한은?**

일반적인 프로그램은 다음 권한으로 생성을 요청한다.

```text
일반 파일: 0666 = rw-rw-rw-
디렉터리:   0777 = rwxrwxrwx
```

새 일반 파일은 실행 파일임을 전제로 하지 않으므로 기본 요청 모드에 `x`가 없다.

**계산은 단순 뺄셈인가?**

엄밀히는 비트 제거 연산이다.

```text
최종 권한 = 요청 권한 & ~umask
```

대표 결과:

| umask | 파일 | 디렉터리 |
|---:|---:|---:|
| `0002` | `0664` | `0775` |
| `0022` | `0644` | `0755` |
| `0027` | `0640` | `0750` |
| `0077` | `0600` | `0700` |
| `0011` | `0666` | `0766` |
| `0033` | `0644` | `0744` |

`umask 0011`의 경우 파일 요청 권한 `0666`에는 실행 비트가 원래 없으므로 최종 파일 권한은 `0666`이다.

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 현재 셸에서 변경

```bash
umask 0027
```

확인:

```bash
umask
umask -S
```

### Step 2. 생성 결과 검증

```bash
mkdir umask-test
cd umask-test

mkdir testD
touch testF

stat -c '%a %A %n' testD testF
```

`umask 0022`라면 일반적으로:

```text
755 drwxr-xr-x testD
644 -rw-r--r-- testF
```

### Step 3. `umask 0002`

```bash
umask 0002
mkdir mode0002D
touch mode0002F
```

예상:

```text
775 mode0002D
664 mode0002F
```

### Step 4. `umask 0011`

```bash
umask 0011
mkdir mode0011D
touch mode0011F
```

예상:

```text
766 mode0011D
666 mode0011F
```

### Step 5. `umask 0033`

```bash
umask 0033
mkdir mode0033D
touch mode0033F
```

예상:

```text
744 mode0033D
644 mode0033F
```

### Step 6. 영구 설정 위치

사용자별 후보:

```text
~/.bashrc
~/.bash_profile
~/.profile
```

시스템 전체 후보:

```text
/etc/profile
/etc/bashrc
/etc/profile.d/*.sh
/etc/login.defs
PAM pam_umask 설정
```

예시:

```bash
umask 0027
```

systemd 서비스:

```ini
[Service]
UMask=0027
```

변경 후:

```bash
systemctl daemon-reload
systemctl restart <서비스>
```

---

## 3. 검증 및 트러블슈팅

### 3-1. root와 일반 사용자의 값 비교

```bash
sudo -i
umask
exit

su - <사용자>
umask
```

root와 일반 사용자의 기본값이 반드시 같은 것은 아니다. 배포판, 로그인 방식, PAM, 셸 초기화 파일에 따라 달라질 수 있다.

### 3-2. umask를 바꿨는데 기존 파일이 그대로다

`umask`는 앞으로 새로 생성되는 객체에만 적용한다.

기존 객체는 `chmod`로 변경한다.

```bash
chmod 640 existing-file
chmod 750 existing-directory
```

### 3-3. 파일 권한이 계산 결과와 다르다

프로그램이 반드시 `0666` 또는 `0777`을 요청하는 것은 아니다. 애플리케이션이 더 제한된 권한으로 생성했을 수 있다.

추가 원인:

- 기본 ACL
- 애플리케이션 자체 권한 설정
- `install -m`
- 보안 프로그램
- 네트워크 파일시스템 정책

확인:

```bash
getfacl <경로>
strace -e openat,creat,mkdir <명령>
```

### 3-4. 홈 디렉터리 권한이 예상과 다르다

```bash
grep -E '^(UMASK|HOME_MODE|USERGROUPS_ENAB)' /etc/login.defs
useradd -D
```

`HOME_MODE`가 설정되어 있으면 새 홈 디렉터리 권한에 우선 사용될 수 있다.

>  **핵심 요약**
> - 파일 요청 모드: 일반적으로 `0666`
> - 디렉터리 요청 모드: 일반적으로 `0777`
> - 최종 권한: 요청 모드에서 umask 비트 제거
> - 기존 파일에는 적용되지 않음
> - 영구 설정은 로그인 방식과 서비스 환경을 구분
> - 관련: 6-1.  허가권 (Permission) — chmod & rwx·UGO 모델 · 6-6.  특수권한 Set-GID — 소유 그룹 자동 상속 (2XXX)
