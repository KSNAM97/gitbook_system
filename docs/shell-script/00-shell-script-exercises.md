# Shell Script 종합 연습문제

> **Tag:** #ShellScript #Bash #연습문제 #변수 #Metacharacters #expr #let #exit #test #조건문 #case #반복문 #배열 #RANDOM #위치매개변수 #chmod #chown #umask #특수권한 #find #SCP #SSH #vsftpd #심화
>
> **목적:** 강의 내용을 스스로 떠올려 풀면서 손에 익히는 것. 답을 보기 전에 반드시 직접 실행해볼 것.
>
> **난이도:**  기초(Level 1) →  응용(Level 2) →  심화(Level 3, 실무형 종합)

---

## 문제 구성 & 관련 문서

| PART | 주제 | 문항 | 관련 문서 |
| --- | --- | --- | --- |
| 1 | 변수 & 환경변수 | Q1~Q6 | Shell Script - 변수와 환경변수 (커널·쉘 개념 포함) |
| 2 | Metacharacters | Q7~Q13 | Shell Script - Metacharacters (메타문자) |
| 3 | expr & let | Q14~Q17 | Shell Script - expr · let (산술 연산) |
| 4 | exit & test | Q18~Q20 | Shell Script - exit 상태와 test 명령 |
| 5 | 조건문 (if) | Q21~Q26 | Shell Script - 조건문 (if · case) |
| 6 | 반복문 (for/while/until) | Q27~Q32 | Shell Script - 반복문 (for · while · until) |
| 7 | 배열 (Array) | Q33~Q36 | Shell Script - 배열(Array)과 RANDOM |
| 8 | 종합 스크립트 | Q37~Q40 | Shell Script - 통합 정리 |
| 9 | 위치 매개변수 | Q41~Q45 | Shell Script - 위치 매개변수 (Positional Parameters) |
| 10 | case & read | Q46~Q49 | Shell Script - 조건문 (if · case) |
| 11 | break·continue·폴링 | Q50~Q53 | Shell Script - 반복문 (for · while · until) |
| 12 | 배열 심화 | Q54~Q56 | Shell Script - 배열(Array)과 RANDOM |
| 13 |  실무 종합 심화 | Q57~Q60 | Shell Script - 통합 정리 · Shell Script - 트러블슈팅 치트시트 |
| 14 | cron · anacron | Q61~Q65 | Shell Script - cron · anacron (스케줄 자동화) |
| 15 | 허가권 · 소유권 · find | Q66~Q70 | 허가권 상세 (chmod & 8진수 · 심볼릭 표기) · Umask · 소유권 & 특수 권한 |
| 16 | 네트워크 서비스 자동화 | Q71~Q73 | SSH 개념 & 프로세스·보안 설정 · SCP 파일 전송 · vsFTP 설치 & 접근 제어 |

> 명령어 문법이 기억나지 않을 때는 Shell Script - 명령어 퀵 레퍼런스, 오류가 났을 때는 Shell Script - 트러블슈팅 치트시트 를 먼저 확인한다.

---

## PART 1. 변수 & 환경변수

>  관련 문서: Shell Script - 변수와 환경변수 (커널·쉘 개념 포함)

### 핵심 개념 복습

```text
변수 선언     : name=value  (= 앞뒤 공백 없음)
변수 참조     : $name  또는  ${name}
명령 치환     : today=$(date)
산술 계산     : $((num1 + num2))
환경 변수     : export VAR=value  (자식 프로세스에 전달)
변수 삭제     : unset VAR
```

---

### 기초 (Level 1)

**Q1.** 변수 `city`에 `Seoul`을 저장하고 `I live in Seoul` 형식으로 출력하시오.

>  **힌트:** 변수 앞에 `$`를 붙이면 값으로 치환된다. 공백 있는 문장은 `""`로 묶는다.

```bash
# 예제
fruit="apple"
echo "I like $fruit"
# 출력: I like apple
```

> [!success]-  정답 보기
>
> ```bash
> city="Seoul"
> echo "I live in $city"
> # 출력: I live in Seoul
> ```


---

**Q2.** 변수 `a=100`, `b=37`을 선언하고 `a - b`의 결과를 출력하시오.

>  **힌트:** 문자열로 저장된 숫자를 계산하려면 `$(( ))` 산술 확장을 사용한다.

```bash
# 예제
x=10
y=3
echo $((x * y))    # 출력: 30
```

> [!success]-  정답 보기
>
> ```bash
> a=100
> b=37
> echo $((a - b))
> # 출력: 63
> ```


---

**Q3.** `hostname` 명령의 결과를 변수 `myhost`에 저장하고 출력하시오.

>  **힌트:** 명령어 실행 결과를 변수에 담으려면 `$(명령어)` 형태를 사용한다.

```bash
# 예제
now=$(date +%F)
echo "오늘 날짜: $now"
```

> [!success]-  정답 보기
>
> ```bash
> myhost=$(hostname)
> echo $myhost
> # 출력: Server-A  (서버 이름에 따라 다름)
> ```


---

### 응용 (Level 2)

**Q4.** 환경 변수 `BACKUP_DIR`에 `/backup/scripts`를 설정하고, `env` 명령으로 해당 변수가 출력되는지 확인하시오.

>  **힌트:** 일반 변수(`VAR=value`)는 `env`에 나타나지 않는다. `export`로 환경 변수로 승격해야 자식 프로세스와 `env`에 보인다.

```bash
# 잘못된 방법 vs 올바른 방법
MYVAR=test
env | grep MYVAR      # 출력 없음

export MYVAR=test
env | grep MYVAR      # MYVAR=test 출력
```

> [!success]-  정답 보기
>
> ```bash
> # 방법 1: 두 줄로
> BACKUP_DIR=/backup/scripts
> export BACKUP_DIR
>
> # 방법 2: 한 줄로
> export BACKUP_DIR=/backup/scripts
>
> # 확인
> env | grep BACKUP_DIR
> # 출력: BACKUP_DIR=/backup/scripts
> ```


---

**Q5.** 다음 조건을 모두 만족하는 스크립트를 작성하시오.

- 변수 `logdir`에 `/var/log` 경로를 저장한다.
- `$logdir` 안의 파일 개수를 `count` 변수에 저장한다.
- `"/var/log 안의 파일 수: XX개"` 형식으로 출력한다.

>  **힌트:** `ls 경로 | wc -l` 로 줄 수를 셀 수 있다. 결과를 `$()` 로 변수에 담는다.

```bash
# 예제
dir="/etc"
n=$(ls "$dir" | wc -l)
echo "$dir 안의 항목 수: ${n}개"
```

> [!success]-  정답 보기
>
> ```bash
> logdir=/var/log
> count=$(ls "$logdir" | wc -l)
> echo "$logdir 안의 파일 수: ${count}개"
> ```


---

**Q6.** `PS1` 환경 변수를 수정해 프롬프트가 `[시간 사용자@호스트 경로]$` 형식으로 출력되게 하시오. 설정 후 `~/.bashrc`에 저장해 재로그인 시에도 유지되도록 하시오.

>  **힌트:** `\t`=시간, `\u`=사용자, `\h`=호스트, `\W`=현재 디렉터리(짧게), `\$`=프롬프트 기호

> [!success]-  정답 보기
>
> ```bash
> # 즉시 적용
> PS1='[\t \u@\h \W]\$ '
>
> # ~/.bashrc에 저장 (재로그인 후에도 유지)
> echo "PS1='[\\t \\u@\\h \\W]\\$ '" >> ~/.bashrc
> source ~/.bashrc
> ```


---

## PART 2. Metacharacters (메타문자)

>  관련 문서: Shell Script - Metacharacters (메타문자)

### 핵심 개념 복습

```text
글롭 패턴  :  *  (0글자↑)  /  ?  (정확히 1글자)  /  [abc]  (문자 집합)
범위       :  [a-z]  [0-9]  [A-Z]
부정       :  [!abc]  또는  [^abc]
중괄호     :  {a,b,c}  /  {1..5}  /  {01..12}
명령 연결  :  cmd1 && cmd2  (앞이 성공해야 실행)
            cmd1 || cmd2  (앞이 실패하면 실행)
리다이렉션 :  >  (덮어쓰기)  /  >>  (이어쓰기)  /  <  (입력)
파이프     :  cmd1 | cmd2
```

---

### 기초 (Level 1)

**Q7.** `/etc` 디렉터리에서 `.conf`로 끝나는 파일 목록을 출력하시오.

>  **힌트:** `*` 는 0글자 이상 아무 문자와 매칭된다.

```bash
# 예제
ls /etc/*.conf     # .conf로 끝나는 모든 파일
ls /var/log/*.log  # .log로 끝나는 모든 파일
```

> [!success]-  정답 보기
>
> ```bash
> ls /etc/*.conf
> ```


---

**Q8.** `/etc` 디렉터리에서 파일명이 정확히 5글자인 파일만 출력하시오.

>  **힌트:** `?` 하나가 정확히 1글자를 의미한다. 5글자면 `?????` 처럼 5개 나열한다.

```bash
# 예제
ls /etc/????       # 4글자짜리
ls /etc/???.conf   # 3글자 이름 + .conf
```

> [!success]-  정답 보기
>
> ```bash
> ls /etc/?????
> ```


---

**Q9.** 현재 디렉터리에서 파일명이 `file`로 시작하고 뒤에 숫자 한 자리가 붙는 `.txt` 파일을 출력하시오. (예: `file0.txt` ~ `file9.txt`)

>  **힌트:** `[0-9]` 는 숫자 0~9 중 하나와 매칭된다.

```bash
# 예제
ls file[a-z].txt    # filea.txt ~ filez.txt
ls backup[ABC]      # backupA, backupB, backupC
```

> [!success]-  정답 보기
>
> ```bash
> ls file[0-9].txt
> ```


---

**Q10.** 중괄호 확장을 이용해 `/tmp` 아래에 `dir_2024`, `dir_2025`, `dir_2026` 디렉터리 3개를 한 명령으로 생성하시오.

>  **힌트:** `{값1,값2,값3}` 형태로 여러 값을 한 번에 확장할 수 있다.

```bash
# 예제
mkdir /tmp/{a,b,c}          # /tmp/a, /tmp/b, /tmp/c 동시 생성
touch /tmp/file{1,2,3}.txt  # 파일 3개 동시 생성
```

> [!success]-  정답 보기
>
> ```bash
> mkdir /tmp/{dir_2024,dir_2025,dir_2026}
>
> # 확인
> ls -ld /tmp/dir_202*
> ```


---

### 응용 (Level 2)

**Q11.** 다음 명령 연결 상황을 완성하시오.

1. `/backup` 디렉터리가 없으면 생성하고, 생성 성공 시 `"backup dir ready"` 출력.
2. `/etc/hosts` 파일이 없으면 `"hosts not found"` 출력.

>  **힌트:** `&&` = 앞 명령 성공 시 뒤 실행. `||` = 앞 명령 실패 시 뒤 실행.

```bash
# 예제
mkdir /tmp/test && echo "created"
ls /tmp/없는파일 || echo "not found"
```

> [!success]-  정답 보기
>
> ```bash
> # 1번
> mkdir /backup && echo "backup dir ready"
>
> # 2번
> ls /etc/hosts > /dev/null 2>&1 || echo "hosts not found"
> # 또는
> [ -f /etc/hosts ] || echo "hosts not found"
> ```


---

**Q12.** `/etc/passwd` 파일에서 `root`가 포함된 줄을 `root_info.txt`에 저장하시오. 이어서 `sshd`가 포함된 줄을 같은 파일에 **추가**하시오.

>  **힌트:** `>` 는 덮어쓰기, `>>` 는 이어쓰기.

```bash
# 예제
echo "line1" > output.txt    # 생성/덮어쓰기
echo "line2" >> output.txt   # 이어쓰기
```

> [!success]-  정답 보기
>
> ```bash
> grep "root" /etc/passwd > root_info.txt
> grep "sshd" /etc/passwd >> root_info.txt
>
> # 확인
> cat root_info.txt
> ```


---

**Q13.** `/etc/passwd` 파일에서 로그인 셸이 `/sbin/nologin`인 계정 수를 출력하시오.

>  **힌트:** `grep 패턴 파일 | wc -l` 조합으로 줄 수를 센다.

```bash
# 예제
grep "bash" /etc/passwd | wc -l
```

> [!success]-  정답 보기
>
> ```bash
> grep "/sbin/nologin" /etc/passwd | wc -l
> ```


---

## PART 3. expr & let

>  관련 문서: Shell Script - expr · let (산술 연산)

### 핵심 개념 복습

```text
expr
  형식    : expr 값1 연산자 값2
  특징    : 공백 필수 / * 는 \* 로 escape / 결과를 $()로 받아야 저장
  예      : result=$(expr 10 + 5)

let
  형식    : let 변수=수식
  특징    : 변수 앞 $ 불필요 / 공백 있으면 따옴표 필요 / 출력은 echo로 별도 실행
  예      : let x=5+3  /  let "y = x * 2"  /  let n++
```

---

### 기초 (Level 1)

**Q14.** `expr`을 사용해 `a=15`, `b=4`의 몫과 나머지를 각각 구해 출력하시오.

>  **힌트:** 나눗셈 `/` 은 정수 몫만 반환. 나머지는 `%`. `*` 는 `\*` 로 escape.

```bash
# 예제
expr 10 / 3     # 출력: 3 (몫)
expr 10 % 3     # 출력: 1 (나머지)
```

> [!success]-  정답 보기
>
> ```bash
> a=15
> b=4
>
> quotient=$(expr $a / $b)
> remainder=$(expr $a % $b)
>
> echo "몫: $quotient"
> echo "나머지: $remainder"
> # 출력:
> # 몫: 3
> # 나머지: 3
> ```


---

**Q15.** `let`을 사용해 변수 `count=0`에서 시작해 3번 증가시키고 최종 값을 출력하시오.

>  **힌트:** `let count++` 는 1 증가. `let count+=3` 은 3 한 번에 더하기.

```bash
# 예제
count=0
let count++
let count++
echo $count    # 출력: 2
```

> [!success]-  정답 보기
>
> ```bash
> count=0
> let count++
> let count++
> let count++
> echo $count
> # 출력: 3
>
> # 또는 한 번에
> count=0
> let count+=3
> echo $count
> # 출력: 3
> ```


---

### 응용 (Level 2)

**Q16.** `expr`을 사용해 섭씨 25도를 화씨로 변환하시오. 공식: `F = C * 9 / 5 + 32`

>  **힌트:** `expr`은 단계별로 계산한다. `*` 는 반드시 `\*` 로 escape해야 한다.

```bash
# 예제 (단계 계산)
a=7
b=3
c=$(expr $a \* $b)
d=$(expr $c + 1)
echo $d
```

> [!success]-  정답 보기
>
> ```bash
> C=25
> step1=$(expr $C \* 9)
> step2=$(expr $step1 / 5)
> F=$(expr $step2 + 32)
> echo "${C}°C = ${F}°F"
> # 출력: 25°C = 77°F
> ```


---

**Q17.** `let`을 사용해 1부터 5까지의 합을 구하는 스크립트를 작성하시오.

>  **힌트:** 변수를 하나씩 더하거나, `let "total = 1 + 2 + 3 + ..."` 처럼 한 번에 계산한다.

> [!success]-  정답 보기
>
> ```bash
> # 방법 1: 한 줄
> let "total = 1 + 2 + 3 + 4 + 5"
> echo $total
> # 출력: 15
>
> # 방법 2: 단계별
> total=0
> let total+=1
> let total+=2
> let total+=3
> let total+=4
> let total+=5
> echo $total
> ```


---

## PART 4. exit & test

>  관련 문서: Shell Script - exit 상태와 test 명령

### 핵심 개념 복습

```text
$?         : 직전 명령의 종료 코드 (0=성공, 1~255=실패)
exit 코드  : 0=성공 / 1=일반오류 / 2=문법오류 / 127=명령없음 / 130=Ctrl+C

test 형식  : test 조건식  또는  [ 조건식 ]
주의       : [ 조건 ] 안쪽에 공백 필수 → [ $x -gt 5 ] (O)  [$x -gt 5] (X)

숫자 비교  : -eq(==)  -ne(!=)  -gt(>)  -lt(<)  -ge(>=)  -le(<=)
문자열     : = (같음)  != (다름)  -z (빈문자열)  -n (비어있지않음)
파일       : -e (존재)  -f (일반파일)  -d (디렉터리)  -r (읽기가능)  -s (크기>0)
```

---

### 기초 (Level 1)

**Q18.** 다음 각 명령을 실행한 후 `$?`로 종료 코드를 확인하고 예상값을 적어보시오.

```bash
ls /etc/passwd
echo $?      # 예상값: ___

ls /없는경로
echo $?      # 예상값: ___

xyzabcnone
echo $?      # 예상값: ___
```

>  **힌트:** 성공=0, 일반 실패=1~2, 명령어를 찾을 수 없음=127

> [!success]-  정답 보기
>
> ```bash
> ls /etc/passwd    # 성공
> echo $?           # 0
>
> ls /없는경로      # 실패 (경로 없음)
> echo $?           # 2
>
> xyzabcnone        # 명령어 없음
> echo $?           # 127
> ```


---

**Q19.** `test` 명령으로 다음 조건을 확인하고 각각 `echo $?`로 결과를 확인하시오.

1. 변수 `n=10`이 5보다 큰지
2. 문자열 `"hello"`가 비어있지 않은지(`-n`)
3. `/etc/passwd`가 일반 파일인지(`-f`)

>  **힌트:** `test` 는 결과를 화면에 출력하지 않는다. `echo $?` 로 0(참) 또는 1(거짓)을 확인한다.

```bash
# 예제
[ 5 -gt 3 ]
echo $?          # 출력: 0 (참)

[ 5 -lt 3 ]
echo $?          # 출력: 1 (거짓)
```

> [!success]-  정답 보기
>
> ```bash
> # 1번
> n=10
> [ $n -gt 5 ]
> echo $?          # 0 (참, 10 > 5)
>
> # 2번
> str="hello"
> [ -n "$str" ]
> echo $?          # 0 (참, 비어있지 않음)
>
> # 3번
> [ -f /etc/passwd ]
> echo $?          # 0 (참, 일반 파일)
> ```


---

### 응용 (Level 2)

**Q20.** `/backup` 디렉터리가 존재하면 `"backup exists"`, 없으면 `mkdir /backup` 후 `"backup created"`를 출력하는 한 줄 명령을 `&&`, `||`를 이용해 작성하시오.

>  **힌트:** `[ -d 경로 ] && 참일때 || 거짓일때` 패턴.

```bash
# 예제 패턴
[ -d /tmp ] && echo "exists" || echo "not exists"
```

> [!success]-  정답 보기
>
> ```bash
> [ -d /backup ] && echo "backup exists" || (mkdir /backup && echo "backup created")
> ```


---

## PART 5. 조건문 (if)

>  관련 문서: Shell Script - 조건문 (if · case) · Shell Script - exit 상태와 test 명령

### 핵심 개념 복습

```bash
# 기본 형식
if [ 조건 ]; then
    명령
elif [ 조건2 ]; then
    명령2
else
    명령3
fi

# 산술 비교는 (( )) 사용 가능
if (( a > b )); then
    echo "a가 크다"
fi
```

---

### 기초 (Level 1)

**Q21.** 변수 `score=75`일 때, 60 이상이면 `"합격"`, 미만이면 `"불합격"`을 출력하시오.

>  **힌트:** `if-else-fi` 구조. 숫자 비교는 `-ge` (크거나 같다).

> [!success]-  정답 보기
>
> ```bash
> score=75
> if [ $score -ge 60 ]; then
>     echo "합격"
> else
>     echo "불합격"
> fi
> ```


---

**Q22.** 변수 `user`에 현재 로그인한 사용자(`$USER`)를 저장하고, `root`이면 `"관리자입니다"`, 아니면 `"일반 사용자입니다"`를 출력하시오.

>  **힌트:** 문자열 비교는 `=` 사용. 변수를 `"$user"` 처럼 겹따옴표로 감싸는 것이 안전하다.

> [!success]-  정답 보기
>
> ```bash
> user=$USER
> if [ "$user" = "root" ]; then
>     echo "관리자입니다"
> else
>     echo "일반 사용자입니다"
> fi
> ```


---

**Q23.** `/etc/hostname` 파일이 존재하면 내용을 출력하고, 없으면 `"파일이 없습니다"` 출력.

>  **힌트:** `-e` 는 파일/디렉터리 존재 여부, `-f` 는 일반 파일 여부.

> [!success]-  정답 보기
>
> ```bash
> if [ -f /etc/hostname ]; then
>     cat /etc/hostname
> else
>     echo "파일이 없습니다"
> fi
> ```


---

### 응용 (Level 2)

**Q24.** 다음 성적 등급 판정 스크립트를 완성하시오.

```text
90 이상 → A
80 이상 → B
70 이상 → C
60 이상 → D
60 미만 → F
```

>  **힌트:** `if-elif-elif-elif-else-fi` 구조. `elif`는 앞 조건이 모두 거짓일 때 검사한다.

> [!success]-  정답 보기
>
> ```bash
> score=85
>
> if [ $score -ge 90 ]; then
>     echo "등급: A"
> elif [ $score -ge 80 ]; then
>     echo "등급: B"
> elif [ $score -ge 70 ]; then
>     echo "등급: C"
> elif [ $score -ge 60 ]; then
>     echo "등급: D"
> else
>     echo "등급: F"
> fi
> ```


---

**Q25.** 인수 1개를 받아 그 경로가 일반 파일인지, 디렉터리인지, 존재하지 않는지를 판별하는 스크립트 `check_path.sh`를 작성하시오.

실행 예:

```bash
./check_path.sh /etc/passwd   # "/etc/passwd 는 일반 파일입니다"
./check_path.sh /etc          # "/etc 는 디렉터리입니다"
./check_path.sh /없는경로     # "/없는경로 는 존재하지 않습니다"
```

>  **힌트:** 스크립트에 전달된 첫 번째 인수는 `$1`로 참조한다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> target=$1
>
> if [ -f "$target" ]; then
>     echo "$target 는 일반 파일입니다"
> elif [ -d "$target" ]; then
>     echo "$target 는 디렉터리입니다"
> else
>     echo "$target 는 존재하지 않습니다"
> fi
> ```
>
> ```bash
> chmod +x check_path.sh
> ./check_path.sh /etc/passwd
> ./check_path.sh /etc
> ./check_path.sh /없는경로
> ```


---

**Q26.** 두 숫자를 변수 `a`, `b`에 저장하고, 둘 다 10 이상이면 `"모두 통과"`, 하나만 10 이상이면 `"하나만 통과"`, 둘 다 미만이면 `"모두 미달"`을 출력하시오.

>  **힌트:** `[ ] && [ ]` 형태로 AND 조건 결합. OR은 `[ ] || [ ]`.

> [!success]-  정답 보기
>
> ```bash
> a=12
> b=7
>
> if [ $a -ge 10 ] && [ $b -ge 10 ]; then
>     echo "모두 통과"
> elif [ $a -ge 10 ] || [ $b -ge 10 ]; then
>     echo "하나만 통과"
> else
>     echo "모두 미달"
> fi
> ```


---

## PART 6. 반복문 (for / while / until)

>  관련 문서: Shell Script - 반복문 (for · while · until)

### 핵심 개념 복습

```bash
# for 기본
for 변수 in 값1 값2 값3; do
    명령
done

# for 범위
for i in {1..10}; do echo $i; done

# for C스타일
for ((i=0; i<5; i++)); do echo $i; done

# while (조건 참인 동안 반복)
while [ 조건 ]; do
    명령
done

# until (조건 참이 될 때까지 반복)
until [ 조건 ]; do
    명령
done
```

---

### 기초 (Level 1)

**Q27.** `for` 문으로 `apple`, `banana`, `cherry`를 한 줄씩 출력하시오.

```bash
# 예제
for item in A B C; do
    echo $item
done
```

> [!success]-  정답 보기
>
> ```bash
> for item in apple banana cherry; do
>     echo $item
> done
> ```


---

**Q28.** `for` 문과 `{1..10}` 범위를 이용해 1부터 10까지 짝수만 출력하시오.

>  **힌트:** `{2..10..2}` 로 step을 주거나, `if (( i % 2 == 0 ))` 으로 판별하는 두 가지 방법이 있다.

> [!success]-  정답 보기
>
> ```bash
> # 방법 1: step 사용
> for i in {2..10..2}; do
>     echo $i
> done
>
> # 방법 2: if로 짝수 판별
> for i in {1..10}; do
>     if (( i % 2 == 0 )); then
>         echo $i
>     fi
> done
> ```


---

**Q29.** `while` 문을 사용해 변수 `n=1`부터 시작해 5가 될 때까지 출력하고, 매 반복마다 n을 1씩 증가시키시오.

>  **힌트:** 루프 안에서 `n=$((n+1))` 로 증가. 증가를 빠뜨리면 무한루프.

> [!success]-  정답 보기
>
> ```bash
> n=1
> while [ $n -le 5 ]; do
>     echo $n
>     n=$((n + 1))
> done
> ```


---

### 응용 (Level 2)

**Q30.** 다음 조건을 만족하는 스크립트 `user_create.sh`를 작성하시오.

- 사용자 목록 `users=("testA" "testB" "testC")`
- 각 사용자가 존재하지 않으면 `useradd`로 생성하고 `"testA 생성 완료"` 출력
- 이미 존재하면 `"testA 이미 존재"` 출력

>  **힌트:** `id 사용자명 &>/dev/null` 의 종료 코드로 존재 여부 판단 (0=존재).

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> users=("testA" "testB" "testC")
>
> for user in "${users[@]}"; do
>     if id "$user" &>/dev/null; then
>         echo "$user 이미 존재"
>     else
>         useradd "$user"
>         echo "$user 생성 완료"
>     fi
> done
> ```
>
> >  실습 후 `userdel -r testA testB testC` 로 정리할 것.


---

**Q31.** `until` 문으로 카운트다운 10→1→`"발사!"` 를 출력하는 스크립트를 작성하시오.

>  **힌트:** `while`과 조건 방향만 반대. `until [ $n -lt 1 ]` 처럼 종료 조건을 설정.

```bash
# until 구조 예제
n=3
until [ $n -lt 1 ]; do
    echo $n
    n=$((n - 1))
done
echo "종료"
```

> [!success]-  정답 보기
>
> ```bash
> n=10
> until [ $n -lt 1 ]; do
>     echo "$n"
>     n=$((n - 1))
> done
> echo "발사!"
> ```


---

**Q32.** 스크립트 `sum_loop.sh`를 작성하시오.

- 1부터 100까지의 합을 `for` C스타일 반복문으로 구한다.
- 결과를 `"1~100 합계: XXXX"` 형식으로 출력한다.

>  **힌트:** `for ((i=1; i<=100; i++))` 와 `total=$((total+i))` 를 조합한다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> total=0
>
> for ((i=1; i<=100; i++)); do
>     total=$((total + i))
> done
>
> echo "1~100 합계: $total"
> # 출력: 1~100 합계: 5050
> ```


---

## PART 7. 배열 (Array)

>  관련 문서: Shell Script - 배열(Array)과 RANDOM

### 핵심 개념 복습

```bash
선언       : arr=("값1" "값2" "값3")
접근       : ${arr[0]}  (인덱스는 0부터 시작)
전체 출력  : ${arr[@]}  또는  ${arr[*]}
길이       : ${#arr[@]}
인덱스 목록: ${!arr[@]}
요소 추가  : arr+=("새값")  또는  arr[4]="값"
요소 삭제  : unset 'arr[인덱스]'
RANDOM     : 0~32767 범위 임의 정수  /  RANDOM % N → 0~N-1 범위
```

---

### 기초 (Level 1)

**Q33.** 배열 `colors=("red" "green" "blue" "yellow")`를 선언하고 다음을 각각 출력하시오.

1. 세 번째 요소 (`blue`)
2. 전체 요소
3. 배열 길이
4. 인덱스 목록

>  **힌트:** 인덱스는 0부터 시작하므로 세 번째 요소는 인덱스 2이다.

> [!success]-  정답 보기
>
> ```bash
> colors=("red" "green" "blue" "yellow")
>
> echo ${colors[2]}       # blue
> echo ${colors[@]}       # red green blue yellow
> echo ${#colors[@]}      # 4
> echo ${!colors[@]}      # 0 1 2 3
> ```


---

**Q34.** 위 `colors` 배열에서 `"green"` 요소를 `"orange"`로 수정하고, `"purple"`을 배열 끝에 추가한 뒤 전체를 출력하시오.

>  **힌트:** 수정: `arr[인덱스]=새값`, 추가: `arr+=("값")`.

> [!success]-  정답 보기
>
> ```bash
> colors=("red" "green" "blue" "yellow")
>
> colors[1]="orange"       # green → orange 수정
> colors+=("purple")       # purple 추가
>
> echo ${colors[@]}
> # 출력: red orange blue yellow purple
> ```


---

### 응용 (Level 2)

**Q35.** 배열 `scores=(45 82 60 37 91 55 78)`에서 60점 이상인 점수만 출력하는 스크립트를 작성하시오.

>  **힌트:** `for score in "${scores[@]}"` 로 순회하면서 `if (( score >= 60 ))` 로 판별.

> [!success]-  정답 보기
>
> ```bash
> scores=(45 82 60 37 91 55 78)
>
> for score in "${scores[@]}"; do
>     if (( score >= 60 )); then
>         echo $score
>     fi
> done
> # 출력:
> # 82
> # 60
> # 91
> # 78
> ```


---

**Q36.** `$RANDOM`을 사용해 0~99 사이의 정수 20개를 배열에 저장하고, 50 이상인 것과 미만인 것의 개수를 각각 출력하시오.

>  **힌트:** `RANDOM % 100` 으로 0~99 범위 제한. 카운터 두 개를 두고 `(( count++ ))` 로 증가.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> nums=()
> high=0
> low=0
>
> for ((i=0; i<20; i++)); do
>     nums[i]=$(( RANDOM % 100 ))
> done
>
> echo "생성된 수: ${nums[@]}"
> echo
>
> for val in "${nums[@]}"; do
>     if (( val >= 50 )); then
>         (( high++ ))
>     else
>         (( low++ ))
>     fi
> done
>
> echo "50 이상: ${high}개"
> echo "50 미만: ${low}개"
> ```


---

## PART 8. 종합 스크립트 문제

>  관련 문서: Shell Script - 통합 정리 · Shell Script - 명령어 퀵 레퍼런스

> 이 파트는 여러 개념을 결합하는 문제이다. 스크립트 파일로 작성하고 실행해보는 것을 목표로 한다.

---

**Q37. 로그 파일 자동 정리 스크립트**

`clean_logs.sh`를 작성하시오.

- `logdir=/var/log`에서 `.log`로 끝나는 파일을 배열로 저장한다.
- 각 파일을 순회하며 파일 크기가 0이면 `"파일명: 빈 파일 스킵"` 출력.
- 0이 아니면 `"파일명: 처리 대상"` 을 출력한다.

>  **힌트:** `-s` 옵션이 크기 0 초과(내용 있음)를 의미한다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> logdir=/var/log
> files=( $(ls "$logdir"/*.log 2>/dev/null) )
>
> if [ ${#files[@]} -eq 0 ]; then
>     echo ".log 파일이 없습니다."
>     exit 0
> fi
>
> for f in "${files[@]}"; do
>     if [ -s "$f" ]; then
>         echo "$f: 처리 대상"
>     else
>         echo "$f: 빈 파일 스킵"
>     fi
> done
> ```


---

**Q38. 구구단 출력 스크립트**

`gugudan.sh`를 작성하시오.

- 인수 1개를 받아 해당 단의 구구단을 출력한다.
- 인수가 없으면 `"사용법: ./gugudan.sh 단수"` 출력 후 종료.
- 단수가 1~9 범위를 벗어나면 `"1~9 사이 숫자를 입력하세요"` 출력.

실행 예:

```bash
./gugudan.sh 3
# 3 x 1 = 3
# ...
# 3 x 9 = 27
```

>  **힌트:** 인수 없음: `[ -z "$1" ]`. 범위 확인: `(( $1 < 1 || $1 > 9 ))`.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> if [ -z "$1" ]; then
>     echo "사용법: ./gugudan.sh 단수"
>     exit 1
> fi
>
> if (( $1 < 1 || $1 > 9 )); then
>     echo "1~9 사이 숫자를 입력하세요"
>     exit 1
> fi
>
> for i in {1..9}; do
>     echo "$1 x $i = $(( $1 * i ))"
> done
> ```


---

**Q39. 시스템 정보 요약 스크립트**

`sysinfo.sh`를 작성하시오. 다음 항목을 출력한다.

```text
=== 시스템 정보 ===
날짜/시간  : 2026-07-29 14:30:00
호스트명   : Server-A
현재 사용자: root
홈 디렉터리: /root
사용 중인 셸: /bin/bash
/etc 파일 수: 213개
```

>  **힌트:** `date +"%F %T"`, `hostname`, `$USER`, `$HOME`, `$SHELL`, `ls /etc | wc -l` 활용.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> dt=$(date +"%F %T")
> host=$(hostname)
> etc_count=$(ls /etc | wc -l)
>
> echo "=== 시스템 정보 ==="
> echo "날짜/시간   : $dt"
> echo "호스트명    : $host"
> echo "현재 사용자 : $USER"
> echo "홈 디렉터리 : $HOME"
> echo "사용 중인 셸: $SHELL"
> echo "/etc 파일 수: ${etc_count}개"
> ```


---

**Q40. 다중 스크립트 동시 실행 & 결과 수집**

> 이 문제는 백그라운드 실행(`&`), `wait`, 종료 코드 수집을 결합한 실무형 문제이다.

다음 세 가지 점검 스크립트를 **동시에 실행**하고, 모든 작업이 끝난 후 각각의 성공/실패 여부를 보고하는 `run_all.sh`를 작성하시오.

**점검 대상 스크립트 3개 (먼저 작성):**

```bash
# check_disk.sh
#!/bin/bash
use=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if (( use >= 90 )); then
    echo "[WARN] 디스크 사용률 ${use}% - 위험"
    exit 1
else
    echo "[OK] 디스크 사용률 ${use}%"
    exit 0
fi
```

```bash
# check_mem.sh
#!/bin/bash
free_mb=$(free -m | awk 'NR==2 {print $4}')
if (( free_mb < 100 )); then
    echo "[WARN] 가용 메모리 ${free_mb}MB - 부족"
    exit 1
else
    echo "[OK] 가용 메모리 ${free_mb}MB"
    exit 0
fi
```

```bash
# check_sshd.sh
#!/bin/bash
if systemctl is-active --quiet sshd; then
    echo "[OK] sshd 서비스 실행 중"
    exit 0
else
    echo "[WARN] sshd 서비스 중지됨"
    exit 1
fi
```

**조건:**

- 세 스크립트를 **동시에(백그라운드)** 실행한다.
- 세 스크립트가 **모두 끝날 때까지 기다린다.**
- 각 스크립트의 종료 코드를 수집해 최종 결과를 출력한다.
- 하나라도 실패(exit 1)이면 `"[RESULT] 점검 실패 항목 있음"`, 모두 성공이면 `"[RESULT] 모든 점검 통과"` 출력.

>  **힌트:**
> 
> - `스크립트 &` : 백그라운드 실행
> - `$!` : 직전에 백그라운드로 실행한 프로세스의 PID
> - `wait PID` : 해당 PID가 끝날 때까지 대기하고 종료 코드를 반환
> - PID를 배열에 모아두고 나중에 `wait`으로 순서대로 수집하는 것이 핵심

```bash
# 예제 - 백그라운드 실행 & wait 패턴
sleep 2 &
pid1=$!

sleep 3 &
pid2=$!

wait $pid1    # sleep 2가 끝날 때까지 대기
echo "pid1 종료: $?"

wait $pid2
echo "pid2 종료: $?"
```

> [!success]-  정답 보기
>
> **준비: 세 스크립트 실행 권한 부여**
>
> ```bash
> chmod +x check_disk.sh check_mem.sh check_sshd.sh
> ```
>
> **run_all.sh**
>
> ```bash
> #!/bin/bash
>
> SCRIPT_DIR=$(dirname "$0")   # 스크립트가 있는 디렉터리 기준으로 경로 설정
>
> echo "=== 병렬 점검 시작 ==="
> echo
>
> # 1. 세 스크립트를 동시에 백그라운드로 실행 & PID 수집
> "$SCRIPT_DIR/check_disk.sh" &
> pid_disk=$!
>
> "$SCRIPT_DIR/check_mem.sh" &
> pid_mem=$!
>
> "$SCRIPT_DIR/check_sshd.sh" &
> pid_sshd=$!
>
> # 2. 세 프로세스가 모두 끝날 때까지 대기 & 종료 코드 수집
> wait $pid_disk
> rc_disk=$?
>
> wait $pid_mem
> rc_mem=$?
>
> wait $pid_sshd
> rc_sshd=$?
>
> # 3. 결과 집계
> echo
> echo "=== 점검 결과 ==="
> [ $rc_disk  -eq 0 ] && echo "  디스크  : PASS" || echo "  디스크  : FAIL"
> [ $rc_mem   -eq 0 ] && echo "  메모리  : PASS" || echo "  메모리  : FAIL"
> [ $rc_sshd  -eq 0 ] && echo "  SSH 데몬: PASS" || echo "  SSH 데몬: FAIL"
> echo
>
> # 4. 종합 판정
> if [ $rc_disk -eq 0 ] && [ $rc_mem -eq 0 ] && [ $rc_sshd -eq 0 ]; then
>     echo "[RESULT] 모든 점검 통과"
>     exit 0
> else
>     echo "[RESULT] 점검 실패 항목 있음"
>     exit 1
> fi
> ```
>
> **실행:**
>
> ```bash
> chmod +x run_all.sh
> ./run_all.sh
> ```
>
> **출력 예시:**
>
> ```
> === 병렬 점검 시작 ===
>
> [OK] 디스크 사용률 34%
> [OK] 가용 메모리 512MB
> [OK] sshd 서비스 실행 중
>
> === 점검 결과 ===
>   디스크  : PASS
>   메모리  : PASS
>   SSH 데몬: PASS
>
> [RESULT] 모든 점검 통과
> ```
>
> >  **핵심 동작 원리**
> >
> > - `&` 없이 실행하면 스크립트 3개가 **순서대로(직렬)** 실행된다.
> > - `&` 를 붙이면 즉시 다음 줄로 넘어가며 **병렬 실행**된다.
> > - `wait PID` 는 해당 프로세스가 끝날 때까지 기다리고, **그 종료 코드를 `$?` 로 반환**한다.
> > - PID를 먼저 변수에 저장해두지 않으면 `$!`가 마지막 PID로 덮어써지므로 반드시 **실행 직후 즉시 저장**해야 한다.


---

## PART 9. 위치 매개변수 (Positional Parameters)

>  관련 문서: Shell Script - 위치 매개변수 (Positional Parameters)

### 핵심 개념 복습

```text
$0         : 실행된 스크립트 이름 (이름만 필요하면 basename "$0")
$1 ~ $9    : 1~9번째 인자  /  ${10} 이상은 중괄호 필수 ($10 은 $1 + 문자 0)
$#         : 인자 개수 ($0 은 포함되지 않음)
"$@"       : 전체 인자를 개별 값으로 전개 (다른 명령에 넘길 때 표준)
"$*"       : 전체 인자를 하나의 문자열로 결합 (메시지 출력용)
$? $$ $!   : 직전 종료 코드 / 현재 스크립트 PID / 마지막 백그라운드 PID
기본값     : dest="${2:-/backup}"   (2번 인자가 없으면 /backup)
shift      : $2→$1 로 밀고 $# 을 1 감소
검증 순서  : 개수 → 형식 → 존재  →  실패 시 Usage 출력 + exit 1
```

---

### 기초 (Level 1)

**Q41.** 스크립트 `args_info.sh`를 작성해 스크립트 이름, 1~3번째 인자, 인자 개수, 개별 전개, 문자열 결합을 각각 출력하시오.

실행 예:

```bash
./args_info.sh apple banana cherry
```

>  **힌트:** `$0`, `$1~$3`, `$#`, `"$@"`, `"$*"` 를 순서대로 출력한다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> echo "스크립트 이름 : $0"
> echo "첫 번째 값   : $1"
> echo "두 번째 값   : $2"
> echo "세 번째 값   : $3"
> echo "인자 개수    : $#"
> echo "개별 전개    : $@"
> echo "문자열 결합  : $*"
> ```
>
> ```bash
> chmod +x args_info.sh
> ./args_info.sh apple banana cherry
> # 스크립트 이름 : ./args_info.sh
> # 인자 개수    : 3
> ```
>
> > `./args_info.sh "kim lee" park` 로도 실행해 볼 것. `"$@"` 는 인자 2개를 유지하고 `"$*"` 는 하나의 문자열로 합친다.


---

**Q42.** 인수 2개를 받아 더 큰 수를 출력하는 `max2.sh`를 작성하시오. 인수 개수가 2개가 아니면 사용법을 출력하고 `exit 1` 로 종료해야 한다.

>  **힌트:** `[ $# -ne 2 ]` 로 개수 검증. Usage 메시지에는 하드코딩된 이름 대신 `$0` 을 쓴다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> if [ $# -ne 2 ]
> then
>     echo "Usage : $0 <number1> <number2>"
>     exit 1
> fi
>
> if [ "$1" -gt "$2" ]
> then
>     echo "큰 수 : $1"
> else
>     echo "큰 수 : $2"
> fi
> ```
>
> ```bash
> ./max2.sh 100          # Usage : ./max2.sh <number1> <number2>
> echo $?                # 1
> ./max2.sh 100 200      # 큰 수 : 200
> ```


---

### 응용 (Level 2)

**Q43.** 인수로 받은 **모든 정수의 합**을 출력하는 `sum_args.sh`를 작성하시오.

- 인수가 하나도 없으면 사용법 출력 후 `exit 1`.
- 인수 중 정수가 아닌 값이 하나라도 있으면 그 값을 알려주고 `exit 1`.
- 검증을 모두 통과하면 인자 개수와 총합을 출력한다.

실행 예:

```bash
./sum_args.sh 1 2 3 4 5     # 인자 개수 : 5 / 인자 총합 : 15
./sum_args.sh {1..10}       # 총합 55
./sum_args.sh 10 20 A       # 정수가 아닌 값 : A
```

>  **힌트:** `$1`, `$2` 만 검사하면 `10 20 A` 가 통과된다. `for arg in "$@"` 로 **전체를 순회하며** ` "$arg" =~ ^-?[0-9]+$ ` 로 검증해야 한다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> if (( $# == 0 ))
> then
>     echo "Usage : $0 <int1> <int2> ..."
>     exit 1
> fi
>
> # ① 전체 인자 형식 검증 (앞의 2개만 보면 안 된다)
> for arg in "$@"
> do
>     if !  "$arg" =~ ^-?[0-9]+$ 
>     then
>         echo "Usage : $0 <int1> <int2> ..."
>         echo "정수가 아닌 값 : $arg"
>         exit 1
>     fi
> done
>
> # ② 합계 계산
> sum=0
> for num in "$@"
> do
>     sum=$(( sum + num ))
> done
>
> echo "인자 개수 : $#"
> echo "인자 총합 : $sum"
> ```
>
> > ` ` 와 `=~` 는 Bash 확장 문법이므로 `sh sum_args.sh` 로 실행하면 동작하지 않는다. 반드시 `#!/bin/bash` + `./sum_args.sh` 로 실행한다.


---

**Q44.** 다음 두 가지를 확인하는 `arg_trap.sh`를 작성하시오.

1. 12개의 인수를 받아 `1번`, `10번`, `12번` 인자를 각각 출력한다.
2. 2번 인자가 없으면 `/backup` 을 기본값으로 사용해 `"저장 경로 : ..."` 를 출력한다.

실행 예:

```bash
./arg_trap.sh a b c d e f g h i j k l
./arg_trap.sh /var/log          # 저장 경로 : /backup
```

>  **힌트:** `$10` 은 10번째 인자가 아니라 `$1` 뒤에 문자 `0` 이 붙은 값으로 해석된다. 기본값은 `${2:-기본값}`.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> echo "1번  : $1"
> echo "10번 : ${10}"        # 중괄호 필수
> echo "12번 : ${12}"
> echo "잘못된 표기 \$10 : $10"   # $1 + 문자 0 → "a0"
>
> dest="${2:-/backup}"       # 2번 인자가 없으면 기본값 사용
> echo "저장 경로 : $dest"
> ```
>
> ```bash
> ./arg_trap.sh a b c d e f g h i j k l
> # 1번  : a
> # 10번 : j
> # 12번 : l
> # 잘못된 표기 $10 : a0
> ```


---

**Q45.** `shift` 를 사용해 첫 번째 인자를 **동작(action)** 으로 소비하고, 나머지 인자를 **대상 목록**으로 처리하는 `do_action.sh`를 작성하시오.

- 인자가 2개 미만이면 `"Usage : $0 <action> <target1> [target2 ...]"` 출력 후 `exit 1`.
- 남은 인자 개수를 출력한 뒤, 각 대상에 대해 `"<action> 대상 : <target>"` 을 출력한다.

실행 예:

```bash
./do_action.sh backup /etc /var/log /home
# 동작 : backup / 대상 3개
# backup 대상 : /etc
# ...
```

>  **힌트:** `action="$1"; shift` 이후 `$#` 이 1 줄어들고 `"$@"` 에는 대상만 남는다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> if [ $# -lt 2 ]
> then
>     echo "Usage : $0 <action> <target1> [target2 ...]"
>     exit 1
> fi
>
> action="$1"
> shift                       # 첫 인자를 소비 → $2가 $1로 밀려온다
>
> echo "동작 : $action / 대상 ${#}개"
>
> for target in "$@"
> do
>     echo "$action 대상 : $target"
> done
>
> # shift 방식으로 순차 처리하는 대안
> # while [ $# -gt 0 ]
> # do
> #     echo "$action 대상 : $1"
> #     shift
> # done
> ```
>
> > `shift` 로 원본 인자가 사라지므로 목록을 보존해야 하면 `targets=("$@")` 로 배열에 먼저 옮겨 담는다.


---

## PART 10. case 문 & read 입력

>  관련 문서: Shell Script - 조건문 (if · case)

### 핵심 개념 복습

```bash
case "$변수" in                 # 변수는 반드시 겹따옴표로 감쌀 것
    패턴1)        명령1 ;;      # ;; = 분기 종료 후 esac으로 이동
    패턴2|패턴3)  명령2 ;;      # | 로 여러 패턴 묶기 (OR)
    *.log|*.txt)  명령3 ;;      # 글롭 패턴도 사용 가능
    *)            기본실행문 ;; # else 역할
esac

shopt -s nocasematch            # 대/소문자 구분 없이 패턴 비교
read -p "메시지 : " 변수         # 안내 메시지 + 입력
read -s -p "비밀번호 : " pw      # 입력 내용 숨김 (직후 echo로 줄바꿈)
read -t 5 -p "입력 : " 변수      # 5초 제한 (종료 코드로 시간 초과 판단)
```

---

### 기초 (Level 1)

**Q46.** `1`, `2`, `3` 중 하나를 입력받아 각각 다른 메시지를 출력하고, 그 외 입력은 `"잘못된 입력입니다"` 를 출력하는 스크립트를 `case` 문으로 작성하시오.

>  **힌트:** `*)` 분기가 else 역할을 한다. `esac` 으로 닫는다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> read -p "정수 입력(1/2/3) : " num
>
> case "$num" in
>     1) echo "1번 선택) cmd 1 running" ;;
>     2) echo "2번 선택) cmd 2 running" ;;
>     3) echo "3번 선택) cmd 3 running" ;;
>     *) echo "잘못된 입력입니다." ;;
> esac
> ```


---

**Q47.** `y/n` 을 입력받아 진행/종료를 분기하는 스크립트를 작성하시오. 단 `y`, `Y`, `yes`, `YES` 를 모두 같은 값으로 처리해야 하며, 10초 안에 입력이 없으면 `"입력 시간 초과"` 를 출력하고 `exit 1` 하시오.

>  **힌트:** 대소문자 무시는 `shopt -s nocasematch`, 시간 제한은 `read -t 10` 이며 시간 초과는 `read` 의 종료 코드로 판단한다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> shopt -s nocasematch                 # 대/소문자 구분하지 않음
>
> if ! read -t 10 -p "계속 진행하시겠습니까? (y/n) : " answer
> then
>     echo
>     echo "입력 시간 초과로 종료합니다."
>     exit 1
> fi
>
> case "$answer" in
>     y|yes) echo "YES 선택 - 계속 진행합니다." ;;
>     n|no)  echo "NO 선택 - 종료합니다." ; exit 0 ;;
>     *)     echo "잘못 입력했습니다. y 또는 n 을 입력하세요" ; exit 1 ;;
> esac
> ```
>
> > `read -s` 로 비밀번호를 받을 때는 줄바꿈이 되지 않으므로 직후에 빈 `echo` 를 넣어야 다음 출력이 같은 줄에 붙지 않는다.


---

### 응용 (Level 2)

**Q48.** 파일명을 인수로 받아 확장자에 따라 알맞은 압축 해제 명령을 실행하는 `extract.sh`를 작성하시오.

```text
*.tar.gz, *.tgz → tar -xzf
*.tar.bz2       → tar -xjf
*.tar           → tar -xf
*.gz            → gunzip
*.zip           → unzip
그 외           → "지원하지 않는 형식" 출력 후 exit 1
```

>  **힌트:** `case` 의 패턴에는 글롭(`*`)을 그대로 쓸 수 있다. `*.tar.gz` 를 `*.gz` 보다 **먼저** 배치해야 한다(위에서부터 첫 일치 분기만 실행).

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> if [ $# -ne 1 ]
> then
>     echo "Usage : $0 <archive-file>"
>     exit 1
> fi
>
> file="$1"
>
> if [ ! -f "$file" ]
> then
>     echo "파일이 없습니다 : $file"
>     exit 1
> fi
>
> case "$file" in
>     *.tar.gz|*.tgz) tar -xzf "$file" ;;
>     *.tar.bz2)      tar -xjf "$file" ;;
>     *.tar)          tar -xf  "$file" ;;
>     *.gz)           gunzip   "$file" ;;
>     *.zip)          unzip    "$file" ;;
>     *)
>         echo "지원하지 않는 형식입니다 : $file"
>         exit 1
>         ;;
> esac
>
> if [ $? -eq 0 ]
> then
>     echo "압축 해제 완료 : $file"
> else
>     echo "압축 해제 실패 : $file"
>     exit 1
> fi
> ```
>
> > 순서를 바꿔 `*.gz` 를 위에 두면 `data.tar.gz` 가 `gunzip` 분기로 잡혀 `.tar` 만 남는다. `case` 는 **위에서부터 첫 일치 분기만** 실행한다는 점이 핵심이다.


---

**Q49.** 위치 매개변수와 `case` 를 결합해 서비스를 제어하는 `svc.sh`를 작성하시오.

- 사용법: `./svc.sh {start|stop|restart|status} <service-name>`
- 인수가 2개가 아니면 사용법 출력 후 `exit 1`.
- `start|stop|restart` 는 `systemctl` 로 실행하고, 실행 후 `systemctl is-active` 결과를 출력한다.
- `status` 는 페이저 없이 상태를 출력한다.
- 정의되지 않은 동작은 `"Unknown Action : ..."` 출력 후 `exit 1`.

>  **힌트:** `systemctl status` 는 서비스가 정지 상태일 때 종료 코드 `3` 을 반환하므로 `$?` 로 성공/실패를 판단하면 안 된다. 활성 여부는 `systemctl is-active` 로 확인한다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> if [ $# -ne 2 ]
> then
>     echo "Usage : $0 {start|stop|restart|status} <service-name>"
>     exit 1
> fi
>
> action="$1"
> service="$2"
>
> case "$action" in
>     start|stop|restart)
>         systemctl "$action" "$service" || { echo "$service $action 실패"; exit 1; }
>         echo "$service $action 완료 / 상태 : $(systemctl is-active "$service")"
>         ;;
>     status)
>         systemctl status "$service" --no-pager      # 종료 코드 3 → $? 판정 금지
>         ;;
>     *)
>         echo "Unknown Action : $action"
>         echo "Usage : $0 {start|stop|restart|status} <service-name>"
>         exit 1
>         ;;
> esac
> ```
>
> ```bash
> ./svc.sh                        # Usage 출력 후 종료
> ./svc.sh run httpd              # Unknown Action : run
> ./svc.sh restart sshd           # sshd restart 완료 / 상태 : active
> ```


---

## PART 11. break · continue · 폴링(polling)

>  관련 문서: Shell Script - 반복문 (for · while · until)

### 핵심 개념 복습

```text
break      : 반복문 자체를 즉시 종료 (스크립트 종료가 아님)
break 2    : 바깥쪽 반복문까지 2단계 종료 (중첩 루프)
continue   : 이번 회차의 남은 명령만 건너뛰고 다음 회차로
until+sleep: 조건이 참이 될 때까지 일정 간격으로 재확인 (폴링)
주의       : {1..$N} 은 확장되지 않는다 → for (( i=1; i<=N; i++ )) 또는 seq
```

---

### 기초 (Level 1)

**Q50.** 1부터 20까지 순회하며 **처음 만나는 7의 배수**를 출력한 뒤 반복을 즉시 종료하시오.

>  **힌트:** `(( num % 7 == 0 ))` 로 판별하고 `break` 로 빠져나온다.

> [!success]-  정답 보기
>
> ```bash
> for num in {1..20}
> do
>     if (( num % 7 == 0 ))
>     then
>         echo "첫 7의 배수 : $num"
>         break
>     fi
> done
> # 출력: 첫 7의 배수 : 7
> ```


---

### 응용 (Level 2)

**Q51.** 2단부터 9단까지 구구단을 출력하되, **곱한 값이 50을 넘는 순간 전체 반복을 모두 종료**하시오. 또한 결과가 홀수인 항목은 출력하지 않고 건너뛰시오.

>  **힌트:** 중첩 반복문에서 `break` 는 **가장 안쪽 루프만** 종료한다. 바깥까지 종료하려면 `break 2`. 건너뛰기는 `continue`.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> for dan in {2..9}
> do
>     for i in {1..9}
>     do
>         result=$(( dan * i ))
>
>         if (( result > 50 ))
>         then
>             echo "결과 ${result} > 50 → 전체 종료"
>             break 2                     # 안쪽·바깥쪽 루프 모두 종료
>         fi
>
>         if (( result % 2 != 0 ))
>         then
>             continue                    # 홀수는 출력하지 않고 다음 회차로
>         fi
>
>         echo "$dan x $i = $result"
>     done
> done
> ```
>
> > `break` 만 쓰면 안쪽 루프만 끝나고 다음 단으로 계속 진행된다. 상태 플래그 변수를 두고 바깥 조건에서 검사하는 방식도 대안이다.


---

**Q52.** `/tmp/ready.flag` 파일이 생성될 때까지 3초 간격으로 대기하는 폴링 스크립트를 작성하시오. 단 **30초를 초과하면** `"타임아웃"` 을 출력하고 `exit 1` 로 종료해야 한다.

>  **힌트:** `until [ -f "$file" ]` 에 경과 시간 카운터를 함께 두고, 루프 안에 `sleep 3` 을 반드시 넣어야 CPU 과점유를 막을 수 있다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> file="/tmp/ready.flag"
> waited=0
> timeout=30
> interval=3
>
> echo "$file 생성 대기 (최대 ${timeout}초)"
>
> until [ -f "$file" ]
> do
>     if (( waited >= timeout ))
>     then
>         echo "타임아웃 : ${timeout}초 내에 $file 이 생성되지 않았습니다."
>         exit 1
>     fi
>
>     echo "대기중... (${waited}초 경과)"
>     sleep "$interval"
>     waited=$(( waited + interval ))     # 증가 누락 시 무한 루프
> done
>
> echo "$file 생성 확인 - 후속 작업을 진행합니다."
> exit 0
> ```
>
> ```bash
> # 다른 터미널에서 실행해 조건을 만족시켜 볼 것
> touch /tmp/ready.flag
> ```


---

**Q53.** 아이디(`user1`)와 비밀번호(`admin1234`)를 **최대 3회까지** 확인하는 로그인 스크립트를 작성하시오. 성공 시 `"로그인 성공"` 후 `exit 0`, 3회 모두 실패하면 `"3회 실패하여 종료합니다"` 후 `exit 1`.

>  **힌트:** 카운터 변수 + `while [ "$count" -le 3 ]`. 비밀번호는 `read -s` 로 받고 직후 `echo` 로 줄바꿈한다. 카운터 증가를 빠뜨리면 무한 루프.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> id="user1"
> pw="admin1234"
> count=1
>
> while [ "$count" -le 3 ]
> do
>     read -p "아이디 : " input_id
>     read -s -p "비밀번호 : " input_pw
>     echo                                     # -s 는 줄바꿈을 하지 않는다
>
>     if [ "$input_id" = "$id" ] && [ "$input_pw" = "$pw" ]
>     then
>         echo "로그인 성공"
>         exit 0
>     else
>         echo "로그인 실패 ($count/3)"
>     fi
>
>     count=$(( count + 1 ))                   # 누락 시 무한 루프
> done
>
> echo "3회 실패하여 종료합니다."
> exit 1
> ```
>
> > `break` 는 while 루프만 벗어나 이후 코드가 계속 실행되고, `exit 0` 은 스크립트 자체를 종료한다. 설계 의도에 맞게 선택한다.


---

## PART 12. 배열 심화

>  관련 문서: Shell Script - 배열(Array)과 RANDOM

### 핵심 개념 복습

```text
슬라이싱   : ${arr[@]:1:3}     (1번부터 3개)
마지막 요소: ${arr[-1]}        (Bash 4.3+)
요소 개수  : ${#arr[@]}        /  ${#arr[0]} 은 "0번 요소의 문자 길이"
인덱스 목록: ${!arr[@]}        (희소 배열 순회의 표준)
재정렬     : arr=("${arr[@]}") (unset으로 생긴 인덱스 구멍 제거)
연관 배열  : declare -A map    (Bash 4.0+, 선언 필수)
구조 확인  : declare -p arr    (인덱스+값 한 번에 진단)
```

---

### 응용 (Level 2)

**Q54.** 배열 `name=("kim" "lee" "park" "ryu")` 에서 인덱스 2를 삭제한 뒤 다음을 확인하시오.

1. 요소 개수와 인덱스 목록
2. `for ((i=0; i<${#name[@]}; i++))` 로 순회했을 때의 문제점
3. 문제를 해결하는 두 가지 방법

>  **힌트:** `unset` 은 개수만 줄이고 뒤 요소를 앞으로 끌어당기지 않는다(희소 배열).

> [!success]-  정답 보기
>
> ```bash
> name=("kim" "lee" "park" "ryu")
> unset 'name[2]'                 # 대괄호가 글롭으로 해석되지 않도록 작은따옴표 사용
>
> echo "${#name[@]}"              # 3   (개수는 줄었지만)
> echo "${!name[@]}"              # 0 1 3  ← 인덱스 2가 비어 있다
>
> # ② 문제 : 인덱스가 연속이라는 전제가 깨져 빈 값 출력 + 마지막 요소 누락
> for (( i=0; i<${#name[@]}; i++ )); do echo "$i : ${name[i]}"; done
> # 0 : kim / 1 : lee / 2 :        ← ryu 누락
>
> # ③ 해결 1) 인덱스 목록 기준 순회 (희소 배열에 안전)
> for i in "${!name[@]}"; do echo "$i : ${name[i]}"; done
> # 0 : kim / 1 : lee / 3 : ryu
>
> # ③ 해결 2) 배열 재생성으로 인덱스 재정렬
> name=("${name[@]}")
> echo "${!name[@]}"              # 0 1 2
> declare -p name                 # 구조 최종 확인
> ```


---

**Q55.** 연관 배열로 서비스와 포트를 매핑해 다음을 처리하시오.

```text
sshd:22, httpd:80, https:443, vsftpd:21
```

1. 전체 `서비스 = 포트` 목록을 출력한다.
2. `httpd` 의 포트만 조회해 출력한다.
3. 등록된 서비스 개수를 출력한다.
4. 사용자가 입력한 서비스명이 등록되어 있는지 확인해 없으면 `"등록되지 않은 서비스"` 를 출력한다.

>  **힌트:** 연관 배열은 `declare -A` 선언이 **필수**다. 키 목록은 `"${!map[@]}"`, 키 존재 확인은 `[ -v map[키] ]` 또는 `[ -n "${map[$key]}" ]`.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> declare -A port                          # 선언 없으면 인덱스 배열로 취급되어 값이 덮어써진다
> port[sshd]=22
> port[httpd]=80
> port[https]=443
> port[vsftpd]=21
>
> # ① 전체 목록
> for key in "${!port[@]}"
> do
>     echo "$key = ${port[$key]}"
> done
>
> # ② 특정 키 조회
> echo "httpd 포트 : ${port[httpd]}"
>
> # ③ 등록 개수
> echo "등록 서비스 : ${#port[@]}개"
>
> # ④ 키 존재 확인
> read -p "조회할 서비스명 : " svc
>
> if [ -v port[$svc] ]                     # Bash 4.2+ : 키 존재 여부만 판단
> then
>     echo "$svc 포트 : ${port[$svc]}"
> else
>     echo "등록되지 않은 서비스입니다 : $svc"
>     exit 1
> fi
> ```
>
> > 연관 배열은 **키 순서가 보장되지 않는다.** 정렬 출력이 필요하면 `for key in $(echo "${!port[@]}" | tr ' ' '\n' | sort)` 처럼 별도로 정렬한다.


---

**Q56.** `$RANDOM` 으로 30~100 사이의 점수 10개를 배열에 저장한 뒤 다음을 출력하시오.

1. 생성된 전체 점수와 요소 개수
2. 최고점과 최저점
3. 합계와 평균(정수) — 소수점 둘째 자리 평균도 함께 출력
4. 인덱스 1번부터 3개만 슬라이싱해서 출력
5. 마지막 요소

>  **힌트:** 범위식은 `$(( RANDOM % (max - min + 1) + min ))`. 최대·최소는 첫 요소로 초기화한 뒤 순회 비교한다. 소수점 계산은 `echo "scale=2; $sum/$count" | bc`.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> scores=()
>
> for (( i=0; i<10; i++ ))
> do
>     scores[i]=$(( RANDOM % 71 + 30 ))          # 30 ~ 100
> done
>
> echo "생성된 점수 : ${scores[@]}"
> echo "요소 개수   : ${#scores[@]}개"
>
> # ② 최고 / 최저
> max=${scores[0]}
> min=${scores[0]}
> sum=0
>
> for score in "${scores[@]}"
> do
>     (( score > max )) && max=$score
>     (( score < min )) && min=$score
>     sum=$(( sum + score ))
> done
>
> echo "최고점 : $max / 최저점 : $min"
>
> # ③ 합계와 평균
> echo "합계   : $sum"
> echo "평균   : $(( sum / ${#scores[@]} ))점 (정수 나눗셈, 소수점 절삭)"
> echo "평균   : $(echo "scale=2; $sum/${#scores[@]}" | bc)점"
>
> # ④ 슬라이싱 / ⑤ 마지막 요소
> echo "1번부터 3개 : ${scores[@]:1:3}"
> echo "마지막 요소 : ${scores[-1]}"
> ```
>
> > `RANDOM` 은 모듈로 편향이 있고 예측 가능하므로 실습 데이터 생성에만 쓴다. 균일 난수는 `shuf -i 30-100 -n 10`, 비밀번호·토큰 생성에는 절대 사용하지 않는다.


---

## PART 13. 실무 종합 심화 (Level 3)

>  관련 문서: Shell Script - 통합 정리 · Shell Script - 위치 매개변수 (Positional Parameters) · Shell Script - 트러블슈팅 치트시트

> 이 파트는 **위치 매개변수 → 검증 → 배열 → 반복 → 조건 → 산술 → 종료 코드** 8계층을 한 스크립트에 모두 결합한다. 반드시 파일로 작성하고 `bash -n` → `bash -x` 순서로 검증할 것.

---

**Q57. 로그 백업 스크립트 (인자 검증 + 배열 + tar + 종료 코드)**

`log_backup.sh`를 작성하시오.

- 사용법: `./log_backup.sh <대상디렉터리> [저장디렉터리]` — 저장 디렉터리 기본값은 `/backup`.
- 인수 개수가 1~2개가 아니면 사용법 출력 후 `exit 1`.
- 대상 디렉터리가 없으면 종료, 저장 디렉터리가 없으면 생성한다.
- 대상 디렉터리의 `.log` 파일을 배열로 모으고, 하나도 없으면 안내 후 `exit 0`.
- 백업 파일명은 `대상디렉터리명_YYYY-MM-DD_HHMMSS.tar.gz` 형식.
- 압축 성공 시 크기와 아카이브 내용 일부를 출력하고 `exit 0`, 실패 시 남은 파일을 지우고 `exit 1`.

>  **힌트:**
> 
> - 기본값은 `dest="${2:-/backup}"`.
> - 파일명에 콜론이 들어가면 `scp`·타 OS 전송에서 문제가 되므로 `date +%F_%H%M%S` 를 사용한다.
> - 글롭이 하나도 매치되지 않으면 패턴 문자열이 그대로 배열에 남는다 → `[ ! -e "${files[0]}" ]` 로 확인.
> - `tar -C 상위경로 대상이름` 으로 상대경로 압축하면 `Removing leading '/'` 경고를 피할 수 있다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> # 사용법 : ./log_backup.sh <대상디렉터리> [저장디렉터리(기본 /backup)]
>
> # ① 개수 검증
> if [ $# -lt 1 ] || [ $# -gt 2 ]
> then
>     echo "Usage : $0 <source_dir> [dest_dir]"
>     exit 1
> fi
>
> src="$1"
> dest="${2:-/backup}"                        # 기본값 치환
>
> # ② 존재 검증
> if [ ! -d "$src" ]
> then
>     echo "대상 디렉터리가 없습니다 : $src"
>     exit 1
> fi
>
> if [ ! -d "$dest" ]
> then
>     mkdir -p "$dest" || { echo "저장 디렉터리 생성 실패 : $dest"; exit 1; }
> fi
>
> dest=$(cd "$dest" && pwd)                   # 상대경로를 절대경로로 정규화
> base=$(basename "$src")
> stamp=$(date +%F_%H%M%S)                    # 콜론 없는 형식
> archive="${dest}/${base}_${stamp}.tar.gz"
>
> # ③ 대상 목록 확보 (글롭 + 배열)
> cd "$src" || exit 1
> files=( *.log )
>
> if [ ! -e "${files[0]}" ]                   # 매치 실패 시 "*.log" 문자열이 그대로 남는다
> then
>     echo "$src : .log 파일이 없어 종료합니다."
>     exit 0
> fi
>
> echo "대상 ${#files[@]}개 → $archive"
>
> # ④ 압축 후 종료 코드로 판정
> tar -czf "$archive" "${files[@]}"
>
> if [ $? -eq 0 ]
> then
>     echo "백업 성공 : $(du -h "$archive" | awk '{print $1}')"
>     tar -tzf "$archive" | head -n 5         # 아카이브 내용 검증
>     exit 0
> else
>     echo "백업 실패"
>     rm -f "$archive"                        # 깨진 아카이브 정리
>     exit 1
> fi
> ```
>
> ```bash
> mkdir -p /temp/logtest && touch /temp/logtest/{app,error,access}.log
> chmod +x log_backup.sh
>
> ./log_backup.sh                             # Usage 출력 → exit 1
> ./log_backup.sh /temp/logtest               # /backup 에 저장
> ./log_backup.sh /temp/logtest /backup/logs  # 지정 경로에 저장
> echo $?
> ```
>
> > 저장 경로를 대상 디렉터리 **하위**로 지정하면 자기 자신을 압축하는 재귀가 발생한다. 두 경로는 반드시 독립된 위치로 둔다.


---

**Q58. 대량 계정 점검 & 자동 교정 스크립트**

`user_audit.sh`를 작성하시오.

- 점검 대상: `user1` ~ `user5`
- 계정이 없으면 생성한다(실패 시 `continue` 로 다음 계정).
- 홈 디렉터리가 없으면 생성한다.
- 소유자가 계정명과 다르면 `chown` 으로 교정한다.
- 권한이 `700` 이 아니면 `chmod 700` 으로 교정한다.
- 마지막에 `생성 N건 / 교정 N건 / 실패 N건` 을 집계해 출력하고, 실패가 있으면 `exit 1`.

>  **힌트:** 존재 확인은 `id "$user" > /dev/null 2>&1`, 소유자·권한 조회는 `stat -c "%U"` / `stat -c "%a"`. 각 단계마다 `$?` 를 확인하고 실패 시 `continue` 로 넘기는 방어 설계가 표준이다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> # 실행 : sudo ./user_audit.sh
>
> created=0
> fixed=0
> failed=0
>
> for user in user{1..5}
> do
>     # ① 계정 존재 확인 → 없으면 생성
>     if ! id "$user" > /dev/null 2>&1
>     then
>         useradd "$user"
>         if [ $? -ne 0 ]
>         then
>             echo "[FAIL] $user 계정 생성 실패"
>             (( failed++ ))
>             continue                        # 다음 계정으로 넘어간다
>         fi
>         echo "[NEW]  $user 계정 생성"
>         (( created++ ))
>     fi
>
>     home="/home/$user"
>
>     # ② 홈 디렉터리 확인
>     if [ ! -d "$home" ]
>     then
>         mkdir -p "$home" || { echo "[FAIL] $home 생성 실패"; (( failed++ )); continue; }
>         echo "[NEW]  $home 생성"
>     fi
>
>     # ③ 소유권 점검·교정
>     owner=$(stat -c "%U" "$home")
>     if [ "$owner" != "$user" ]
>     then
>         chown "$user:$user" "$home"
>         if [ $? -ne 0 ]
>         then
>             echo "[FAIL] $home 소유권 변경 실패"
>             (( failed++ ))
>             continue
>         fi
>         echo "[FIX]  $home 소유권 $owner → $user"
>         (( fixed++ ))
>     fi
>
>     # ④ 권한 점검·교정
>     perm=$(stat -c "%a" "$home")
>     if [ "$perm" -ne 700 ]
>     then
>         chmod 700 "$home"
>         if [ $? -ne 0 ]
>         then
>             echo "[FAIL] $home 권한 변경 실패"
>             (( failed++ ))
>             continue
>         fi
>         echo "[FIX]  $home 권한 $perm → 700"
>         (( fixed++ ))
>     fi
>
>     echo "[OK]   $user : $(stat -c '%U %a' "$home")"
> done
>
> echo "-----------------------------"
> echo "생성 ${created}건 / 교정 ${fixed}건 / 실패 ${failed}건"
>
> (( failed > 0 )) && exit 1
> exit 0
> ```
>
> ```bash
> # 검증
> stat -c "%U %G %a %n" /home/user{1..5}
>
> # 실습 후 정리
> userdel -r user1 user2 user3 user4 user5
> ```
>
> > `chown`/`chmod` 뒤에 `$?` 확인이 없으면 실패가 조용히 무시되어 "점검했는데 여전히 잘못된 상태"가 남는다.


---

**Q59. 서버 통합 점검 리포트 (배열 + case + 로그 + 종료 코드)**

`health_check.sh`를 작성하시오.

- 사용법: `./health_check.sh {report|log|all} [디스크임계값%] [메모리임계값MB]`
- 기본값: 모드 `report`, 디스크 `80`, 메모리 `200`.
- 임계값이 정수가 아니면 사용법 출력 후 `exit 1`.
- 점검 항목: ① 루트 파일시스템 사용률 ② 가용 메모리 ③ 서비스 배열(`sshd`, `crond`, `firewalld`) 활성 여부.
- 각 항목을 `[OK]` / `[WARN]` 으로 표기하고 경고 건수를 집계한다.
- 모드에 따라 `report`=화면 출력, `log`=`/var/log/health_check.log` 에 이어쓰기, `all`=둘 다.
- 경고가 하나라도 있으면 `exit 1`, 모두 정상이면 `exit 0`.

>  **힌트:**
> 
> - 사용률: `df / | awk 'NR==2 {print $5}' | tr -d '%'`, 가용 메모리: `free -m | awk 'NR==2 {print $7}'`.
> - 임시 파일은 `/tmp/health_$$.txt` 처럼 `$$`(PID)를 붙여 동시 실행 충돌을 막는다.
> - 결과를 한 번에 리다이렉션하려면 `{ 명령들; } > 파일` 로 **중괄호 그룹**을 사용한다. `( )` 서브셸을 쓰면 안에서 증가시킨 카운터가 밖으로 전달되지 않는다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> # 사용법 : ./health_check.sh {report|log|all} [disk%] [memMB]
>
> MODE="${1:-report}"
> DISK_LIMIT="${2:-80}"
> MEM_LIMIT="${3:-200}"
> LOGFILE="/var/log/health_check.log"
> TMP="/tmp/health_$$.txt"                     # PID로 임시파일 충돌 방지
>
> # ① 형식 검증
> if !  "$DISK_LIMIT" =~ ^[0-9]+$  || !  "$MEM_LIMIT" =~ ^[0-9]+$ 
> then
>     echo "Usage : $0 {report|log|all} [disk%] [memMB]"
>     exit 1
> fi
>
> services=("sshd" "crond" "firewalld")
> warn=0
>
> # ② 점검 실행 → 결과를 임시 파일로 모은다 ( { } 그룹은 서브셸이 아니므로 카운터 유지 )
> {
>     echo "===== 점검 시각 : $(date +'%F %T') / $(hostname) ====="
>
>     use=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
>     if (( use >= DISK_LIMIT ))
>     then
>         echo "[WARN] 디스크 사용률 ${use}% (임계 ${DISK_LIMIT}%)"
>         (( warn++ ))
>     else
>         echo "[OK]   디스크 사용률 ${use}%"
>     fi
>
>     free_mb=$(free -m | awk 'NR==2 {print $7}')
>     if (( free_mb < MEM_LIMIT ))
>     then
>         echo "[WARN] 가용 메모리 ${free_mb}MB (임계 ${MEM_LIMIT}MB)"
>         (( warn++ ))
>     else
>         echo "[OK]   가용 메모리 ${free_mb}MB"
>     fi
>
>     for svc in "${services[@]}"
>     do
>         if systemctl is-active --quiet "$svc"
>         then
>             echo "[OK]   $svc active"
>         else
>             echo "[WARN] $svc inactive"
>             (( warn++ ))
>         fi
>     done
>
>     echo "----- 경고 ${warn}건 -----"
> } > "$TMP"
>
> # ③ 출력 모드 분기
> case "$MODE" in
>     report) cat "$TMP" ;;
>     log)    cat "$TMP" >> "$LOGFILE" ; echo "로그 기록 완료 : $LOGFILE" ;;
>     all)    tee -a "$LOGFILE" < "$TMP" ;;
>     *)
>         echo "Usage : $0 {report|log|all} [disk%] [memMB]"
>         rm -f "$TMP"
>         exit 1
>         ;;
> esac
>
> rm -f "$TMP"                                 # 임시 파일 정리
>
> # ④ 결과를 종료 코드로 반환 (cron·상위 스크립트에서 판정 가능)
> (( warn > 0 )) && exit 1
> exit 0
> ```
>
> ```bash
> chmod +x health_check.sh
> ./health_check.sh                            # 기본값(report/80/200)
> ./health_check.sh all 60 500                 # 화면 + 로그, 임계값 지정
> echo $?                                      # 경고 있으면 1
> ./health_check.sh log abc                    # 형식 검증 실패 → Usage
> ```
>
> > 이 스크립트를 cron에 등록하면 `exit 1` 여부로 알림 발송을 판정할 수 있다. 종료 코드를 남기지 않으면 자동화에서 성공/실패를 구분할 수 없다.


---

**Q60.  버그 잡기 (디버깅 심화)**

다음 스크립트는 `.log` 파일 크기를 점검하려는 의도이지만 **버그가 5개** 있다. 각 버그의 원인과 수정 방법을 설명하고 올바르게 고친 스크립트를 작성하시오.

```bash
#!/bin/bash
# 사용법 : ./check_size.sh <디렉터리> <임계값KB>

if [$# -ne 2]                          # 버그 ①
then
    echo "Usage : $0 <dir> <limit_KB>"
    exit 1
fi

dir=$1
limit=$2

files=( $(ls $dir/*.log) )
count=${#files}                         # 버그 ②

echo "점검 대상 : ${count}개"

for f in "${files[*]}"                  # 버그 ③
do
    size=$(du -k "$f" | awk '{print $1}')
    if [ $size > $limit ]               # 버그 ④
    then
        echo "$f : 용량 초과 (${size}KB)"
    fi
done

for i in {1..$count}                    # 버그 ⑤
do
    echo "처리 회차 : $i"
done
```

>  **힌트:** 대괄호 공백 / 배열 개수 문법 / `[@]` vs `[*]` / `[ ]` 안의 `>` 는 무엇으로 해석되는가 / 중괄호 확장과 변수.

> [!success]-  정답 보기
>
> **버그 원인과 수정**
>
> | # | 잘못된 코드 | 원인 | 수정 |
> | --- | --- | --- | --- |
> | ① | `[$# -ne 2]` | `[` 는 명령이므로 양쪽에 공백이 필수. 문법 오류 발생 | `[ $# -ne 2 ]` |
> | ② | `${#files}` | `${#files[0]}` 과 동일 → **0번 요소의 문자 길이** 반환 | `${#files[@]}` |
> | ③ | `"${files[*]}"` | 전체를 하나의 문자열로 결합 → 반복이 **1회만** 실행 | `"${files[@]}"` |
> | ④ | `[ $size > $limit ]` | `>` 가 리다이렉션으로 해석되어 `$limit` 이름의 **빈 파일이 생성**되고 조건은 항상 참 | `[ "$size" -gt "$limit" ]` 또는 `(( size > limit ))` |
> | ⑤ | `{1..$count}` | 중괄호 확장은 파싱 시점에 처리되어 **변수를 인식하지 못함** (`{1..3}` 문자열 그대로) | `for (( i=1; i<=count; i++ ))` 또는 `$(seq 1 "$count")` |
>
> **보너스 결함:** `files=( $(ls $dir/*.log) )` 는 공백이 포함된 파일명을 쪼개고, 매치되는 파일이 없을 때 패턴 문자열이 그대로 남는다. 글롭을 직접 쓰고 `[ ! -e "${files[0]}" ]` 로 확인하는 편이 안전하다.
>
> **수정된 스크립트**
>
> ```bash
> #!/bin/bash
> # 사용법 : ./check_size.sh <디렉터리> <임계값KB>
>
> if [ $# -ne 2 ]                                  # ① 대괄호 안쪽 공백
> then
>     echo "Usage : $0 <dir> <limit_KB>"
>     exit 1
> fi
>
> dir="$1"
> limit="$2"
>
> if [ ! -d "$dir" ]
> then
>     echo "디렉터리가 없습니다 : $dir"
>     exit 1
> fi
>
> if !  "$limit" =~ ^[0-9]+$ 
> then
>     echo "임계값은 정수여야 합니다 : $limit"
>     exit 1
> fi
>
> files=( "$dir"/*.log )                           # 보너스 : ls 대신 글롭 사용
>
> if [ ! -e "${files[0]}" ]
> then
>     echo "$dir : .log 파일이 없습니다."
>     exit 0
> fi
>
> count=${#files[@]}                               # ② 요소 개수
> echo "점검 대상 : ${count}개"
>
> over=0
> for f in "${files[@]}"                           # ③ 개별 전개
> do
>     size=$(du -k "$f" | awk '{print $1}')
>
>     if (( size > limit ))                        # ④ 산술 평가로 숫자 비교
>     then
>         echo "$f : 용량 초과 (${size}KB)"
>         (( over++ ))
>     else
>         echo "$f : 정상 (${size}KB)"
>     fi
> done
>
> for (( i=1; i<=count; i++ ))                     # ⑤ C스타일 for
> do
>     echo "처리 회차 : $i"
> done
>
> echo "초과 ${over}건 / 전체 ${count}건"
> (( over > 0 )) && exit 1
> exit 0
> ```
>
> ```bash
> bash -n check_size.sh              # 문법 사전 검사 (① 같은 오류를 여기서 잡는다)
> bash -x ./check_size.sh /var/log 100   # 전개 과정 추적
> ```


---

## PART 14. cron · anacron (스케줄 자동화)

>  관련 문서: Shell Script - cron · anacron (스케줄 자동화) · Shell Script - 트러블슈팅 치트시트

### 핵심 개념 복습

```text
cron 스케줄  : 분 시 일 월 요일 [user] command  (5필드)
특수 표현    : */N (N단위 반복)  /  @daily  @weekly  @monthly  @reboot
주의 3원칙   : 절대 경로  /  chmod +x  /  >> log 2>&1 리다이렉션
anacron     : 시스템 꺼져 있던 동안 놓친 일·주·월 작업을 부팅 후 보완 실행
연동 패턴   : test -x /usr/sbin/anacron || run-parts /etc/cron.daily
```

---

### 기초 (Level 1)

**Q61.** 다음 cron 스케줄 표현식 5개의 의미를 설명하고, 주어진 조건에 맞는 표현식을 작성하시오.

**해석할 표현식:**

```text
① 0 3 * * *
② */5 * * * *
③ 0 1 * * 1
④ 30 18 1 * *
⑤ 0 */2 * * *
```

**작성할 표현식 (조건):**
- 매일 새벽 01시 30분
- 평일(월~금)만 매 10분마다
- 매월 15일 09시 00분

>  **힌트:** 필드 순서 — 분(0~59) / 시(0~23) / 일(1~31) / 월(1~12) / 요일(0~7, 0·7이 일요일). `*/N` 은 N 단위 반복. 요일 범위는 `1-5` (월~금).

> [!success]-  정답 보기
>
> **해석:**
>
> ```text
> ① 0 3 * * *         → 매일 새벽 03시 00분
> ② */5 * * * *       → 매 5분마다 (00분, 05분, 10분 …)
> ③ 0 1 * * 1         → 매주 월요일 01시 00분
> ④ 30 18 1 * *       → 매월 1일 오후 18시 30분
> ⑤ 0 */2 * * *       → 매 2시간마다 (00시, 02시, 04시 …)
> ```
>
> **작성:**
>
> ```bash
> 30 1 * * *           # 매일 01시 30분
> */10 * * * 1-5       # 평일(월~금)만 매 10분마다
> 0 9 15 * *           # 매월 15일 09시 00분
> ```


---

**Q62.** crond 서비스 상태를 확인하고 활성화한 뒤, `/etc/crontab`에 `/tmp/hello.sh` 스크립트를 매 5분마다 root 권한으로 실행하도록 등록하시오.

>  **힌트:** `/etc/crontab` 에는 `요일` 다음에 **실행 사용자** 필드가 하나 더 있다. 사용자별 `crontab -e` 와 구분할 것.

> [!success]-  정답 보기
>
> ```bash
> # 1. crond 상태 확인 및 자동 시작 등록
> systemctl status crond
> systemctl enable --now crond
>
> # 2. 현재 사용자 cron 목록 확인 (사용자별 crontab)
> crontab -l
>
> # 3. /etc/crontab 등록 (시스템 전체용, user 필드 필수)
> #    vi /etc/crontab 에서 추가:
> */5 * * * * root /tmp/hello.sh >> /var/log/hello.log 2>&1
>
> # 4. 등록 확인
> tail -1 /etc/crontab
> ```
>
> > `/etc/crontab` 은 6번째 필드에 실행 사용자명이 있다. 사용자별 `crontab -e` 에는 사용자 필드가 없다.


---

### 응용 (Level 2)

**Q63.** 서버에 현재 접속한 사용자 정보를 5분마다 자동으로 기록하는 시스템을 구축하시오.

**조건:**
- 스크립트: `/script/hourly/login_user_check.sh`
- 로그: `/var/log/login_user_check.log` (누적 저장, `>>`)
- 로그 형식: 구분선(=) → 확인 시간 (`date '+%F %T'`) → `who` 출력 순
- `/etc/crontab`에 5분마다 root 권한으로 실행 등록
- 수동 실행 후 로그 파일 내용 확인

>  **힌트:** cron은 터미널이 없으므로 모든 출력을 `>> "$LOG"` 로 리다이렉션해야 로그 파일에 기록된다. `chmod +x` 필수.

> [!success]-  정답 보기
>
> ```bash
> # 1. 디렉터리 생성
> mkdir -p /script/hourly
>
> # 2. 스크립트 작성
> vi /script/hourly/login_user_check.sh
> ```
>
> ```bash
> #!/bin/bash
>
> LOG="/var/log/login_user_check.log"
>
> echo "=================================================="  >> "$LOG"
> echo "확인시간 : $(date '+%F %T')"  >> "$LOG"
> who >> "$LOG"
> ```
>
> ```bash
> # 3. 실행 권한 부여 & 수동 테스트
> chmod +x /script/hourly/login_user_check.sh
> /script/hourly/login_user_check.sh
>
> # 4. 로그 확인
> cat /var/log/login_user_check.log
> # 출력 예:
> # ==================================================
> # 확인시간 : 2026-07-30 10:16:13
> # root     pts/0   2026-07-30 09:27 (192.168.10.1)
>
> # 5. /etc/crontab 등록 (vi로 편집 후 저장)
> # */5 * * * * root /script/hourly/login_user_check.sh
> ```
>
> > 수동 실행으로 정상 동작을 먼저 확인한 뒤 cron에 등록한다. 등록 후 `tail -f /var/log/login_user_check.log` 로 자동 실행을 실시간 모니터링할 수 있다.


---

**Q64.** `/backup/log` 디렉터리의 파일을 tar.gz로 백업하고, 수정된 지 7일을 초과한 파일을 자동으로 삭제하는 스크립트를 작성하시오.

**조건:**
- 스크립트: `/script/hourly/log_backup.sh`
- 백업 저장: `/temp/log/log_YYYY-MM-DD.tar.gz`
- 로그: `/var/log/log_cleanup.log` (백업 성공/실패 + 삭제된 파일 목록 누적)
- **백업 성공 시에만** 7일 초과 파일 삭제
- 테스트용 오래된 파일을 생성해 동작 확인
- `/etc/crontab`에 매일 23시 30분 실행 등록

>  **힌트:**
> - `date +%F` 는 **대문자 F** → `YYYY-MM-DD`. 소문자 `%f` 는 마이크로초 → 날짜 형식이 아니다 (흔한 실수).
> - `touch -d "8 days ago" 파일명` 으로 과거 타임스탬프 파일 생성.
> - `find -mtime +7 -print -delete` : 삭제 전 파일명을 출력(-print)하고 바로 삭제(-delete).

> [!success]-  정답 보기
>
> ```bash
> # 테스트용 디렉터리 & 오래된 파일 생성
> mkdir -p /backup/log
> touch -d "8 days ago"  /backup/log/oldFile8.log
> touch -d "10 days ago" /backup/log/oldFile10.log
> touch /backup/log/newFile.log        # 최신 파일 (삭제 대상 아님)
>
> ls -l /backup/log/
> ```
>
> ```bash
> #!/bin/bash
> # /script/hourly/log_backup.sh
>
> SRC="/backup/log/"
> DEST="/temp/log/"
> LOG="/var/log/log_cleanup.log"
> DATE="$(date '+%F')"                             # 반드시 대문자 %F
> BACKUP_FILE="${DEST}log_${DATE}.tar.gz"
>
> [ ! -d "$DEST" ] && mkdir -p "$DEST"
>
> echo "=================================================="  >> "$LOG"
> echo "$(date '+%F %T') - 로그 백업 시작"  >> "$LOG"
>
> tar czf "$BACKUP_FILE" -C "$SRC" .
>
> if [ $? -eq 0 ]
> then
>     echo "$(date '+%F %T') - 백업 성공 : $BACKUP_FILE" >> "$LOG"
>     echo "$(date '+%F %T') - 장기 로그 파일 삭제 시작"  >> "$LOG"
>     find "$SRC" -type f -mtime +7 -print -delete >> "$LOG"
>     echo "$(date '+%F %T') - 장기 로그 파일 삭제 완료"  >> "$LOG"
> else
>     echo "$(date '+%F %T') - 백업 실패" >> "$LOG"
> fi
> ```
>
> ```bash
> # 실행 권한 & 수동 테스트
> chmod +x /script/hourly/log_backup.sh
> /script/hourly/log_backup.sh
>
> # 결과 확인
> cat /var/log/log_cleanup.log
> ls -l /backup/log/                   # oldFile8, oldFile10 삭제됐는지 확인
> tar tzf /temp/log/log_$(date +%F).tar.gz   # 백업 내용 목록 (압축 안 풀고)
>
> # /etc/crontab 등록 (매일 23시 30분)
> # 30 23 * * * root /script/hourly/log_backup.sh
> ```


---

### 심화 (Level 3)

**Q65. anacron · cron 연동 시스템 구축**

다음 조건을 만족하는 anacron·cron 연동 시스템을 구축하시오.

**조건:**
- `/etc/cron.daily/daily_test` 스크립트를 작성한다 (**확장자 없이**).
- 스크립트는 실행 시간과 실행 주체를 `/var/log/daily_test.log`에 기록한다.
- anacron에 실행 권한이 있으면 **anacron**이 daily 작업을 담당한다.
- anacron에 실행 권한이 없으면 **cron**이 `/etc/cron.daily`를 직접 실행한다 (폴백).
- 로그에서 실행 주체(`ANACRON` / `CRON`)를 구분할 수 있어야 한다.
- anacron 권한을 제거해 cron 폴백을 테스트한 뒤 복원하시오.

>  **힌트:**
> - `run-parts` 는 파일명에 `.sh` 확장자가 있으면 실행 대상에서 제외 → 파일명에 확장자를 붙이지 않는다.
> - `${RUN_BY:-UNKNOWN}` : 환경변수 `RUN_BY` 가 없으면 `UNKNOWN` 사용 (기본값 치환).
> - `run-parts --test 디렉터리` : 실제 실행 없이 실행 대상 파일 목록만 출력.
> - `anacron -n -f` : 지연 없이(`-n`) 날짜 무관 강제 실행(`-f`).
> - `/var/spool/anacron/` : anacron의 마지막 실행 날짜 기록 위치 (당일 재실행 방지용).

> [!success]-  정답 보기
>
> **1단계: daily_test 스크립트 (확장자 없이)**
>
> ```bash
> vi /etc/cron.daily/daily_test
> ```
>
> ```bash
> #!/bin/bash
> LOG="/var/log/daily_test.log"
> echo "$(date '+%F %T') - 실행 주체 : ${RUN_BY:-UNKNOWN}" >> "$LOG"
> ```
>
> ```bash
> chmod +x /etc/cron.daily/daily_test
>
> # run-parts 실행 대상 확인 (.sh 없어야 포함됨)
> run-parts --test /etc/cron.daily/
> # 출력: /etc/cron.daily/daily_test
> ```
>
> **2단계: anacron 설정 (/etc/anacrontab)**
>
> ```bash
> # vi /etc/anacrontab 에서 추가:
> # 주기(일)  지연(분)  식별자             명령
> 1           1        cron.daily_test    env RUN_BY=ANACRON run-parts /etc/cron.daily
>
> # 문법 검사
> anacron -T
>
> # 강제 실행 (지연 없이, 날짜 무관)
> anacron -n -f
> cat /var/log/daily_test.log
> # 2026-07-30 15:27:28 - 실행 주체 : ANACRON
> ```
>
> **3단계: cron 폴백 등록 + 테스트**
>
> ```bash
> # /etc/crontab 에 추가:
> # anacron 권한 없을 때 cron이 직접 /etc/cron.daily 실행
> 55 15 * * * root test -x /usr/sbin/anacron || (cd / && env RUN_BY=CRON run-parts /etc/cron.daily)
>
> # anacron 실행 권한 제거 → 폴백 유도
> chmod -x /usr/sbin/anacron
> test -x /usr/sbin/anacron; echo $?    # 1 (권한 없음 확인)
>
> # cron 실행 후 로그 확인
> cat /var/log/daily_test.log
> # 2026-07-30 15:27:28 - 실행 주체 : ANACRON
> # 2026-07-30 16:11:01 - 실행 주체 : CRON   ← 폴백 동작 확인
> ```
>
> **4단계: 복원**
>
> ```bash
> chmod +x /usr/sbin/anacron
> ls -l /usr/sbin/anacron               # -rwxr-xr-x 확인
> ```
>
> > **동작 구조:**
> > - anacron 권한 있음 → anacron이 `/etc/anacrontab` 확인 → `env RUN_BY=ANACRON run-parts /etc/cron.daily`
> > - anacron 권한 없음 → `test -x` 실패 → `||` 오른쪽 실행 → `env RUN_BY=CRON run-parts /etc/cron.daily`


---

## PART 15. 허가권 & 소유권 (chmod · chown · umask · 특수권한)

>  관련 문서: 허가권 상세 (chmod & 8진수 · 심볼릭 표기) · 소유권 (Ownership) — chown & UID·GID 소유 모델 · Umask — 기본 권한 마스크 (User Mask) · 소유권 & 특수 권한 (chown & chgrp & SUID · SGID · Sticky)

### 핵심 개념 복습

```text
chmod 8진수   : r=4 w=2 x=1 / 파일 644 · 디렉터리 755 이 기본
chmod 심볼릭  : u/g/o/a  +/-/=  r/w/x/X
umask         : 최종권한 = 요청권한 & ~umask  (파일 0666 / 디렉터리 0777)
chown         : chown 소유자:그룹 파일  /  chown -R (재귀)
find -perm    : -perm 권한 (정확일치) / -perm -권한 (모두 포함) / -perm /권한 (하나 이상)
특수권한      : SUID(4xxx)  SGID(2xxx)  Sticky(1xxx)
stat -c "%a"  : 8진수 권한  /  stat -c "%U"  : 소유자명
```

---

### 기초 (Level 1)

**Q66.** umask 현재 설정값을 8진수·심볼릭 두 가지 방식으로 출력하고, 해당 umask에서 새로 생성되는 파일과 디렉터리의 기본 권한을 계산해 출력하는 스크립트를 작성하시오.

>  **힌트:** `umask`=8진수, `umask -S`=심볼릭 출력. 계산은 파일 `0666 & ~umask`, 디렉터리 `0777 & ~umask`. Bash 산술에서 `~` 는 비트 NOT이므로 `$(( 0666 & ~mask_int ))` 사용. `printf '%04o'` 로 8진수 형식 출력.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> mask=$(umask)
> echo "현재 umask (8진수) : $mask"
> echo "현재 umask (심볼릭) : $(umask -S)"
>
> # 8진수 문자열 → 정수 (8진수로 해석)
> mask_int=$(( 8#${mask} ))
>
> file_perm=$(printf '%04o' $(( 0666 & ~mask_int )) )
> dir_perm=$(printf  '%04o' $(( 0777 & ~mask_int )) )
>
> echo "새 파일 기본 권한     : $file_perm"
> echo "새 디렉터리 기본 권한 : $dir_perm"
> ```
>
> 실행 예 (umask 0022):
> ```text
> 현재 umask (8진수) : 0022
> 현재 umask (심볼릭) : u=rwx,g=rx,o=rx
> 새 파일 기본 권한     : 0644
> 새 디렉터리 기본 권한 : 0755
> ```


---

**Q67.** `/home` 디렉터리 아래에서 SUID가 설정된 파일과 권한이 정확히 `777`인 파일을 각각 찾아 목록과 개수를 출력하는 스크립트를 작성하시오.

>  **힌트:** `find -perm -4000` 은 SUID 비트가 포함된 파일. `-perm 0777` 은 정확히 777. `2>/dev/null` 로 권한 거부 메시지를 억제한다.

```bash
# 예제
find /usr -perm -4000 2>/dev/null    # SUID 파일
find /tmp -perm 0777  2>/dev/null    # 정확히 777인 파일
```

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> TARGET="/home"
>
> echo "=== SUID 파일 (${TARGET}) ==="
> find "$TARGET" -perm -4000 -type f 2>/dev/null
>
> echo
> echo "=== 권한 777 파일 (${TARGET}) ==="
> find "$TARGET" -perm 0777 2>/dev/null
>
> suid_cnt=$(find "$TARGET" -perm -4000 -type f 2>/dev/null | wc -l)
> world_cnt=$(find "$TARGET" -perm 0777 2>/dev/null | wc -l)
> echo
> echo "SUID 파일 : ${suid_cnt}개 / 777 파일 : ${world_cnt}개"
> ```


---

### 응용 (Level 2)

**Q68.** 디렉터리 `/project` 아래 파일·디렉터리에 다음 규칙을 한 스크립트로 적용하시오.

- 디렉터리에는 `750` 권한, 파일에는 `640` 권한을 재귀 적용한다.
- 소유자를 `root`, 소유 그룹을 `devteam`으로 변경한다.
- 적용 후 `stat -c "%n %a %U %G"` 형식으로 각 항목을 출력한다.

>  **힌트:** `find -type d -exec chmod {} +` / `find -type f -exec chmod {} +` 로 파일·디렉터리를 분리해 권한을 적용한다. `-exec ... {} +` 는 `-exec ... {} \;` 보다 효율적(여러 파일을 한 번에 처리).

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> TARGET="/project"
>
> # 테스트 구조 생성
> mkdir -p "$TARGET"/{src,docs}
> touch "$TARGET"/src/{main.sh,lib.sh} "$TARGET"/docs/README.txt
>
> # 그룹 없으면 생성
> getent group devteam > /dev/null 2>&1 || groupadd devteam
>
> # 권한 재귀 적용 (파일·디렉터리 분리)
> find "$TARGET" -type d -exec chmod 750 {} +
> find "$TARGET" -type f -exec chmod 640 {} +
>
> # 소유권 재귀 변경
> chown -R root:devteam "$TARGET"
>
> echo "=== 적용 결과 ==="
> find "$TARGET" -exec stat -c "%n %a %U %G" {} +
> ```
>
> >  실습 후 `rm -rf /project && groupdel devteam` 로 정리할 것.


---

**Q69.** 팀 공유 디렉터리 `/shared/team` 을 생성하고 다음 보안 설정을 자동화하는 `setup_shared.sh` 를 작성하시오.

- 소유 그룹을 `teamgrp` 으로 변경한다.
- **SGID + Sticky bit** 를 동시에 설정한다 (새 파일 그룹 자동 상속 + 자신의 파일만 삭제 가능).
- 디렉터리 권한을 `2770` 으로 설정한다.
- 설정 후 `ls -ld` 로 결과를 출력한다.

>  **힌트:** SGID(2000) + Sticky(1000) + rwxrwx--- = `3770`. `ls -ld` 출력에서 그룹 실행 자리의 `s` (SGID), Other 실행 자리의 `T` (Sticky) 가 보여야 한다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> DIR="/shared/team"
>
> # 그룹 생성 (없으면)
> getent group teamgrp > /dev/null 2>&1 || groupadd teamgrp
>
> # 디렉터리 생성 및 설정
> mkdir -p "$DIR"
> chown :teamgrp "$DIR"
> chmod 3770 "$DIR"    # SGID(2000) + Sticky(1000) + rwxrwx--- = 3770
>
> echo "=== 설정 결과 ==="
> ls -ld "$DIR"
> stat -c "권한(8진수): %a  소유자: %U  그룹: %G" "$DIR"
> ```
>
> 실행 예:
> ```text
> drwxrws--T.  2 root teamgrp  6 2026-07-31 ...
> 권한(8진수): 3770  소유자: root  그룹: teamgrp
> ```
>
> > `ls -ld` 에서 그룹 실행 자리가 `s` 이면 SGID, Other 실행 자리가 `T` (실행권한 없음) 또는 `t` (실행권한 있음) 이면 Sticky bit다.


---

### 심화 (Level 3)

**Q70. 보안 감사 스크립트 — SUID · SGID · 777 파일 리포트**

`permission_audit.sh` 를 작성하시오.

- 점검 대상 디렉터리를 인수로 받는다. 없으면 `/` 전체를 대상으로 한다.
- 다음 세 항목을 점검하고 발견 목록을 `/var/log/perm_audit_YYYYMMDD.log` 에 기록한다.
  - SUID 파일 (`-perm -4000`)
  - SGID 파일 (`-perm -2000`)
  - 권한 777 파일 (`-perm 0777`)
- 각 항목 개수를 집계해 출력하고, 777 파일이 하나라도 있으면 `exit 1`.

>  **힌트:** `find ... | tee -a "$LOG"` 로 화면과 로그에 동시에 출력한다. `/proc`, `/sys` 는 검색 제외(`-prune`). `find ... | wc -l` 로 개수를 센다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> TARGET="${1:-/}"
> DATE=$(date +%Y%m%d)
> LOG="/var/log/perm_audit_${DATE}.log"
>
> # /proc /sys 는 가상 파일시스템이므로 제외
> FIND_BASE=(find "$TARGET" -path /proc -prune -o -path /sys -prune -o)
>
> echo "=== 권한 감사 : $(date +'%F %T') / 대상 : $TARGET ===" | tee -a "$LOG"
>
> echo; echo "--- SUID 파일 ---" | tee -a "$LOG"
> "${FIND_BASE[@]}" -perm -4000 -type f -print 2>/dev/null | tee -a "$LOG"
> suid_cnt=$("${FIND_BASE[@]}" -perm -4000 -type f -print 2>/dev/null | wc -l)
>
> echo; echo "--- SGID 파일 ---" | tee -a "$LOG"
> "${FIND_BASE[@]}" -perm -2000 -type f -print 2>/dev/null | tee -a "$LOG"
> sgid_cnt=$("${FIND_BASE[@]}" -perm -2000 -type f -print 2>/dev/null | wc -l)
>
> echo; echo "--- 권한 777 파일 ---" | tee -a "$LOG"
> "${FIND_BASE[@]}" -perm 0777 -print 2>/dev/null | tee -a "$LOG"
> world_cnt=$("${FIND_BASE[@]}" -perm 0777 -print 2>/dev/null | wc -l)
>
> echo | tee -a "$LOG"
> echo "=== 집계 : SUID ${suid_cnt}개 / SGID ${sgid_cnt}개 / 777 ${world_cnt}개 ===" | tee -a "$LOG"
> echo "로그 저장 : $LOG"
>
> if (( world_cnt > 0 ))
> then
>     echo "[경고] 권한 777 파일 발견 — 즉시 점검하세요." | tee -a "$LOG"
>     exit 1
> fi
>
> exit 0
> ```
>
> ```bash
> chmod +x permission_audit.sh
> ./permission_audit.sh /home      # 특정 디렉터리만 점검
> ./permission_audit.sh            # 전체 시스템 (시간 소요)
> echo $?
> ```


---

## PART 16. 네트워크 서비스 자동화 (SSH · SCP · FTP)

>  관련 문서: SSH 개념 & 프로세스·보안 설정 · SCP 파일 전송 (Linux·Windows) · vsFTP 설치 & 접근 제어 (user_list·chroot)

### 핵심 개념 복습

```text
SCP
  업로드   : scp 로컬경로 계정@호스트:원격경로
  다운로드 : scp 계정@호스트:원격경로 로컬경로
  디렉터리 : -r 옵션 필수  /  포트 지정 : -P 포트번호
  $? 로 성공(0)/실패(비0) 판단

SSH 키 인증 자동화
  키 생성  : ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
  배포     : ssh-copy-id -o StrictHostKeyChecking=no -i 키파일 계정@호스트
  검증     : ssh -o BatchMode=yes -o ConnectTimeout=5 호스트 true

vsftpd 상태 확인
  서비스   : systemctl is-active vsftpd
  포트     : ss -tnlp | grep :21
```

---

### 기초 (Level 1)

**Q71.** 인수로 `<source> <destination>` 두 개를 받아 SCP 전송을 수행하고 결과를 로그에 기록하는 `scp_transfer.sh` 를 작성하시오.

- 인수가 2개가 아니면 업로드·다운로드 사용법 예시를 포함한 Usage 출력 후 `exit 1`.
- 전송 성공/실패를 타임스탬프와 함께 `/var/log/scp_transfer.log` 에 기록한다.

>  **힌트:** `scp "$1" "$2"` 로 두 인수를 그대로 전달하면 업로드·다운로드 모두 처리된다. 소스에 `계정@호스트:경로` 가 있으면 다운로드, 목적지에 있으면 업로드이다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> LOG="/var/log/scp_transfer.log"
>
> if [ $# -ne 2 ]
> then
>     echo "Usage : $0 <source> <destination>"
>     echo "  업로드  : $0 /local/file.txt user@host:/remote/"
>     echo "  다운로드: $0 user@host:/remote/file.txt /local/"
>     exit 1
> fi
>
> echo "$(date '+%F %T') - 전송 시작 : $1 → $2" >> "$LOG"
>
> scp "$1" "$2"
>
> if [ $? -eq 0 ]
> then
>     echo "$(date '+%F %T') - 전송 성공 : $1 → $2" >> "$LOG"
>     echo "전송 성공"
> else
>     echo "$(date '+%F %T') - 전송 실패 : $1 → $2" >> "$LOG"
>     echo "전송 실패"
>     exit 1
> fi
> ```
>
> ```bash
> chmod +x scp_transfer.sh
> ./scp_transfer.sh /etc/hostname root@192.168.10.100:/tmp/
> ./scp_transfer.sh root@192.168.10.100:/tmp/hostname /tmp/recv_hostname
> ```


---

### 응용 (Level 2)

**Q72.** vsFTP(`vsftpd`) 서비스가 중지 상태이거나 21번 포트가 열려 있지 않으면 자동으로 재시작하는 `ftp_watchdog.sh` 를 작성하시오.

- 서비스 활성 여부는 `systemctl is-active`, 포트 수신 여부는 `ss -tnlp | grep :21` 로 확인한다.
- 상태와 조치 결과를 `/var/log/ftp_watchdog.log` 에 타임스탬프와 함께 기록한다.
- 재시작 실패 시 `exit 1` 로 종료한다.

>  **힌트:** 서비스가 `active` 이더라도 포트가 없으면 실제 동작하지 않는 경우가 있다. 두 조건을 **모두** 확인하는 것이 중요하다. `ts()` 함수를 만들어 타임스탬프 중복을 줄인다.

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
>
> LOG="/var/log/ftp_watchdog.log"
> SVC="vsftpd"
> ts() { date '+%F %T'; }
>
> status=$(systemctl is-active "$SVC")
> port_open=$(ss -tnlp 2>/dev/null | grep -c ':21')
>
> echo "$(ts) - 상태: $status / 포트21: ${port_open}개" >> "$LOG"
>
> if [ "$status" != "active" ] || [ "$port_open" -eq 0 ]
> then
>     echo "$(ts) - $SVC 이상 감지 → 재시작" | tee -a "$LOG"
>
>     systemctl restart "$SVC"
>
>     if [ $? -eq 0 ]
>     then
>         echo "$(ts) - 재시작 성공 / 상태: $(systemctl is-active "$SVC")" >> "$LOG"
>     else
>         echo "$(ts) - 재시작 실패" >> "$LOG"
>         exit 1
>     fi
> else
>     echo "$(ts) - $SVC 정상 운영 중" >> "$LOG"
> fi
>
> exit 0
> ```
>
> ```bash
> chmod +x ftp_watchdog.sh
> ./ftp_watchdog.sh
> cat /var/log/ftp_watchdog.log
>
> # cron으로 1분마다 감시 등록 예시:
> # * * * * * root /script/ftp_watchdog.sh
> ```


---

### 심화 (Level 3)

**Q73. SSH 공개키 일괄 배포 & 검증 스크립트**

여러 서버에 SSH 공개키를 자동 배포하는 `deploy_ssh_key.sh` 를 작성하시오.

- 대상 서버 목록을 배열 `HOSTS=("192.168.10.101" "192.168.10.102" "192.168.10.103")` 으로 정의한다.
- `~/.ssh/id_ed25519` 가 없으면 `ssh-keygen` 으로 키를 먼저 생성한다 (비대화식, 빈 패스프레이즈).
- 각 서버에 `ssh-copy-id` 로 공개키를 배포하고 성공/실패를 집계한다.
- 배포 성공한 서버에만 `ssh -o BatchMode=yes` 로 키 인증 접속을 검증한다.
- 마지막에 `배포 성공 N대 / 검증 성공 N대 / 실패 N대` 를 출력하고 실패가 있으면 `exit 1`.

>  **힌트:**
>
> - `ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519` : 빈 패스프레이즈 비대화식 생성.
> - `ssh-copy-id -o StrictHostKeyChecking=no -i 키파일 계정@호스트` : 비대화식 배포.
> - `ssh -o BatchMode=yes -o ConnectTimeout=5 호스트 true` : 키 인증 검증 (비밀번호 입력창 없음).

> [!success]-  정답 보기
>
> ```bash
> #!/bin/bash
> # 사용법 : ./deploy_ssh_key.sh [계정명]  (기본: root)
>
> USER="${1:-root}"
> KEY="$HOME/.ssh/id_ed25519"
> HOSTS=("192.168.10.101" "192.168.10.102" "192.168.10.103")
>
> deployed=0
> verified=0
> failed=0
>
> # ① 키가 없으면 비대화식 생성
> if [ ! -f "$KEY" ]
> then
>     echo "SSH 키 없음 → 생성 중..."
>     ssh-keygen -t ed25519 -N "" -f "$KEY"
>     if [ $? -ne 0 ]
>     then
>         echo "키 생성 실패. 종료합니다."
>         exit 1
>     fi
>     echo "키 생성 완료 : $KEY"
> fi
>
> # ② 각 서버에 공개키 배포 + 검증
> for host in "${HOSTS[@]}"
> do
>     echo
>     echo "=== $host ==="
>
>     ssh-copy-id -o StrictHostKeyChecking=no \
>                 -i "${KEY}.pub" \
>                 "${USER}@${host}" 2>/dev/null
>
>     if [ $? -eq 0 ]
>     then
>         echo "$host : 배포 성공"
>         (( deployed++ ))
>
>         # ③ 키 인증 접속 검증
>         ssh -o BatchMode=yes -o ConnectTimeout=5 \
>             "${USER}@${host}" true 2>/dev/null
>
>         if [ $? -eq 0 ]
>         then
>             echo "$host : 검증 성공"
>             (( verified++ ))
>         else
>             echo "$host : 배포됐으나 키 인증 실패"
>             (( failed++ ))
>         fi
>     else
>         echo "$host : 배포 실패"
>         (( failed++ ))
>     fi
> done
>
> echo
> echo "============================="
> echo "배포 성공 : ${deployed}대"
> echo "검증 성공 : ${verified}대"
> echo "실 패     : ${failed}대"
>
> (( failed > 0 )) && exit 1
> exit 0
> ```
>
> ```bash
> chmod +x deploy_ssh_key.sh
> ./deploy_ssh_key.sh root     # root 계정으로 배포
> ./deploy_ssh_key.sh guest    # guest 계정으로 배포
> ```
>
> > 실습 환경에서는 `ssh-copy-id` 가 대상 서버의 비밀번호를 요구한다. 완전 비대화식 자동화가 필요하면 `sshpass -p 비밀번호 ssh-copy-id ...` 를 사용하지만, 비밀번호가 셸 히스토리에 남으므로 배포 전용 계정·제한 환경에서만 사용한다.


---

>  **공통 풀이 팁**
> 
> - 터미널에서 한 줄씩 실행해보고 완성되면 스크립트 파일로 옮긴다.
> - `bash -n 스크립트.sh` : 실행 없이 문법만 검사 (작성 직후 습관화).
> - `bash -x 스크립트.sh` : 디버그 모드 실행 (명령 하나씩 출력하며 실행).
> - 오류 발생 시 `echo $?` 로 종료 코드 확인.
> - 파괴적 명령 전에는 `printf '%s\n' 패턴` 으로 대상 범위를 먼저 확인한다.
> - 무한루프가 발생하면 `Ctrl+C` 로 중단.

>  **자기 점검 체크리스트**
> 
> - [ ] `[ ]` 안쪽 공백, 변수 겹따옴표(`"$var"`)를 습관적으로 쓰는가
> - [ ] 배열 개수는 `${#arr[@]}`, 순회는 `"${arr[@]}"` 로 구분해 쓰는가
> - [ ] 스크립트 첫 로직이 **개수 → 형식 → 존재** 3단 검증 + `exit 1` 인가
> - [ ] 반복문 안에서 각 명령의 `$?` 를 확인하고 실패 시 `continue` 로 방어하는가
> - [ ] 마지막에 `exit 0` / `exit 1` 로 결과를 명확히 반환하는가

---

>  **관련 문서**
> 
> - 개념 문서: Shell Script - 변수와 환경변수 (커널·쉘 개념 포함) · Shell Script - Metacharacters (메타문자) · Shell Script - expr · let (산술 연산) · Shell Script - exit 상태와 test 명령 · Shell Script - 조건문 (if · case) · Shell Script - 반복문 (for · while · until) · Shell Script - 배열(Array)과 RANDOM · Shell Script - 위치 매개변수 (Positional Parameters) · Shell Script - cron · anacron (스케줄 자동화)
> - 정리·참고: Shell Script - 통합 정리 · Shell Script - 트러블슈팅 치트시트 · Shell Script - 명령어 퀵 레퍼런스
> - Linux 연계: 허가권 상세 (chmod & 8진수 · 심볼릭 표기) · 소유권 (Ownership) — chown & UID·GID 소유 모델 · Umask — 기본 권한 마스크 (User Mask) · 소유권 & 특수 권한 (chown & chgrp & SUID · SGID · Sticky) · SSH 개념 & 프로세스·보안 설정 · SCP 파일 전송 (Linux·Windows) · vsFTP 설치 & 접근 제어 (user_list·chroot)

---
