# ⏰ Shell Script - cron · anacron (스케줄 자동화)

`cron`은 Linux의 시간 기반 자동화 스케줄러로, `crond` 데몬이 `crontab`에 등록된 작업을 지정 시간에 실행한다. 스케줄은 **분·시·일·월·요일** 5개 필드로 표현하며, 스크립트 경로·실행 권한·출력 리다이렉션이 필수 주의 항목이다. `anacron`은 시스템이 꺼져 있는 동안 놓친 일·주·월 단위 작업을 부팅 후 자동으로 보완 실행한다.


---

## 목차

1. [개요 (Overview)](#개요-overview)
2. [표준 설정 템플릿 (Configuration)](#표준-설정-템플릿-configuration)
3. [검증 및 트러블슈팅 (Verification & Troubleshooting)](#검증-및-트러블슈팅-verification-troubleshooting)
4. [요약](#요약)

---

## 개요 (Overview)

cron은 지정한 시간 또는 주기에 명령어·스크립트를 자동 실행하는 Linux 스케줄러이며, **crond**(데몬)와 **crontab**(설정 파일) 두 요소로 동작한다. **crond** 는 백그라운드에서 상시 실행되며 스케줄을 감시하다가 지정 시간에 작업을 실행하는 데몬으로, crond가 중지되면 등록된 작업은 실행되지 않는다. **crontab** 은 언제, 어떤 명령을 실행할지 기록하는 설정 파일로, 사용자별 crontab과 시스템 전체용 `/etc/crontab`으로 나뉜다.

| 명령 | 설명 |
|---|---|
| `crontab -e` | 현재 사용자의 cron 작업 편집 |
| `crontab -l` | 현재 사용자의 cron 작업 목록 확인 |
| `crontab -r` | 현재 사용자의 cron 작업 전체 삭제 |

cron 스케줄은 **분·시·일·월·요일** 5개 필드로 표현하며, `/etc/crontab`에서는 사용자 이름 필드가 추가된다.

```text
┌─────────────  분   0~59
│ ┌───────────  시   0~23
│ │ ┌─────────  일   1~31
│ │ │ ┌───────  월   1~12
│ │ │ │ ┌─────  요일  0~7 (0과 7은 일요일)
│ │ │ │ │
* * * * *  [user]  command
```

| 예시 | 의미 |
|---|---|
| `0 3 * * *` | 매일 새벽 03시 00분 |
| `5 * * * *` | 매시간 5분 |
| `0 1 * * 1` | 매주 월요일 01시 |
| `30 18 1 * *` | 매월 1일 18시 30분 |
| `0 0 1 1 *` | 매년 1월 1일 00시 |

반복 실행에는 `*/N` 으로 N 단위 반복을, `@키워드`로 자주 쓰는 주기를 간단히 표현할 수 있다.

| 표현 | 의미 |
|---|---|
| `*/5 * * * *` | 매 5분마다 (0, 5, 10분…) |
| `0 */2 * * *` | 매 2시간마다 (0시, 2시, 4시…) |
| `* * * * 1-5` | 평일(월~금)만 매 분 |
| `@reboot` | 부팅 후 crond 시작 시 한 번 |
| `@hourly` | 매 정각 (`0 * * * *`) |
| `@daily` | 매일 00:00 (`0 0 * * *`) |
| `@weekly` | 매주 일요일 00:00 |
| `@monthly` | 매월 1일 00:00 |
| `@yearly` | 매년 1월 1일 00:00 |

cron은 일반 쉘과 다른 환경에서 실행되므로 **절대 경로**, **실행 권한**, **출력 리다이렉션** 세 가지를 반드시 지켜야 한다. 첫째, 절대 경로가 필수인데 cron 환경의 PATH는 `/sbin:/bin:/usr/sbin:/usr/bin`으로 제한되므로 `echo`, `ls`도 `/bin/echo`, `/usr/bin/ls`로 지정해야 한다. 둘째, 스크립트 실행 권한으로 `chmod +x script.sh`로 실행 권한 부여가 필수다. 셋째, 출력 리다이렉션인데 cron은 화면이 없으므로 stdout·stderr를 파일에 저장해야 나중에 확인 가능하며, 리눅스 출력은 **stdout(fd 1)**과 **stderr(fd 2)** 두 종류로 나뉜다.

```bash
# ① stdout과 stderr 각각 다른 파일에 저장
/script/backup.sh > /var/log/out.log 2> /var/log/err.log
# ② stdout + stderr를 하나의 파일에 누적 저장 (cron 표준, 가장 많이 사용)
0 3 * * * root /home/user/backup.sh >> /var/log/backup.log 2>&1
# ③ 출력이 불필요한 경우 버림
0 3 * * * root /script/cleanup.sh > /dev/null 2>&1
```

넷째, 환경 변수는 스크립트 내에서 사용하는 것을 직접 선언하거나 절대 경로를 사용해야 한다.

cron은 정확한 시간에 실행하되 시스템이 꺼져 있으면 작업을 놓치는 반면, anacron은 시스템이 꺼져 있는 동안 놓친 일·주·월 단위 작업을 부팅 후 자동으로 실행한다.

| 구분 | cron | anacron |
|---|---|---|
| 실행 기준 | 정확한 시각 (분 단위) | 실행 주기 (일·주·월) |
| 분 단위 작업 | 가능 | 불가 |
| 시스템 꺼짐 | 작업 놓침 | 부팅 후 보완 실행 |
| 적합한 환경 | 상시 켜진 서버 | 노트북·개인 PC |

anacron은 `/etc/anacrontab`에 설정하고, `/etc/cron.daily`, `/etc/cron.weekly`, `/etc/cron.monthly` 디렉터리와 연동된다. `run-parts` 명령이 지정 디렉터리 내 실행 가능한 스크립트를 일괄 실행하는데, 이때 파일명에 `.sh` 확장자가 있으면 실행 대상에서 제외될 수 있으므로 확장자 없이 작성한다.

anacron과 cron을 연동할 때는 `/usr/sbin/anacron`의 실행 권한 유무를 `test -x`로 확인하여, anacron이 가능하면 anacrontab이 daily 작업을 담당하고, 불가능하면 cron이 `run-parts`로 직접 `/etc/cron.daily`를 실행하도록 설정한다. `/etc/crontab`에 다음 항목을 등록하면 anacron 실행 불가 시 cron이 자동으로 대신 처리한다.

```bash
# anacron 실행 권한이 없으면 cron이 daily 작업 직접 실행
0 3 * * * root test -x /usr/sbin/anacron || (cd / && env RUN_BY=CRON run-parts /etc/cron.daily)
```

`env RUN_BY=CRON`처럼 환경 변수를 주입하면 스크립트 안에서 실행 주체를 로그에 기록할 수 있다.

마지막으로, 리눅스 명령 출력은 **표준 출력(stdout, 파일 디스크립터 1)**과 **표준 에러(stderr, 파일 디스크립터 2)**로 구분되며, cron 환경에서는 화면이 없으므로 두 출력 모두 파일로 저장해야 실행 결과를 확인할 수 있다.

| 구분 | 파일 디스크립터 | 내용 |
|---|---|---|
| 표준 출력(stdout) | 1 | 정상 실행 결과 |
| 표준 에러(stderr) | 2 | 오류 메시지 |

`2>&1` 의 의미는 "stderr(fd 2)를 stdout(fd 1)이 가리키는 곳으로 보내라"이며, 즉 `>> log 2>&1` 은 stdout과 stderr를 동일한 파일에 누적 저장한다. `>>` (append)를 써야 기존 로그가 지워지지 않으며, `>` (overwrite)는 실행할 때마다 이전 로그를 덮어쓴다.

---

## 표준 설정 템플릿 (Configuration)

> **적용 환경:** Bash 기반 Linux 셸 환경 (RHEL 계열 기본 `/bin/bash`).

### Step 1. crond 상태 확인 및 활성화

```bash
systemctl status crond
systemctl start crond
systemctl enable crond    # 재부팅 후 자동 시작
```

### Step 2. crontab 편집

```bash
crontab -e    # 편집
crontab -l    # 목록 확인
crontab -r    # 전체 삭제
```

### Step 3. /etc/crontab 형식 (시스템 전체)

```bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# 분  시  일  월  요일  사용자  명령
*/5  *  *  *  *  root  /script/hourly/login_user_check.sh
0    3  *  *  *  root  /home/user/backup.sh >> /var/log/backup.log 2>&1
40  18  *  *  1-5 root  /script/hourly/temp_backup.sh
30  23  *  *  *  root   /script/hourly/log_backup.sh
@reboot              root  /script/hourly/reboot_check.sh
```

### Step 4. 백업 스크립트 기본 템플릿

```bash
#!/bin/bash

SRC="/원본경로/"
DEST="/백업경로/"
LOG="/var/log/backup.log"
DATE=$(date +%F)
BACKUP_FILE="${DEST}backup_${DATE}.tar.gz"

[ ! -d "$DEST" ] && mkdir -p "$DEST"

echo "==========================================" >> "$LOG"
echo "$(date '+%F %T') - 백업 시작" >> "$LOG"

tar czf "$BACKUP_FILE" -C "$SRC" .

if [ $? -eq 0 ]; then
    echo "$(date '+%F %T') - 백업 성공 : $BACKUP_FILE" >> "$LOG"
else
    echo "$(date '+%F %T') - 백업 실패" >> "$LOG"
fi
```

### Step 5. 오래된 파일 자동 삭제 패턴

```bash
# 수정된 지 7일 초과한 파일 목록 출력 후 삭제
find /backup/log -type f -mtime +7 -print -delete >> /var/log/cleanup.log
```

### Step 6. /etc/anacrontab 형식

```bash
SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
RANDOM_DELAY=45
START_HOURS_RANGE=3-22

# 주기(일)  지연(분)  작업식별자        명령
1           5        cron.daily        nice run-parts /etc/cron.daily
7           25       cron.weekly       nice run-parts /etc/cron.weekly
@monthly    45       cron.monthly      nice run-parts /etc/cron.monthly
```

### Step 7. /etc/cron.d 디렉터리 구조

```bash
ls -l /etc/ | grep cron
# /etc/cron.d        : 패키지별 개별 cron 파일
# /etc/cron.hourly   : 매시간 실행 스크립트
# /etc/cron.daily    : 매일 실행 스크립트
# /etc/cron.weekly   : 매주 실행 스크립트
# /etc/cron.monthly  : 매월 실행 스크립트
# /etc/crontab       : 시스템 전체 cron 설정
# /etc/anacrontab    : anacron 설정
```

### Step 8. 접속자 모니터링 스크립트 (EX1)

접속한 사용자 정보를 5분마다 자동으로 기록하는 패턴이다.

```bash
#!/bin/bash
# /script/hourly/login_user_check.sh

LOG="/var/log/login_user_check.log"

echo "=================================================="  >> "$LOG"
echo "확인시간 : $(date '+%F %T')"  >> "$LOG"
who >> "$LOG"
```

```bash
mkdir -p /script/hourly
chmod +x /script/hourly/login_user_check.sh
/script/hourly/login_user_check.sh          # 수동 실행 후 로그 확인
cat /var/log/login_user_check.log

# /etc/crontab 등록 (5분마다)
*/5 * * * * root /script/hourly/login_user_check.sh
```

### Step 9. 설정 파일 정기 백업 스크립트 (EX2)

지정 디렉터리를 tar.gz로 압축하고 결과를 로그에 기록하는 패턴이다.

```bash
#!/bin/bash
# /script/hourly/temp_backup.sh

SRC="/temp"
DEST="/backup/etc/"
LOG="/var/log/temp_backup.log"
DATE=$(date +%F)                              # 반드시 대문자 %F (소문자 %f는 마이크로초)
BACKUP_FILE="${DEST}temp_${DATE}.tar.gz"

if [ ! -d "$DEST" ]
then
    mkdir -p "$DEST"
fi

tar czf "$BACKUP_FILE" -C "$SRC" .
# -c : 새 압축 파일 생성 (create)
# -z : gzip 방식으로 압축
# -f : 생성할 파일 이름 지정
# -C : 압축 전 기준 디렉터리 변경 (change directory)

if [ $? -eq 0 ]
then
    echo "$(date '+%F %T') - 백업 성공 : $BACKUP_FILE" >> "$LOG"
else
    echo "$(date '+%F %T') - 백업 실패" >> "$LOG"
fi
```

```bash
chmod +x /script/hourly/temp_backup.sh
/script/hourly/temp_backup.sh

# 백업 내용 확인 (압축 풀지 않고 목록만)
tar tzf /backup/etc/temp_2026-07-30.tar.gz
# -t : 압축 풀지 않고 내부 목록만 확인

# /etc/crontab 등록 (평일 18시 40분)
40 18 * * 1-5 root /script/hourly/temp_backup.sh
```

### Step 10. 백업 + 자동 정리 조합 스크립트 (EX3)

백업 성공 시에만 오래된 파일을 삭제하는 조합 패턴이다. `find -print -delete`로 삭제 전 목록을 로그에 기록한 뒤 삭제한다.

```bash
#!/bin/bash
# /script/hourly/log_backup.sh

SRC="/backup/log/"
DEST="/temp/log/"
LOG="/var/log/log_cleanup.log"
DATE="$(date '+%F')"
BACKUP_FILE="${DEST}log_${DATE}.tar.gz"

[ ! -d "$DEST" ] && mkdir -p "$DEST"

echo "=================================================="  >> "$LOG"
echo "$(date '+%F %T') - 로그 백업 시작"  >> "$LOG"

tar czf "$BACKUP_FILE" -C "$SRC" .

if [ $? -eq 0 ]
then
    echo "$(date '+%F %T') - 백업 성공 : $BACKUP_FILE" >> "$LOG"
    echo "$(date '+%F %T') - 장기 로그 파일 삭제 시작"  >> "$LOG"
    find "$SRC" -type f -mtime +7 -print -delete >> "$LOG"
    echo "$(date '+%F %T') - 장기 로그 파일 삭제 완료"  >> "$LOG"
else
    echo "$(date '+%F %T') - 백업 실패" >> "$LOG"
fi
```

```bash
# 테스트용 오래된 파일 생성 (touch -d 로 과거 타임스탬프 지정)
touch -d "8 days ago"  /backup/log/oldFile8.log
touch -d "10 days ago" /backup/log/oldFile10.log

chmod +x /script/hourly/log_backup.sh
/script/hourly/log_backup.sh
cat /var/log/log_cleanup.log    # 삭제된 파일 목록 + 백업 성공 확인

# /etc/crontab 등록 (매일 23시 30분)
30 23 * * * root /script/hourly/log_backup.sh
```

### Step 11. anacron · cron 연동 상세 패턴 (EX4)

anacron 실행 권한 유무에 따라 daily 작업 주체를 자동으로 전환하고, 실행 주체를 로그로 확인하는 패턴이다.

```bash
# 1. /etc/cron.daily/daily_test 작성 (확장자 .sh 없이 → run-parts 실행 대상 포함)
#!/bin/bash
LOG="/var/log/daily_test.log"
echo "$(date '+%F %T') - 실행 주체 : ${RUN_BY:-UNKNOWN}" >> "$LOG"
# ${RUN_BY:-UNKNOWN} : 환경변수 RUN_BY가 있으면 그 값, 없으면 UNKNOWN 출력
```

```bash
chmod +x /etc/cron.daily/daily_test
run-parts --test /etc/cron.daily/    # 실제 실행 없이 실행 대상 파일만 확인
```

```bash
# 2. /etc/anacrontab 설정 추가
# 주기(일)  지연(분)  식별자            명령
1           1        cron.daily_test   env RUN_BY=ANACRON run-parts /etc/cron.daily

anacron -T              # anacrontab 문법 검사
anacron -n -f           # 지연 없이 날짜 무관 강제 실행 (-n: 지연생략, -f: 날짜무관)
cat /var/log/daily_test.log   # 실행 주체 : ANACRON 확인

# /var/spool/anacron/ : anacron이 마지막 실행 날짜를 기록 (당일 재실행 방지)
ls /var/spool/anacron/
```

```bash
# 3. /etc/crontab 폴백 설정 (anacron 권한 없을 때 cron이 대신 처리)
55 15 * * * root test -x /usr/sbin/anacron || (cd / && env RUN_BY=CRON run-parts /etc/cron.daily)

# anacron 실행 권한 제거 → cron 폴백 테스트
chmod -x /usr/sbin/anacron
test -x /usr/sbin/anacron; echo $?    # 1 (권한 없음)
# cron 실행 후 로그에 '실행 주체 : CRON' 확인

# 복원
chmod +x /usr/sbin/anacron
test -x /usr/sbin/anacron; echo $?    # 0 (권한 있음)
```

---

## 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
systemctl status crond                   # crond 실행 상태 확인
crontab -l                               # 현재 사용자 작업 목록
tail -f /var/log/cron                    # cron 실행 로그 실시간 모니터링
run-parts --test /etc/cron.daily/        # daily 실행 대상 파일 확인 (실제 실행 안 함)
anacron -T                               # anacrontab 문법 검사
anacron -n -f                            # anacron 강제 즉시 실행 (-n: 지연 생략, -f: 무조건 실행)
test -x /usr/sbin/anacron; echo $?       # anacron 실행 권한 확인 (0=있음, 1=없음)
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. 스크립트를 수동 실행하면 되는데 cron에서는 실행되지 않음

- **원인:** cron 환경의 PATH가 제한적이어서 명령어 경로를 찾지 못함.
- **해결:** 스크립트 내 또는 crontab에 명시적 경로 지정.
  ```bash
  # crontab 또는 스크립트 상단에 PATH 선언
  PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
  # 또는 명령 절대 경로 사용
  /usr/bin/find /backup -type f -mtime +7 -delete
  ```

#### 🚨 시나리오 2. cron 작업 결과를 확인할 수 없음

- **원인:** cron은 백그라운드 실행이라 화면 출력이 없음.
- **해결:** stdout·stderr를 파일로 리다이렉션 후 확인.
  ```bash
  # 오류까지 같은 파일에 저장
  0 3 * * * /home/user/script.sh >> /var/log/script.log 2>&1
  # 실시간 모니터링
  tail -f /var/log/script.log
  ```

#### 🚨 시나리오 3. `run-parts`가 스크립트를 실행하지 않음

- **원인:** 스크립트 파일명에 `.sh` 확장자가 있거나 실행 권한이 없음.
- **해결:**
  ```bash
  # 확장자 제거 (daily_test.sh → daily_test)
  mv /etc/cron.daily/daily_test.sh /etc/cron.daily/daily_test
  chmod +x /etc/cron.daily/daily_test
  run-parts --test /etc/cron.daily/    # 실행 대상에 포함됐는지 확인
  ```

#### 🚨 시나리오 4. cron 작업이 등록됐는데도 실행되지 않음

- **원인 진단 순서:**
  1. `systemctl status crond` — crond 실행 중인지 확인
  2. `tail /var/log/cron` — cron 로그에서 작업 실행 흔적 확인
  3. `crontab -l` — 작업이 실제로 등록됐는지 확인
  4. 스크립트에 실행 권한(`chmod +x`) 있는지 확인
  5. 스크립트 내 경로가 모두 절대 경로인지 확인

#### 🚨 시나리오 5. anacron 작업이 이미 실행된 날 재실행이 안 됨

- **원인:** anacron은 마지막 실행 날짜를 `/var/spool/anacron/` 에 기록하고, 같은 날은 재실행하지 않음.
- **해결:** 강제 실행 옵션 사용.
  ```bash
  anacron -n -f    # 지연 생략 + 날짜 무관 강제 실행
  ```

#### 🚨 시나리오 6. 로그 파일에 날짜 대신 `%f` 문자 또는 마이크로초가 출력됨

- **원인:** `date` 포맷에서 대문자 `%F`(YYYY-MM-DD) 대신 소문자 `%f`를 사용하면 마이크로초(또는 플랫폼에 따라 문자 그대로) 출력.
  ```bash
  DATE=$(date +%f)    # ❌ → '%f' 또는 마이크로초 (의도와 다름)
  DATE=$(date +%F)    # ✅ → 2026-07-30 (YYYY-MM-DD)
  ```
- **해결:** 백업 파일명과 로그 날짜에는 반드시 대문자 `%F` 사용. 날짜+시간 전체는 `date '+%F %T'`.

#### 🚨 시나리오 7. 오래된 파일 삭제를 테스트할 파일이 없음

- **원인:** 실제 7일 이상 된 파일이 없어서 `find -mtime +7` 이 아무것도 반환하지 않음.
- **해결:** `touch -d`로 과거 타임스탬프를 가진 테스트 파일을 즉시 생성.
  ```bash
  touch -d "8 days ago"  /backup/log/test8.log
  touch -d "10 days ago" /backup/log/test10.log
  find /backup/log -type f -mtime +7 -print    # 삭제 전 대상 확인
  find /backup/log -type f -mtime +7 -delete   # 삭제 실행
  ```

---

## 요약
- 📌 **핵심 요약**
- cron 구성: `crond`(데몬) + `crontab`(설정) — crond가 중지되면 작업 미실행
- 스케줄 형식: `분 시 일 월 요일 [user] command` — `*/N`으로 N단위 반복, `@daily` 등 예약어 사용 가능
- 필수 3원칙: **절대 경로** · **실행 권한(`chmod +x`)** · **출력 리다이렉션(`>> log 2>&1`)**
- anacron: 시스템이 꺼져 있는 동안 놓친 일·주·월 작업을 부팅 후 보완 실행 (`anacron -n -f`로 강제 실행)
- cron·anacron 연동: `test -x /usr/sbin/anacron || run-parts /etc/cron.daily`
- 관련: **8. 🎯 Shell Script - 위치 매개변수 (Positional Parameters)** · **5. 🔀 Shell Script - 조건문 (if · case)** · **6. 🔁 Shell Script - 반복문 (for · while · until)** · **10. 🧩 Shell Script - 통합 정리**

