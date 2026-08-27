# 🔐 허가권 (Permission) — chmod & rwx·UGO 모델

> **Tag:** #Linux #Permission #chmod #rwx #UGO #Security  
> **핵심 요약:** Linux 허가권은 파일과 디렉터리에 대해 소유자(Owner), 소유 그룹(Group), 그 외 사용자(Other)가 수행할 수 있는 작업을 `r`, `w`, `x`로 구분한다. 같은 권한 문자라도 파일과 디렉터리에서 의미가 다르며, 파일 삭제·이름 변경은 파일 자체 권한이 아니라 상위 디렉터리의 `w+x` 권한과 Sticky-bit 정책을 중심으로 판정한다.

---

## 1. 📖 개요 (Overview)

Linux는 여러 사용자가 동시에 접속해 같은 파일시스템을 사용하는 멀티유저 운영체제이다.

모든 사용자에게 모든 경로의 접근을 허용하면 다음과 같은 문제가 발생한다.

- 다른 사용자의 개인 파일 열람
- 설정 파일의 무단 변경
- 공유 디렉터리의 파일 삭제
- 실행 파일 변조
- 시스템 파일 손상
- 사용자 간 데이터 유출

Linux는 이러한 문제를 방지하기 위해 다음 두 가지 정보를 함께 사용한다.

```text
Ownership(소유권)
→ 누구의 파일 또는 디렉터리인가?

Permission(허가권)
→ 그 사용자가 어떤 작업을 할 수 있는가?
```

예시:

```text
-rw-r----- 1 user1 teamA 1024 Jul 10 10:00 report.txt
```

구성:

```text
-           일반 파일
rw-         Owner 권한
r--         Group 권한
---         Other 권한
user1       소유자
teamA       소유 그룹
report.txt  파일명
```

UGO 모델은 다음과 같다.

| 구분 | 기호 | 의미 |
|---|---|---|
| User/Owner | `u` | 파일 또는 디렉터리의 소유자 |
| Group | `g` | 해당 객체의 소유 그룹에 속한 사용자 |
| Other | `o` | Owner도 아니고 적용 Group에도 속하지 않은 사용자 |
| All | `a` | Owner·Group·Other 전체 |

다음 파일을 예로 든다.

```text
-rw-r----- 1 user1 teamA report.txt
```

권한 적용:

```text
user1            → Owner 권한 rw-
teamA 구성원     → Group 권한 r--
그 외 사용자     → Other 권한 ---
```

Owner·Group·Other 권한은 합산하지 않는다.

```text
현재 프로세스 UID가 Owner UID와 같은가?
├─ 예
│  └─ Owner 권한 적용
└─ 아니요
   └─ 프로세스 그룹 목록에 파일 GID가 있는가?
      ├─ 예
      │  └─ Group 권한 적용
      └─ 아니요
         └─ Other 권한 적용
```

현재 사용자:

```bash
id
```

특정 사용자:

```bash
id <사용자>
```

숫자 UID·GID:

```bash
ls -ln <경로>
stat <경로>
```

`ls -l` 출력의 첫 번째 문자는 파일 종류를 나타낸다.

| 문자 | 의미 |
|---|---|
| `-` | 일반 파일 |
| `d` | 디렉터리 |
| `l` | 심볼릭 링크 |
| `b` | 블록 장치 |
| `c` | 문자 장치 |
| `s` | 소켓 |
| `p` | FIFO 또는 Named Pipe |

```text
-rwxr-x---
││  │  │
││  │  └─ Other 권한
││  └──── Group 권한
│└─────── Owner 권한
└──────── 파일 종류
```

**파일과 디렉터리의 `r`, `w`, `x`는 어떻게 다른가?**

#### 일반 파일

| 권한 | 의미 | 대표 작업 |
|---|---|---|
| `r` | 파일 내용 읽기 | `cat`, `less`, 원본 복사 |
| `w` | 파일 내용 변경 | 편집, 덮어쓰기, 내용 비우기 |
| `x` | 프로그램 실행 | 바이너리·스크립트 실행 |

#### 디렉터리

| 권한 | 의미 | 대표 작업 |
|---|---|---|
| `r` | 엔트리 이름 목록 조회 | `ls` |
| `w` | 엔트리 생성·삭제·이름 변경 | `touch`, `mkdir`, `rm`, `mv` |
| `x` | 디렉터리 탐색·통과 | `cd`, 내부 객체 접근 |

> 디렉터리에서 파일과 하위 디렉터리를 생성·삭제하려면 일반적으로 `w+x`가 함께 필요하다.

**파일의 `r`, `w`, `x`는 실제로 어떻게 쓰이는가?**

파일 읽기:

```bash
cat file
less file
cp file /tmp/
```

파일 수정:

```bash
echo "new data" >> file
```

파일 비우기:

```bash
: > file
```

실행:

```bash
chmod u+x script.sh
./script.sh
```

파일에 `x`가 있어도 다음 조건에 따라 실행이 실패할 수 있다.

- 실행 가능한 바이너리 형식이 아님
- 스크립트의 Shebang 또는 인터프리터 오류
- 마운트 옵션이 `noexec`
- SELinux 정책
- 상위 디렉터리 `x` 권한 없음

**디렉터리의 `r`, `w`, `x`는 실제로 어떻게 쓰이는가?**

목록 조회:

```bash
ls /directory
```

엔트리 생성:

```bash
touch /directory/file
mkdir /directory/subdir
```

엔트리 삭제·이름 변경:

```bash
rm /directory/file
mv /directory/file /directory/new-name
```

디렉터리 탐색:

```bash
cd /directory
cat /directory/known-file
```

`r` 없이 `x`만 있으면 파일명을 알고 있는 객체에 접근할 수 있지만 전체 목록은 조회하지 못할 수 있다.

`r`만 있고 `x`가 없으면 파일명은 보여도 상세 정보가 `?????????`로 표시될 수 있다.

**파일 삭제에는 파일 자체 `w`가 필요한가?**

반드시 그렇지는 않다.

파일 삭제는 파일 내용을 수정하는 작업이 아니라 상위 디렉터리에서 파일명과 inode의 연결을 제거하는 작업이다.

예시:

```text
상위 디렉터리: drwxrwxrwx
대상 파일:     -r--r--r-- root root file
```

상위 디렉터리에 `w+x`가 있고 Sticky-bit가 없다면 읽기 전용 파일도 삭제될 수 있다.

반대 예시:

```text
상위 디렉터리: drwxr-xr-x
대상 파일:     -rwxrwxrwx user user file
```

파일 자체가 `777`이어도 상위 디렉터리에 `w`가 없으면 삭제하거나 이름을 변경할 수 없다.

| 작업 | 주요 판정 대상 |
|---|---|
| 파일 내용 읽기 | 파일 `r` |
| 파일 내용 수정 | 파일 `w` |
| 파일 실행 | 파일 `x` |
| 파일 생성 | 상위 디렉터리 `w+x` |
| 파일 삭제 | 상위 디렉터리 `w+x`와 Sticky-bit |
| 파일 이름 변경 | 상위 디렉터리 `w+x`와 Sticky-bit |
| 목록 조회 | 디렉터리 `r` |
| 내부 경로 접근 | 디렉터리 `x` |

**root는 모든 제한을 무조건 우회하는가?**

root 또는 관련 capability를 가진 프로세스는 일반적인 DAC 권한 검사를 대부분 우회할 수 있다.

그러나 다음 제한까지 항상 무조건 우회하는 것은 아니다.

- 읽기 전용 파일시스템
- immutable 속성
- SELinux
- NFS `root_squash`
- 컨테이너 사용자 네임스페이스
- 손상된 파일시스템
- 스토리지 I/O 오류
- capability 제한

확인:

```bash
findmnt -T <경로>
lsattr <경로>
ls -lZ <경로>
getfacl <경로>
```

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 권한 확인

```bash
ls -l /path/to/file
ls -ld /path/to/directory
stat -c '%A %a %U:%G %n' /path/to/object
```

차이:

```bash
ls -l /net     # /net 내부 목록
ls -ld /net    # /net 디렉터리 자체
```

---

### 2-2. 숫자 방식

```text
r = 4
w = 2
x = 1
```

| 권한 | 숫자 |
|---|---:|
| `---` | 0 |
| `--x` | 1 |
| `-w-` | 2 |
| `-wx` | 3 |
| `r--` | 4 |
| `r-x` | 5 |
| `rw-` | 6 |
| `rwx` | 7 |

대표 예:

```bash
chmod 600 private.key
chmod 640 config.conf
chmod 644 document.txt
chmod 700 private-directory
chmod 750 admin-directory
chmod 755 public-directory
chmod 770 team-directory
```

---

### 2-3. 심볼릭 방식

대상:

```text
u = Owner
g = Group
o = Other
a = 전체
```

연산자:

```text
+ = 추가
- = 제거
= = 재설정
```

예:

```bash
chmod u+x script.sh
chmod g+w team-file
chmod o-r private-file
chmod o-rwx private-directory
chmod a+r document.txt
chmod u=rw,g=r,o= config.conf
```

---

### 2-4. 재귀 권한 변경

모든 객체에 같은 권한:

```bash
chmod -R 750 /project
```

파일과 디렉터리를 구분:

```bash
find /project -type d -exec chmod 750 {} +
find /project -type f -exec chmod 640 {} +
```

대문자 `X`:

```bash
chmod -R g+rX /project
```

`X`는 디렉터리 또는 기존 실행 파일에만 실행 권한을 적용한다.

---

### 2-5. 기본 생성 권한과 umask

일반적인 생성 요청 모드:

```text
파일:       0666
디렉터리:   0777
```

`umask 0022`:

```text
파일:       0644
디렉터리:   0755
```

`umask 0002`:

```text
파일:       0664
디렉터리:   0775
```

확인:

```bash
umask
umask -S
```

---

### 2-6. 실습 — `777` 공유 디렉터리

root:

```bash
mkdir -p /net
chmod 777 /net
ls -ld /net
```

guest:

```bash
cd /net
mkdir guestD
touch guestF
ls -l /net
```

새 객체의 권한은 `/net`의 `777`을 그대로 상속하지 않는다. 생성 프로그램의 요청 모드와 `umask`가 적용된다.

---

### 2-7. 실습 — 읽기 전용 파일 삭제

root:

```bash
cp /etc/passwd /net/passwd
ls -l /net/passwd
```

guest:

```bash
cat /net/passwd
mv /net/passwd /net/password
rm /net/password
```

파일 자체는 Other `r--`이지만 `/net`이 `777`이고 Sticky-bit가 없으므로 이름 변경과 삭제가 가능할 수 있다.

타인 파일 삭제를 제한:

```bash
chmod 1777 /net
```

---

### 2-8. 실습 — Other가 `-wx`

```bash
chmod 773 /net
```

guest:

```bash
cd /net
touch /net/GuestF
mkdir /net/GuestD
ls -l /net
```

예상:

```text
cd·생성 가능
목록 조회 실패
```

---

### 2-9. 실습 — Other가 `r-x`

```bash
chmod 775 /net
```

guest:

```bash
cd /net
ls -l /net
touch /net/GuestF
```

예상:

```text
접근·목록 조회 가능
생성·삭제 불가
```

---

### 2-10. 실습 — Other가 `rw-`

```bash
mkdir -p /permission-test
touch /permission-test/file1
mkdir /permission-test/dir1
chmod 776 /permission-test
```

guest:

```bash
ls -l /permission-test
cd /permission-test
touch /permission-test/new-file
```

`x`가 없으므로 상세 정보 조회와 내부 접근·생성이 실패할 수 있다.

---

### 2-11. 목적별 권장 권한

개인 전용:

```bash
chmod 700 /private
```

공개 조회:

```bash
chmod 755 /public-read
```

팀 공유:

```bash
groupadd project
chown root:project /project
chmod 2770 /project
```

팀 공유 + 삭제 보호:

```bash
chmod 3770 /project
```

공개 임시 공간:

```bash
chown root:root /public-tmp
chmod 1777 /public-tmp
```

| 목적 | 권한 |
|---|---:|
| 개인 전용 | `700` |
| 공개 조회 | `755` |
| 팀 협업 | `2770` |
| 팀 협업·삭제 보호 | `3770` |
| 공개 임시 공간 | `1777` |

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 진단

```bash
id
namei -l /path/to/object
stat /path/to/object
getfacl /path/to/object
ls -lZ /path/to/object
lsattr /path/to/object
findmnt -T /path/to/object
```

---

### 3-2. `777`인데 접근할 수 없다

```bash
namei -l /path/to/file
getfacl /path/to/file
ls -lZ /path/to/file
findmnt -T /path/to/file
```

상위 디렉터리 `x`, ACL, SELinux, 마운트 옵션을 확인한다.

---

### 3-3. 파일에 `w`가 없는데 삭제된다

```bash
ls -l /path/to/file
ls -ld /path/to
```

삭제는 상위 디렉터리 권한으로 판정한다.

Sticky-bit:

```bash
chmod +t /path/to
```

---

### 3-4. 디렉터리에 `w`가 있는데 생성되지 않는다

```bash
ls -ld /path/to/directory
```

`x`가 없다면 실질적인 생성·접근이 제한된다.

```bash
chmod g+x /path/to/directory
```

---

### 3-5. 그룹 변경 후 권한이 적용되지 않는다

```bash
id <사용자>
id
```

현재 세션에 그룹이 없다면 재로그인한다.

```bash
newgrp <그룹>
```

---

### 3-6. root도 수정하지 못한다

```bash
findmnt -T /path/to/file
lsattr /path/to/file
ls -lZ /path/to/file
getfacl /path/to/file
```

immutable 속성을 해제해야 하는 경우:

```bash
chattr -i /path/to/file
```

보호 이유를 확인한 후 적용한다.

---

### 3-7. `chmod -R 777`을 잘못 실행했다

`chmod`는 이전 권한 정보를 저장하지 않는다. 따라서 명령 실행 전의 정확한 권한을 자동 복원할 수 없다.

현재 상태:

```bash
find /project -printf '%M %u:%g %p\n'
```

복원 우선순위:

1. ACL·권한 백업
2. 형상관리 또는 백업
3. 패키지 메타데이터
4. 서비스별 표준 권한 정책
5. 파일과 디렉터리를 구분한 수동 복구

ACL 백업이 있다면:

```bash
setfacl --restore=/root/project.acl
```

패키지 소유 파일:

```bash
rpm -qf <파일>
rpm -V <패키지>
```

그룹만 `project`로 통일해야 하는 정책이라면:

```bash
chgrp -R project /project
```

표준 정책이 다음과 같이 명확히 정의된 경우에만 적용한다.

```bash
find /project -type d -exec chmod 2770 {} +
find /project -type f -exec chmod 0660 {} +
```

> `chown -R root:project /project`는 모든 파일의 사용자 소유자를 root로 변경하므로 일반적인 복구 명령으로 사용하지 않는다.

---

### 3-8. `chmod -R 644` 후 디렉터리에 들어갈 수 없다

디렉터리 `x`가 제거된 상태다.

정책이 디렉터리 `750`, 일반 파일 `640`으로 명확하다면:

```bash
find /project -type d -exec chmod 750 {} +
find /project -type f -exec chmod 640 {} +
```

> 위 명령은 기존 실행 파일의 `x`를 복구하지 못한다. 실행 파일이 있었다면 백업, 패키지 검증 또는 형상관리 정보를 기준으로 별도 복구한다.

---

### 3-9. 안전한 권한 설정 절차

현재 상태 기록:

```bash
stat <경로>
getfacl <경로>
ls -lZ <경로>
```

ACL 백업:

```bash
getfacl -R /project > /root/project.acl
```

사용자와 상위 경로 확인:

```bash
id
namei -l <전체경로>
```

필요한 최소 권한만 적용:

```bash
chmod g+rX <경로>
chmod g+w <공유디렉터리>
```

대상 사용자로 검증:

```bash
sudo -u <사용자> test -r <파일> \
  && echo readable

sudo -u <사용자> test -w <파일> \
  && echo writable

sudo -u <사용자> test -x <경로> \
  && echo executable-or-searchable
```

실제 생성·삭제:

```bash
sudo -u <사용자> touch <디렉터리>/test-file
sudo -u <사용자> rm <디렉터리>/test-file
```

---

> 📌 **핵심 요약**
> - 파일 `r`: 내용 읽기
> - 파일 `w`: 내용 수정
> - 파일 `x`: 실행
> - 디렉터리 `r`: 이름 목록 조회
> - 디렉터리 `w`: 엔트리 생성·삭제·이름 변경
> - 디렉터리 `x`: 경로 탐색
> - 삭제·이름 변경은 상위 디렉터리의 `w+x` 확인
> - `chmod`는 이전 권한을 저장하지 않으므로 재귀 권한 사고 전 백업 필요
> - 관련: 🔧 허가권 상세 (chmod & 8진수 · 심볼릭 표기) · 👤  소유권 (Ownership) — chown & UID·GID 소유 모델 · 🎭  Umask — 기본 권한 마스크 (User Mask) · 👥 특수권한 Set-GID — 소유 그룹 자동 상속 (2XXX) · 📌 특수권한 Sticky-bit — 공유 디렉터리 삭제 방지 (1XXX) · 🧩 허가권·소유권 통합 정리 · 🚑 허가권·소유권 트러블슈팅 치트시트 · ⚡ 허가권·소유권 명령어 퀵 레퍼런스
