# 파일·디렉터리 검색 (find)

> **Tag:** #Linux #find #Search #Audit #Housekeeping  
> **핵심 요약:** `find`는 지정한 경로를 직접 탐색하여 이름, 종류, 수정 시간, 크기, 권한, 소유자 등의 조건으로 파일과 디렉터리를 검색한다. `-exec`와 `-delete`를 사용하면 검색 결과에 작업을 수행할 수 있으므로 반드시 `-print`로 먼저 검증한다.

---

## 1. 개요 (Overview)

`find`와 `locate`의 차이는 다음과 같다.

| 구분 | `find` | `locate`/`plocate` |
|---|---|---|
| 검색 방식 | 파일시스템 직접 탐색 | 미리 생성된 데이터베이스 조회 |
| 결과 시점 | 현재 파일시스템 상태 | 마지막 DB 갱신 시점 |
| 검색 속도 | 범위가 크면 느릴 수 있음 | 일반적으로 빠름 |
| 상세 조건 | 이름·시간·크기·권한 등 | 주로 경로 이름 |
| 대표 용도 | 정밀 검색·감사·자동화 | 파일명 빠른 검색 |

```bash
find /etc -name 'passwd'
locate passwd
```

DB를 수동 갱신할 수 있는 환경이라면 다음 명령을 사용한다.

```bash
updatedb
```

> **참고:** `locate`의 DB 갱신 주기는 배포판과 타이머 설정에 따라 다르다. 항상 하루에 한 번 갱신된다고 단정할 수 없다. Rocky Linux 환경에서는 `plocate` 또는 관련 패키지 설치가 필요할 수 있다.

`find`의 기본 문법은 다음과 같다.

```text
find [검색 시작 경로] [조건식] [액션]
```

예시:

```bash
find /etc -type f -name '*.conf' -print
```

구성:

```text
/etc          검색 시작 경로
-type f       일반 파일
-name         이름 조건
'*.conf'      .conf로 끝나는 이름
-print        결과 출력
```

여러 조건을 작성하면 별도의 연산자가 없는 한 기본적으로 AND 조건으로 연결된다.

```bash
find /etc -type f -name '*.conf'
```

다음 조건을 모두 만족해야 한다.

```text
/etc 아래에 존재
AND 일반 파일
AND 이름이 .conf로 끝남
```

연산자 우선순위는 다음과 같다.

```text
괄호
NOT
AND
OR
```

OR를 사용할 때는 다른 조건과의 결합 범위를 괄호로 명확하게 작성한다.

```bash
find /etc -type f \( -name '*.conf' -o -name '*.cfg' \)
```

삭제 작업에서 지켜야 할 원칙은 다음과 같은 순서를 따르는 것이다: 1) 시작 경로를 절대경로로 지정, 2) 파일 종류를 `-type f`처럼 제한, 3) 이름·시간 조건을 명확하게 작성, 4) `-print`로 결과 검증, 5) 개수와 용량 확인, 6) 같은 조건에서 마지막 액션만 `-delete`로 변경.

```bash
find /var/log -type f -name '*.log' -mtime +30 -print
```

확인 후:

```bash
find /var/log -type f -name '*.log' -mtime +30 -delete
```

> **참고:** `-mindepth 1`은 시작 경로 자체를 제외하는 데 도움이 되지만, 잘못된 시작 경로 또는 과도하게 넓은 검색 범위까지 자동으로 방어하지는 않는다.

---

## 2. 표준 사용 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 기본 검색

조건 없이 실행하면 시작 경로와 그 아래의 모든 항목을 출력한다.

```bash
find /etc
```

현재 디렉터리부터 검색:

```bash
find .
```

검색 깊이 제한:

```bash
find /etc -maxdepth 1
find /etc -mindepth 1 -maxdepth 2
```

- `-maxdepth 1`: 시작 경로와 바로 아래 항목까지만
- `-mindepth 1`: 시작 경로 자체는 액션 대상에서 제외

### Step 2. `-name` — 이름 검색

전체 파일시스템에서 `passwd` 검색:

```bash
find / -name 'passwd' 2>/dev/null
```

`/etc` 아래에서 검색:

```bash
find /etc -name 'passwd'
```

`_config`로 끝나는 이름:

```bash
find /etc -name '*_config'
```

`b`로 시작하는 이름:

```bash
find /etc -name 'b*'
```

대소문자를 무시하고 `.conf` 검색:

```bash
find / -iname '*.conf' 2>/dev/null
```

정확한 파일명 검색:

```bash
find /etc -name 'ens160.nmconnection'
```

접두사 검색:

```bash
find /etc -name 'ens160*'
```

> **참고:** `-name` 뒤의 와일드카드는 셸이 먼저 확장하지 않도록 따옴표로 감싼다.

### Step 3. `-type` — 파일 종류

```bash
find /etc -type f                # 일반 파일
find /etc -type d                # 디렉터리
find /etc -type l                # 심볼릭 링크
find /dev -type b                # 블록 장치
find /dev -type c                # 문자 장치
find /run -type s                # 소켓
find /run -type p                # 이름 있는 파이프
```

`b`로 시작하는 일반 파일:

```bash
find /etc -type f -name 'b*'
```

`b`로 시작하는 디렉터리:

```bash
find /etc -type d -name 'b*'
```

깨진 심볼릭 링크:

```bash
find / -xtype l 2>/dev/null
```

### Step 4. `-newer` — 기준 파일보다 최신인 항목

```bash
find /etc -newer /etc/aliases
```

`-newer`는 기본적으로 기준 파일과 검색 대상의 **수정 시간(mtime)**을 비교한다. 생성 시간을 비교하는 옵션이 아니다.

일반 파일만 검색:

```bash
find /etc -type f -newer /etc/aliases
```

디렉터리만 검색:

```bash
find /etc -type d -newer /etc/aliases
```

### Step 5. `-newermt` — 날짜와 수정 시간 비교

2026년 7월 5일 00:00:00보다 수정 시간이 이후인 일반 파일:

```bash
find /backup2 -type f -newermt '2026-07-05'
```

특정 기간 검색:

```bash
find /backup2 -type f \
  -newermt '2026-07-01' \
  ! -newermt '2026-07-05'
```

위 조건은 대략 다음 범위를 의미한다.

```text
2026-07-01 00:00:00보다 이후
AND
2026-07-05 00:00:00보다 최신은 아님
```

종료 시각을 명확하게 지정하려면:

```bash
find /backup2 -type f \
  -newermt '2026-07-01 00:00:00' \
  ! -newermt '2026-07-06 00:00:00'
```

### Step 6. 상대 시간 검색

```bash
find /var/log -type f -mtime -1
find /var/log -type f -mtime +7
find /tmp -type f -mmin -30
find /home -type f -atime +30
```

의미:

| 조건 | 의미 |
|---|---|
| `-mtime 0` | 완료된 24시간 단위가 0일 |
| `-mtime -1` | 완료된 24시간 단위가 1보다 작음 |
| `-mtime +7` | 완료된 24시간 단위가 7보다 큼 |
| `-mmin -30` | 완료된 분 단위가 30보다 작음 |
| `-atime +30` | 접근 시간 기준 완료된 일수가 30보다 큼 |

> **참고:** `-mtime`은 달력 날짜가 아니라 현재 시각을 기준으로 24시간 단위를 계산하고 소수 부분을 버린다. 달력 날짜 기준 검색은 `-newermt`를 사용하는 것이 명확하다.

### Step 7. 크기 검색

```bash
find /var/log -type f -size +100M
find /var/log -type f -size +1G
find /var/log -type f -size -10k
```

단위:

```text
c  Byte
k  KiB
M  MiB
G  GiB
```

빈 일반 파일 검색:

```bash
find /home -type f -empty
```

빈 디렉터리 검색:

```bash
find /home -type d -empty
```

> **참고:** `-size 0`도 크기가 0인 파일을 찾는 데 사용할 수 있지만, 빈 파일과 빈 디렉터리를 의도적으로 구분하려면 `-empty`와 `-type` 조합이 명확하다.

### Step 8. 권한 검색

권한이 정확히 `4000`인 파일:

```bash
find / -type f -perm 4000 2>/dev/null
```

SUID 비트가 설정된 모든 일반 파일:

```bash
find / -type f -perm -4000 2>/dev/null
```

SGID 비트가 설정된 모든 일반 파일:

```bash
find / -type f -perm -2000 2>/dev/null
```

SUID 또는 SGID가 하나라도 설정된 파일:

```bash
find / -type f -perm /6000 2>/dev/null
```

누구나 쓰기 가능한 일반 파일:

```bash
find / -type f -perm -0002 2>/dev/null
```

누구나 쓰기 가능한 디렉터리:

```bash
find / -type d -perm -0002 2>/dev/null
```

### Step 9. 소유자와 그룹 검색

특정 사용자 소유:

```bash
find / -user nobody 2>/dev/null
```

특정 그룹 소유:

```bash
find / -group users 2>/dev/null
```

현재 존재하지 않는 UID 소유:

```bash
find /home -nouser 2>/dev/null
```

현재 존재하지 않는 GID 소유:

```bash
find /home -nogroup 2>/dev/null
```

소유자 또는 그룹이 없는 항목:

```bash
find /home \( -nouser -o -nogroup \) -print 2>/dev/null
```

> **참고:** 괄호 없이 `-nouser -o -nogroup`을 사용하면 연산자 우선순위 때문에 뒤에 붙인 조건이나 액션이 예상과 다르게 적용될 수 있다.

### Step 10. 조건 조합

AND:

```bash
find /etc -type f -name '*.conf'
```

OR:

```bash
find /etc \( -name '*.conf' -o -name '*.cfg' \)
```

일반 파일이면서 `.conf` 또는 `.cfg`:

```bash
find /etc -type f \( -name '*.conf' -o -name '*.cfg' \)
```

NOT:

```bash
find /var/log -type f ! -name '*.gz'
```

Root 소유가 아닌 일반 파일:

```bash
find /home -type f ! -user root
```

### Step 11. 출력 액션

기본 경로 출력:

```bash
find /etc -type f -name '*.conf' -print
```

상세 정보:

```bash
find /etc -type f -name '*.conf' -ls
```

NULL 문자로 구분하여 출력:

```bash
find /etc -type f -name '*.conf' -print0
```

사용자 지정 출력:

```bash
find /var/log -type f -printf '%s\t%TY-%Tm-%Td %TH:%TM\t%p\n'
```

### Step 12. `-exec`

각 항목마다 한 번씩 실행:

```bash
find /var/log -type f -name '*.log' -mtime +30 \
  -exec ls -l -- {} \;
```

여러 항목을 한 명령에 묶어 실행:

```bash
find /var/log -type f -name '*.log' -mtime +30 \
  -exec ls -l -- {} +
```

오래된 로그 압축 예시:

```bash
find /var/log/myapp -type f -name '*.log' -mtime +30 \
  -exec gzip -- {} +
```

> **주의:** 현재 서비스가 사용 중인 로그를 직접 압축하면 장애가 발생할 수 있다. 운영 로그는 가능한 한 logrotate 정책으로 관리한다.

### Step 13. `-delete`

삭제 전 검증:

```bash
find /tmp -type f -mtime +7 -print
```

확인 후:

```bash
find /tmp -type f -mtime +7 -delete
```

주의사항:

- `-delete`는 즉시 삭제한다.
- `-delete`는 깊이 우선 순회와 관련된 동작을 사용하므로 `-prune`과 함께 사용하기 어렵다.
- 디렉터리까지 삭제하려면 비어 있어야 한다.
- 일반적으로 `-type f`처럼 삭제 대상을 제한한다.
- 넓은 시작 경로에서는 사용하지 않는다.

### Step 14. `xargs` 연동

공백, 줄바꿈 등 특수문자가 있는 경로를 안전하게 처리한다.

```bash
find /var/log -type f -name '*.log' -mtime +30 -print0 |
xargs -0 -r ls -l --
```

삭제 예시:

```bash
find /tmp/myapp -type f -mtime +7 -print0 |
xargs -0 -r rm -f --
```

- `-print0`: NULL 문자로 경로 구분
- `xargs -0`: NULL 문자 입력 처리
- `xargs -r`: 입력이 없으면 명령을 실행하지 않음(GNU `xargs`)

가능하면 단순 작업에는 `-exec ... {} +`를 우선 사용할 수 있다.

### Step 15. 특정 디렉터리 제외

`/proc`, `/sys` 제외:

```bash
find / \
  \( -path /proc -o -path /sys \) -prune \
  -o -type f -name '*.conf' -print
```

여러 가상 파일시스템 제외:

```bash
find / \
  \( -path /proc -o -path /sys -o -path /dev -o -path /run \) \
  -prune \
  -o -type f -name '*.conf' -print
```

현재 파일시스템 경계를 넘지 않기:

```bash
find / -xdev -type f -name '*.conf' 2>/dev/null
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 삭제 전 필수 검증

검색 결과 출력:

```bash
find /var/log/myapp -type f -name '*.log' -mtime +30 -print
```

결과 개수:

```bash
find /var/log/myapp -type f -name '*.log' -mtime +30 -print |
wc -l
```

상세 정보:

```bash
find /var/log/myapp -type f -name '*.log' -mtime +30 -ls
```

예상 삭제 용량:

```bash
find /var/log/myapp -type f -name '*.log' -mtime +30 \
  -exec du -ch -- {} + |
tail -n 1
```

파일 목록을 저장해 검토:

```bash
find /var/log/myapp -type f -name '*.log' -mtime +30 \
  -print0 > /tmp/delete-candidates.list
```

### 3-2. 실습 문제

#### 문제 1. 전체에서 `passwd` 검색

```bash
find / -name 'passwd' 2>/dev/null
```

#### 문제 2. `/etc`에서 `passwd` 검색

```bash
find /etc -name 'passwd'
```

#### 문제 3. 전체에서 `resolv.conf` 검색

```bash
find / -name 'resolv.conf' 2>/dev/null
```

#### 문제 4. `/etc`에서 `_config`로 끝나는 항목 검색

```bash
find /etc -name '*_config'
```

#### 문제 5. `/etc`에서 `b`로 시작하는 일반 파일 검색

```bash
find /etc -type f -name 'b*'
```

#### 문제 6. `/etc`에서 `b`로 시작하는 디렉터리 검색

```bash
find /etc -type d -name 'b*'
```

#### 문제 7. NetworkManager의 `ens160` 프로파일 검색

```bash
find /etc -type f -name 'ens160.nmconnection'
```

#### 문제 8. `/etc/aliases`보다 수정 시간이 최신인 일반 파일

```bash
find /etc -type f -newer /etc/aliases
```

#### 문제 9. `/etc/aliases`보다 수정 시간이 최신인 디렉터리

```bash
find /etc -type d -newer /etc/aliases
```

#### 문제 10. 2026년 7월 5일 이후 수정된 일반 파일

```bash
find /backup2 -type f -newermt '2026-07-05'
```

### 3-3. 대표 트러블슈팅

#### 시나리오 1. `find /home -name *.log` 결과가 이상하다

현재 디렉터리의 `*.log`를 셸이 먼저 확장할 수 있다.

잘못된 예:

```bash
find /home -name *.log
```

올바른 예:

```bash
find /home -name '*.log'
```

전달될 인자를 확인하려면:

```bash
printf '<%s>\n' *.log
```

#### 시나리오 2. 삭제 범위가 예상보다 넓다

위험한 예:

```text
find / -delete
```

이 명령은 실행하지 않는다.

안전한 접근:

```bash
TARGET=/var/log/myapp

: "${TARGET:?TARGET must be set}"

REAL_TARGET=$(realpath -e -- "$TARGET") || exit 1

case "$REAL_TARGET" in
  /var/log/myapp|/var/log/myapp/*)
    ;;
  *)
    echo "허용되지 않은 경로: $REAL_TARGET" >&2
    exit 1
    ;;
esac

find "$REAL_TARGET" \
  -mindepth 1 -type f -name '*.log' -mtime +30 -print
```

목록을 확인한 후 동일 조건에 `-delete`를 사용한다.

> **참고:** `set -euo pipefail`은 빈 변수나 일부 명령 실패를 처리하는 데 유용하지만, 잘못된 비어 있지 않은 경로를 자동으로 차단하지 않는다.

#### 시나리오 3. `Permission denied`가 대량으로 출력된다

표준 오류 숨김:

```bash
find / -name 'passwd' 2>/dev/null
```

접근 불가·가상 파일시스템 제외:

```bash
find / \
  \( -path /proc -o -path /sys -o -path /dev -o -path /run \) \
  -prune \
  -o -name 'passwd' -print
```

파일시스템 경계 유지:

```bash
find / -xdev -name 'passwd' 2>/dev/null
```

#### 시나리오 4. 시작 경로 자체가 결과에 포함된다

예:

```bash
find /backup2 -newermt '2026-07-05'
```

`/backup2` 자체도 조건을 만족하면 출력될 수 있다.

시작 경로 제외:

```bash
find /backup2 -mindepth 1 -newermt '2026-07-05'
```

일반 파일만 검색:

```bash
find /backup2 -mindepth 1 -type f -newermt '2026-07-05'
```

#### 시나리오 5. `-nouser -o -nogroup` 결과가 예상과 다르다

괄호로 OR 범위를 묶는다.

```bash
find /home \( -nouser -o -nogroup \) -print
```

일반 파일로 제한:

```bash
find /home -type f \( -nouser -o -nogroup \) -print
```

#### 시나리오 6. `-delete`와 `-prune`을 함께 사용했는데 제외가 되지 않는다

`-delete`는 깊이 우선 순회와 관련된 동작 때문에 `-prune`과 예상대로 조합되지 않을 수 있다.

먼저 `-prune`과 `-print`로 대상 목록을 만든다.

```bash
find /data \
  -path /data/keep -prune \
  -o -type f -name '*.tmp' -print
```

검증 후 `-exec rm` 방식 등을 사용한다.

```bash
find /data \
  -path /data/keep -prune \
  -o -type f -name '*.tmp' -exec rm -f -- {} +
```

>  **핵심 요약**
> - 기본 문법: `find 경로 조건 액션`
> - 이름 패턴: 반드시 따옴표 사용
> - `-newer`: 생성 시간이 아닌 수정 시간 비교
> - SUID 포함 검색: `-perm -4000`
> - 소유자·그룹 없음: 괄호로 OR 조건 묶기
> - 삭제 전: `-print` → 개수·용량 확인 → `-delete`
> - 관련: 파일 내용 출력 6종 (cat · head · tail · more · less · nl) · cat 리다이렉션 & Heredoc
