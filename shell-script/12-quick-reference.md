# Shell Script - 명령어 퀵 레퍼런스

변수·메타문자·산술·조건·반복·배열·위치 매개변수 문법을 빠르게 조회하는 암기 카드. 이해가 아니라 **"조회·복붙"** 이 목적이다.


---

## 목차

1. [변수 · 환경변수 문법 (Configuration)](#변수-환경변수-문법-configuration)
2. [Metacharacters 문법 (Configuration)](#metacharacters-문법-configuration)
3. [expr · let 문법 (Configuration)](#expr-let-문법-configuration)
4. [exit · test 문법 (Configuration)](#exit-test-문법-configuration)
5. [조건문 문법 (Configuration)](#조건문-문법-configuration)
6. [반복문 문법 (Configuration)](#반복문-문법-configuration)
7. [배열 · RANDOM 문법 (Configuration)](#배열-random-문법-configuration)
8. [위치 매개변수 문법 (Configuration)](#위치-매개변수-문법-configuration)
9. [⏰ cron · anacron 문법 (Configuration)](#cron-anacron-문법-configuration)
10. [빠른 조회표 (Configuration)](#빠른-조회표-configuration)
11. [검증 명령어 모음 (Verification)](#검증-명령어-모음-verification)
12. [요약](#요약)

---

## 변수 · 환경변수 문법 (Configuration)

```bash
name="값"                    # 변수 선언 (= 앞뒤 공백 금지)
echo "$name"                 # 변수 참조 (겹따옴표 권장)
echo "${name}추가문자"        # 변수명과 뒤 문자 구분

result=$(명령어)              # 명령 치환 (권장 방식)
result=`명령어`                # 명령 치환 (구식, 백틱)

export NAME=lee               # 환경 변수 등록 (자식 프로세스에 전달)
unset NAME                    # 변수/환경변수 삭제
env | grep NAME               # 환경 변수 확인

echo $PATH                     # 명령어 탐색 경로
echo $HOME                     # 홈 디렉터리
echo $SHELL                    # 로그인 시 기본 쉘
echo $0                        # 현재 사용 중인 쉘 / 스크립트 이름
```

---

## Metacharacters 문법 (Configuration)

### 1. 글롭(패턴 매칭)

```bash
*                # 0글자 이상 임의 문자열
?                # 정확히 1글자
[abc]            # a,b,c 중 1글자
[a-z] [0-9]      # 범위 지정
[!abc] [^abc]    # 부정 (a,b,c 제외)
```

### 2. 중괄호 확장

```bash
{a,b,c}            # 나열
{1..5}             # 범위
{1..10..2}         # 간격(step)
{01..10}           # 0 패딩
{A,B}{1,2}         # 중첩 조합
```

### 3. 변수 · 치환 · 산술

```bash
$변수              # 변수 참조
${변수}             # 변수명 명확히 구분
$(명령)             # 명령 치환
$((산술식))         # 산술 확장 (값 반환)
((산술식))          # 산술 평가 (종료 코드 반환)
~                  # 현재 사용자 홈 디렉터리
~계정명             # 특정 계정 홈 디렉터리
```

### 4. 리다이렉션 / 파이프

```bash
명령 > 파일          # 표준 출력 덮어쓰기
명령 >> 파일         # 표준 출력 추가
명령 2> 파일         # 표준 오류만 저장
명령 2>> 파일        # 표준 오류 추가
명령 > 파일 2>&1     # 정상+오류 모두 저장
명령 > /dev/null 2>&1  # 정상+오류 모두 숨김
명령 < 파일          # 파일을 입력으로
명령 << EOF ... EOF  # Here-document
명령1 | 명령2        # 파이프 (명령1 출력 -> 명령2 입력)
```

### 5. 명령 연결 / 제어

```bash
명령1; 명령2          # 무조건 순차 실행
명령1 && 명령2        # 명령1 성공 시에만 실행
명령1 || 명령2        # 명령1 실패 시에만 실행
{ 명령1; 명령2; }      # 명령 그룹화
```

---

## expr · let 문법 (Configuration)

```bash
expr 3 + 4             # 덧셈 (공백 필수)
expr 10 \* 2           # 곱셈 (* escape 필수)
expr 7 / 2             # 나눗셈 (정수)
expr 29 % 5            # 나머지
sum=$(expr 10 + 20)    # 결과를 변수에 저장

let x=3+5              # 대입 ($ 불필요)
let x++                # 1 증가
let x--                # 1 감소
let "x = 5 + 3"        # 공백 있으면 따옴표 필수
```

---

## exit · test 문법 (Configuration)

```bash
echo $?                 # 직전 명령 종료 코드 (0=성공)

test <조건식>
[ <조건식> ]              # [ ] 양쪽 공백 필수

# 숫자 비교
[ $x -eq $y ]  [ $x -ne $y ]  [ $x -gt $y ]
[ $x -lt $y ]  [ $x -ge $y ]  [ $x -le $y ]

# 문자열 비교
[ "$x" = "$y" ]  [ "$x" != "$y" ]
[ -z "$x" ]       [ -n "$x" ]

# 정규식 비교 (Bash 전용)
[[ "$x" =~ ^-?[0-9]+$ ]]      # 정수 여부 (음수 허용)

# 파일 테스트
[ -e $x ]  [ -f $x ]  [ -d $x ]  [ -L $x ]
[ -r $x ]  [ -w $x ]  [ -x $x ]  [ -s $x ]
[ $x -nt $y ]  [ $x -ot $y ]  [ $x -ef $y ]
```

**종료 코드 의미표**

| 코드 | 의미 |
| --- | --- |
| `0` | 성공 |
| `1` | 일반 오류 |
| `2` | Syntax error |
| `126` | 실행 불가(권한) |
| `127` | 명령 없음 |
| `128+N` | 신호로 종료 |

---

## 조건문 문법 (Configuration)

```bash
# if
if [ 조건 ]; then 명령; fi
if [ 조건 ]; then 명령1; else 명령2; fi
if [ 조건1 ]; then 명령1; elif [ 조건2 ]; then 명령2; else 명령3; fi

# case
case "$변수" in
    패턴1) 명령1 ;;
    패턴2|패턴3) 명령2 ;;
    *) 기본실행문 ;;
esac

# read
read -p "메시지" 변수         # 안내 메시지와 함께 입력
read -s -p "메시지" 변수       # 입력 내용 숨김 (비밀번호용)
read -t N -p "메시지" 변수     # N초 제한 입력
```

---

## 반복문 문법 (Configuration)

```bash
# for
for 변수 in 값1 값2 값3; do 명령; done
for 변수 in {1..N}; do 명령; done             # 리터럴 범위만 가능
for (( i=1; i<=N; i++ )); do 명령; done        # 변수 범위 가능 (권장)
for 변수 in "${배열[@]}"; do 명령; done         # 배열 개별 순회

# while / until
while [ 조건 ]; do 명령; done      # 참인 동안 반복
until [ 조건 ]; do 명령; done      # 거짓인 동안 반복

# 흐름 제어
break        # 반복문 자체 종료 (가장 안쪽 루프만)
break 2      # 바깥 루프까지 2단계 종료
continue     # 이번 회차만 건너뛰고 다음 회차로
```

---

## 배열 · RANDOM 문법 (Configuration)

### 1. 선언 · 조회

```bash
arr=("값1" "값2" "값3")        # 배열 선언 (요소는 공백 구분)
declare -a arr                 # 인덱스 배열 명시적 선언
declare -A map                 # 연관 배열 선언 (Bash 4.0+, 필수)

echo "${arr[0]}"               # 0번 요소 (인덱스는 0부터)
echo "${arr[@]}"               # 전체 요소 (개별 전개) ← 반복문 표준
echo "${arr[*]}"               # 전체 요소 (문자열 결합)
echo "${!arr[@]}"              # 인덱스 목록
echo "${#arr[@]}"              # 요소 개수
echo "${#arr[0]}"              # 0번 요소의 문자 길이 (개수 아님)
echo "${arr[@]:1:2}"           # 1번부터 2개 슬라이싱
echo "${arr[-1]}"              # 마지막 요소 (Bash 4.3+)
```

### 2. 추가 · 수정 · 삭제

```bash
arr[4]="값"                    # 인덱스 지정 추가
arr+=("값")                    # 최대 인덱스 다음에 추가 (권장)
arr[2]="새값"                  # 수정
unset 'arr[2]'                 # 요소 삭제 (인덱스 재정렬 안 됨)
arr=("${arr[@]}")              # 인덱스 재정렬 (구멍 제거)
unset arr                      # 배열 전체 삭제
```

### 3. 순회 · RANDOM

```bash
for item in "${arr[@]}"; do 명령; done       # 값 기준 순회
for i in "${!arr[@]}"; do 명령; done         # 인덱스 기준 (희소 배열 안전)

echo $(( RANDOM ))                          # 0 ~ 32767
echo $(( RANDOM % 101 ))                     # 0 ~ 100
echo $(( (RANDOM % 71) + 30 ))               # 30 ~ 100
echo $(( RANDOM % (max-min+1) + min ))       # min ~ max 일반식
RANDOM=42                                    # 시드 고정 (재현용)
shuf -i 1-100 -n 1                           # 편향 없는 균일 난수
```

**배열 조회 요약표**

| 문법 | 의미 |
| --- | --- |
| `${arr[@]}` | 전체 요소(개별 전개) |
| `${arr[*]}` | 전체 요소(문자열 결합) |
| `${!arr[@]}` | 인덱스 목록 |
| `${#arr[@]}` | 요소 개수 |
| `${#arr[0]}` | 0번 요소 문자 길이 |
| `${arr[@]:s:n}` | s번부터 n개 슬라이싱 |

---

## 위치 매개변수 문법 (Configuration)

```bash
$0            # 스크립트 이름 (이름만 필요하면 basename "$0")
$1 ~ $9       # 1~9번째 인자
${10}         # 10번째 이후는 중괄호 필수 ($10 은 $1+0)
$#            # 인자 개수 ($0 제외)
"$@"          # 전체 인자 개별 전개 ← 다른 명령에 넘길 때 표준
"$*"          # 전체 인자 문자열 결합 (메시지 출력용)
$?            # 직전 명령 종료 코드

$$            # 현재 스크립트 PID (임시파일명 등)
$!            # 마지막 백그라운드 프로세스 PID

shift         # $2→$1 로 이동, $# 1 감소
shift N       # N개 한 번에 이동
set -- a b c  # 위치 매개변수를 강제 재설정
${1:-기본값}   # 인자가 없으면 기본값 사용
```

### 1. 인자 검증 표준 3단 (복붙용)

```bash
# ① 개수
[ $# -ne 2 ] && { echo "Usage : $0 <arg1> <arg2>"; exit 1; }

# ② 형식 (가변 인자는 전체 순회)
for arg in "$@"; do
    [[ "$arg" =~ ^-?[0-9]+$ ]] || { echo "정수가 아닌 값 : $arg"; exit 1; }
done

# ③ 존재
[ -d "$1" ] || { echo "디렉터리가 없습니다 : $1"; exit 1; }
```

### 2. 가변 인자 처리

```bash
for arg in "$@"; do echo "$arg"; done           # 전체 순회
while [ $# -gt 0 ]; do echo "$1"; shift; done   # shift 방식
action="$1"; shift; targets=("$@")              # 첫 인자 소비 후 나머지 배열화
```

---

## ⏰ cron · anacron 문법 (Configuration)

### 1. cron 스케줄 형식

```bash
# 분  시  일  월  요일  [user]  command
  *   *   *   *    *           명령     # 매 분마다
  0   3   *   *    *           명령     # 매일 03:00
  0   3   *   *    1           명령     # 매주 월요일 03:00
  0   3   1   *    *           명령     # 매월 1일 03:00
*/5   *   *   *    *           명령     # 5분마다
  0  */2  *   *    *           명령     # 2시간마다
  *   *   *   *   1-5          명령     # 평일만 매 분

@reboot    명령    # 부팅 후 crond 시작 시 한 번
@hourly    명령    # 매 정각 (0 * * * *)
@daily     명령    # 매일 00:00
@weekly    명령    # 매주 일요일 00:00
@monthly   명령    # 매월 1일 00:00
```

### 2. crontab 관리 명령

```bash
systemctl status crond           # crond 실행 상태 확인
systemctl enable --now crond     # crond 시작 + 재부팅 후 자동 시작

crontab -e                       # 현재 사용자 cron 편집
crontab -l                       # 현재 사용자 cron 목록
crontab -r                       # 현재 사용자 cron 전체 삭제

tail -f /var/log/cron            # cron 실행 로그 실시간 모니터링
```

### 3. 백업 스크립트 + cron 등록 패턴 (복붙용)

```bash
#!/bin/bash
SRC="/원본/"  DEST="/백업/"  LOG="/var/log/backup.log"
DATE=$(date +%F)
[ ! -d "$DEST" ] && mkdir -p "$DEST"
tar czf "${DEST}backup_${DATE}.tar.gz" -C "$SRC" . >> "$LOG" 2>&1

# /etc/crontab 또는 crontab -e 에 등록 (절대 경로 필수)
# 0 3 * * * root /script/backup.sh >> /var/log/backup.log 2>&1
```

### 4. 오래된 파일 자동 삭제

```bash
# 7일 초과 파일 목록 출력 후 삭제
find /backup -type f -mtime +7 -print -delete >> /var/log/cleanup.log
```

### 5. anacron 관리

```bash
# /etc/anacrontab 주요 필드
# 주기(일)  지연(분)  식별자         명령
# 1         5        cron.daily     nice run-parts /etc/cron.daily
# 7         25       cron.weekly    nice run-parts /etc/cron.weekly
# @monthly  45       cron.monthly   nice run-parts /etc/cron.monthly

anacron -T              # anacrontab 문법 검사
anacron -n -f           # 지연 생략 + 날짜 무관 강제 실행
run-parts --test /etc/cron.daily/   # 실행 대상 파일 사전 확인
ls /var/spool/anacron/              # 마지막 실행 날짜 기록 위치
```

### 6. cron·anacron 연동 패턴

```bash
# /etc/crontab : anacron 불가 시 cron이 daily 직접 실행
0 3 * * * root test -x /usr/sbin/anacron || (cd / && env RUN_BY=CRON run-parts /etc/cron.daily)

# 실행 주체 로그 기록용 daily 스크립트 (파일명에 .sh 없이)
# /etc/cron.daily/daily_test
#!/bin/bash
echo "$(date '+%F %T') - 실행 주체 : ${RUN_BY:-UNKNOWN}" >> /var/log/daily_test.log
# ${RUN_BY:-UNKNOWN} : 환경변수 RUN_BY가 없으면 UNKNOWN 출력
```

### 7. tar 압축 관련

```bash
tar czf backup.tar.gz -C /src .    # 압축 생성 (-C: 기준 디렉터리 변경)
tar tzf backup.tar.gz              # 압축 풀지 않고 목록만 확인 (-t)
tar xzf backup.tar.gz -C /dest    # 압축 해제 (-x)
# c=create  z=gzip  f=파일명  t=list  x=extract  C=change dir
```

### 8. 테스트용 파일 생성

```bash
touch -d "8 days ago"  /backup/log/oldFile8.log    # 8일 전 타임스탬프로 파일 생성
touch -d "10 days ago" /backup/log/oldFile10.log
find /backup/log -type f -mtime +7 -print           # 삭제 대상 목록 확인
find /backup/log -type f -mtime +7 -delete          # 삭제 실행
```

---

## 빠른 조회표 (Configuration)

### 1. 산술 연산자

```bash
+  -  *  /  %  **        # 사칙연산, 나머지, 거듭제곱 (/ 는 정수 나눗셈)
++변수  변수++            # 전위/후위 증가
--변수  변수--            # 전위/후위 감소
+=  -=  *=  /=  %=       # 복합 대입
```

### 2. 비교/논리 연산자 (`$(( ))`, `(( ))` 안에서)

```bash
<  >  <=  >=  ==  !=      # 비교 (참=1, 거짓=0 값 반환)
&&  ||  !                  # 논리 AND/OR/NOT
```

### 3. 파일 디스크립터 번호

| 번호 | 이름 |
| --- | --- |
| `0` | 표준 입력(stdin) |
| `1` | 표준 출력(stdout) |
| `2` | 표준 오류(stderr) |

### 4. 세미콜론 vs && vs ||

```bash
;    : 무조건 다음 명령 실행               → 단순 순차 실행
&&   : 앞 명령 성공(exit 0) 시 실행         → 조건 기반 AND
||   : 앞 명령 실패(exit 1 이상) 시 실행     → 조건 기반 OR
```

### 5. 혼동하기 쉬운 항목

| 비슷한 문법 | 차이 |
| --- | --- |
| `"${arr[@]}"` vs `"${arr[*]}"` | 개별 전개 vs 문자열 결합 |
| `"$@"` vs `"$*"` | 개별 전개 vs 문자열 결합 |
| `${#arr[@]}` vs `${#arr[0]}` | 요소 개수 vs 0번 요소 문자 길이 |
| `$#` vs `${#arr[@]}` | 인자 개수 vs 배열 요소 개수 |
| `$10` vs `${10}` | `$1`+`0` vs 10번째 인자 |
| `$(( ))` vs `(( ))` | 값 반환 vs 종료 코드 반환 |
| `[ ]` vs `****` | POSIX test vs Bash 확장(정규식·패턴) |
| `=` vs `-eq` | 문자열 비교 vs 숫자 비교 |
| `>` vs `>>` | 덮어쓰기 vs 추가 |
| `break` vs `continue` | 반복문 종료 vs 회차 건너뛰기 |
| `exit` vs `return` | 스크립트 종료 vs 함수 종료 |
| `unset arr` vs `unset 'arr[2]'` | 배열 전체 삭제 vs 요소 1개 삭제 |

---

## 검증 명령어 모음 (Verification)

```bash
bash -n <스크립트>              # 문법 사전 검사
bash -x <스크립트> arg1 arg2     # 인자·배열 전개 추적
chmod +x <스크립트>              # 실행 권한 부여
echo $?                        # 종료 코드 확인
echo "$변수"                    # 변수 값 확인
declare -p <배열명>              # 배열 구조 전체 확인
printf '[%s]\n' "${arr[@]}"     # 요소 경계 확인
printf '[%s]\n' "$@"            # 인자 경계 확인
printf '%s\n' <패턴>             # 글롭/중괄호 확장 미리보기
env | grep <변수명>              # 환경 변수 확인
type <명령어>                    # 명령어 종류(내장/외부) 확인
stat -c "%U %G %a %n" <경로>     # 소유자/그룹/권한/경로 확인
echo $BASH_VERSION              # Bash 버전 확인 (연관 배열 4.0+)
```

---

## 요약
- 📌 **핵심 요약**
- 변수 계산: `$(( ))` / `expr` / `let` — 문자열 그대로는 계산 불가
- 조건 판단: 항상 종료 코드 기준, `[ ]` 양쪽 공백 필수
- 반복 목록에 변수 범위가 필요하면 `for (( ))` 사용
- 배열·인자 전개는 예외 없이 `"${arr[@]}"` / `"$@"`, 개수는 `${#arr[@]}` / `$#`
- 스크립트 첫 로직은 인자 3단 검증(개수 → 형식 → 존재) + `exit 1`
- 관련: **10. 🧩 Shell Script - 통합 정리** · **11. 🚑 Shell Script - 트러블슈팅 치트시트** · **9. ⏰ Shell Script - cron · anacron (스케줄 자동화)** · **1. 🐚 Shell Script - 변수와 환경변수 (커널·쉘 개념 포함)** · **2. 🧩 Shell Script - Metacharacters (메타문자)** · **3. 🔢 Shell Script - expr · let (산술 연산)** · **4. 🚦 Shell Script - exit 상태와 test 명령** · **5. 🔀 Shell Script - 조건문 (if · case)** · **6. 🔁 Shell Script - 반복문 (for · while · until)** · **7. 📦 Shell Script - 배열(Array)과 RANDOM** · **8. 🎯 Shell Script - 위치 매개변수 (Positional Parameters)**