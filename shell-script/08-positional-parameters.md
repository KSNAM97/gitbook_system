# 🎯 Shell Script - 위치 매개변수 (Positional Parameters)

> **Tag:** #Linux #ShellScript #Bash #PositionalParameters #위치매개변수 #인자 #shift #Usage #exit
> **핵심 요약:** 위치매개변수(Positional Parameters)는 스크립트를 실행할 때 명령 뒤에 전달한 값을 `$1`, `$2`, `$3` 순서로 자동 저장하는 특수 변수다. `read` 와 달리 대화형 입력 없이 실행 시점에 값을 넘기므로 cron·자동화에 필수적이다. `$0`(스크립트 이름), `$#`(인자 개수), `"$@"`(개별 전개), `"$*"`(문자열 결합)를 함께 사용하며, 두 자리 이상은 `${10}` 처럼 중괄호가 필수다. 실무 스크립트는 반드시 인자 개수·형식을 검증하고 실패 시 Usage 출력과 `exit 1` 로 조기 종료한다.

---

## 1. 📖 개요 (Overview)

`read` 는 실행 중 사용자가 키보드로 값을 입력해야 하므로 **사람이 앞에 있어야** 한다. 위치 매개변수는 실행 명령줄에 값을 함께 전달하므로 cron·systemd timer·다른 스크립트에서 호출하는 **완전 자동 실행**이 가능하다. 리눅스 명령 자체가 동일한 구조다(`cp /etc/passwd /temp/passwd` → `$0=cp`, `$1`, `$2`). 전달 값은 공백을 기준으로 순서대로 `$1`, `$2` … 에 저장되고, 숫자든 문자열이든 모두 문자열로 저장된다. 인자를 하나도 주지 않고 실행하면 `$1~` 은 빈 값, `$#` 은 `0` 이 되므로, 스크립트 첫 줄의 인자 검증이 필수다. `read` 와 병행 설계도 가능한데, 인자가 있으면 그 값을 쓰고 없으면 `read` 로 물어보는 방식이다(`[ -z "$1" ] && read -p "..." val`).

`$0` 은 실행된 스크립트 이름(입력한 경로 그대로), `$#` 은 전달된 인자 개수, `$@` 와 `$*` 는 전체 인자다. 결정적 차이는 **따옴표로 감쌌을 때**로, `"$@"` 는 인자를 각각 독립된 값으로 유지하고 `"$*"` 는 `IFS` 첫 문자로 이어붙인 하나의 문자열이 된다. `$#` 에는 `$0` 이 포함되지 않으므로 `./a.sh x y` 라면 `$#` 은 `2` 다. 두 자리 이상 인자는 `${10}` 처럼 **중괄호가 필수**이며, `$10` 은 `$1` 뒤에 문자 `0` 이 붙은 것으로 해석된다. 파일명에 공백이 있을 수 있는 인자를 다른 명령에 그대로 넘길 때는 반드시 `"$@"` 를 사용한다.

```bash
./test.sh "kim lee" park
# "$@" → "kim lee" "park"   (2개 인자 유지)
# "$*" → "kim lee park"     (1개 문자열)
```

인자 검증의 표준 패턴은 ① 개수 검증(`[ $# -ne 2 ]`) → ② 형식 검증(정수 여부 등 정규식 `[[ "$1" =~ ^[0-9]+$ ]]`) → ③ 존재 검증(`[ ! -d "$1" ]`) 순서로 확인하고, 어느 단계든 실패하면 Usage 메시지를 출력하고 `exit 1` 로 즉시 종료하는 것이다. 검증 없이 본 로직에 들어가면 빈 값으로 파일을 삭제·덮어쓰는 사고가 발생한다. Usage 메시지에는 하드코딩된 이름 대신 `$0` 을 사용해 실제 실행 경로가 그대로 출력되게 한다(`echo "Usage : $0 source_dir dest_dir"`). `[[ "$1" =~ ^[0-9]+$ ]]` 는 음수와 소수를 거부하며, 음수를 허용해야 하면 `^-?[0-9]+$` 로 확장한다. `[[ ]]` 와 `=~` 는 Bash 확장 문법이므로 `sh script.sh` 로 실행하면 동작하지 않고, shebang(`#!/bin/bash`)과 `./script.sh` 실행을 전제로 한다. 기본값이 필요하면 `${1:-기본값}` 형태로 대체할 수 있다(예: `dest="${2:-/backup}"`).

인자가 가변 개수일 때는 `for arg in "$@"` 로 전체를 순회하거나, `shift` 로 인자를 하나씩 앞으로 밀면서 `while [ $# -gt 0 ]` 루프로 처리한다. `shift` 는 `$2→$1`, `$3→$2` 로 재배치하며 `$#` 을 1 감소시키므로, 첫 인자를 명령으로 소비한 뒤 남은 인자를 대상 목록으로 다루는 서브커맨드 구조에 적합하다. `shift N` 으로 여러 개를 한 번에 밀 수 있으며, 남은 인자보다 큰 값을 `shift` 하면 실패(비정상 종료 코드)한다. 옵션 파싱이 복잡해지면 `while getopts "a:b" opt` 또는 `case "$1" in --opt) ... ;; esac` + `shift` 조합으로 구조화하며, `set -- 값1 값2` 로 위치 매개변수를 스크립트 내부에서 임의로 재설정할 수도 있다.

위치 매개변수와 함께 자주 쓰는 특수 변수로는 `$?`(직전 명령 종료 코드), `$$`(현재 스크립트의 PID, 임시파일 이름에 활용), `$!`(마지막 백그라운드 프로세스 PID), `$_`(직전 명령의 마지막 인자)가 있다. 자동화 스크립트에서는 `$?` 와 `$$` 조합이 특히 자주 쓰인다. 임시 파일 충돌 방지에는 `tmp="/tmp/backup_$$.list"` (동시 실행 시 PID로 파일명 분리)를 쓰며, 더 안전한 방법은 `mktemp` 사용이다. 함수 안에서 `$1`, `$2` 는 스크립트 인자가 아니라 **함수 인자**를 의미하므로, 함수 내부에서 스크립트 인자가 필요하면 별도 변수에 미리 저장해 두어야 한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 위치 매개변수 목록

| 변수 | 의미 | 비고 |
| --- | --- | --- |
| `$0` | 실행된 스크립트 이름 | 입력한 경로 그대로. 이름만 필요하면 `basename "$0"` |
| `$1` ~ `$9` | 1~9번째 인자 | |
| `${10}` 이상 | 10번째 이후 인자 | 중괄호 필수 (`$10` 은 `$1`+`0`) |
| `$#` | 인자 개수 | `$0` 은 포함되지 않음 |
| `"$@"` | 전체 인자를 개별 값으로 전개 | 다른 명령에 넘길 때 표준 |
| `"$*"` | 전체 인자를 하나의 문자열로 결합 | 메시지 출력용 |
| `$?` | 직전 명령 종료 코드 | 0=성공 |
| `$$` | 현재 스크립트 PID | 임시파일명 등 |
| `$!` | 마지막 백그라운드 PID | |

### 2-2. 인자 확인 기본 골격

```bash
#!/bin/bash

echo "스크립트 이름 : $0"
echo "첫 번째 값 : $1"
echo "두 번째 값 : $2"
echo "세 번째 값 : $3"
echo "인자 개수 : $#"
echo "개별 전개 : $@"
echo "문자열 결합 : $*"
```

```bash
chmod +x ./script/test.sh
./script/test.sh apple banana cherry
```

```bash
스크립트 이름 : ./script/test.sh
첫 번째 값 : apple
두 번째 값 : banana
세 번째 값 : cherry
인자 개수 : 3
개별 전개 : apple banana cherry
문자열 결합 : apple banana cherry
```

### 2-3. 인자 검증 표준 템플릿

```bash
#!/bin/bash

# ① 개수 검증
if [ $# -ne 2 ]
then
    echo "Usage : $0 <number1> <number2>"
    exit 1
fi

# ② 형식 검증 (정수 여부)
for arg in "$@"
do
    if ! [[ "$arg" =~ ^-?[0-9]+$ ]]        # 음수 허용, 소수 거부
    then
        echo "Usage : $0 <number1> <number2>"
        echo "정수가 아닌 값 : $arg"
        exit 1
    fi
done

# ③ 존재 검증 (경로 인자일 때)
# [ ! -d "$1" ] && { echo "디렉터리가 없습니다 : $1"; exit 1; }

echo "검증 통과 : $1, $2"
```

### 2-4. 기본값 · shift 활용 템플릿

```bash
src="$1"
dest="${2:-/backup}"              # 2번 인자가 없으면 /backup 사용

action="$1"; shift                 # 첫 인자를 동작으로 소비하고 나머지를 대상으로 사용
for target in "$@"
do
    echo "$action 대상 : $target"
done

while [ $# -gt 0 ]                 # 가변 인자 순차 처리
do
    echo "처리 중 : $1"
    shift
done
```

### 2-5. 서브커맨드형 스크립트 골격 (위치 매개변수 + case)

```bash
#!/bin/bash

if [ $# -ne 2 ]
then
    echo "Usage : $0 {install|start|stop|restart|status} <service-name>"
    exit 1
fi

action="$1"
service="$2"

case "$action" in
    install|start|stop|restart|status)
        echo "$service 서비스에 $action 을 수행한다."
        ;;
    *)
        echo "Unknown Action : $action"
        echo "Usage : $0 {install|start|stop|restart|status} <service-name>"
        exit 1
        ;;
esac
```

---

## 3. 📝 실습 예제 모음 (EX)

### 3-1. 두 정수 비교하여 큰 값 출력

```bash
#!/bin/bash
# 인자 2개를 받아 더 큰 값을 출력, 개수가 다르면 사용법 안내 후 종료

if [ $# -ne 2 ]                              # $#: 인자 개수; -ne 2: 2가 아니면 Usage 출력
then
    echo "Usage : $0 <number1> <number2>"
    exit 1
fi

num1="$1"
num2="$2"

if [ "$num1" -gt "$num2" ]                   # -gt: 크면 참 (greater than)
then
    echo "큰 수 : $num1"
else
    echo "큰 수 : $num2"
fi
```

```bash
./script/positional_example01.sh 100
# Usage : ./script/positional_example01.sh <number1> <number2>

./script/positional_example01.sh 100 200
# 큰 수 : 200
```

### 3-2. 전달된 모든 정수의 합 (가변 인자 + 형식 검증)

```bash
#!/bin/bash
# 인자로 받은 모든 정수의 합을 출력, 정수가 아닌 값이 있으면 종료

if (( $# == 0 ))                             # (( )): 산술 비교; $# 는 인자 개수
then
    echo "Usage : $0 <int1> <int2> ..."
    exit 1
fi

for num in "$@"                              # "$@": 전체 인자를 개별 전개 — 공백 포함 인자 안전
do
    if ! [[ "$num" =~ ^[0-9]+$ ]]           # =~: 정규식 매칭; ^[0-9]+$: 양의 정수만 허용
    then
        echo "Usage : $0 <int1> <int2> ..."
        echo "정수가 아닌 값 : $num"
        exit 1
    fi
done

sum=0
for num in "$@"
do
    sum=$(( sum + num ))                     # $(( )): 산술 확장으로 누적 합산
done

echo "인자 개수 : $#"
echo "인자 총합 : $sum"
```

```bash
./script/positional_example02.sh 1 2 3 4 5     # 인자 개수 : 5 / 인자 총합 : 15
./script/positional_example02.sh {1..10}       # 중괄호 확장으로 1~10 전달 → 총합 55
./script/positional_example02.sh 10 20 A       # 정수가 아닌 값 : A
```

> 원본 예제처럼 `$1`, `$2` 만 검증하면 `10 20 A` 가 통과되어 합계가 `30` 으로 잘못 계산된다. 가변 인자는 전체를 순회하며 검증해야 한다.

### 3-3. 서비스 설치 및 제어 스크립트

```bash
#!/bin/bash
# $1 : install|start|stop|restart|status , $2 : 서비스 이름
# sshd 는 패키지명이 openssh-server 이므로 매핑 처리

if [ $# -ne 2 ]
then
    echo "Usage : $0 {install|start|stop|restart|status} <service-name>"
    exit 1
fi

action="$1"
service="$2"

case "$action" in                            # 변수 값을 패턴과 순서대로 비교
    install)
        if [ "$service" = "sshd" ]
        then
            package="openssh-server"          # 서비스명 ≠ 패키지명 매핑
        else
            package="$service"
        fi

        echo "$package 패키지를 설치한다."
        dnf install -y "$package"            # -y: 설치 확인 없이 자동 진행
        # dnf install -y "$package" > /dev/null 2>&1   # 출력 숨김 (오류도 숨겨지므로 $? 확인 필수)

        if [ $? -eq 0 ]
        then
            echo "$package 설치 성공"
        else
            echo "$package 설치 실패"
            exit 1
        fi
        ;;

    start|stop|restart)
        echo "$service 서비스 $action"
        systemctl "$action" "$service" || { echo "$service $action 실패"; exit 1; }  # ||: 실패 시 즉시 오류 출력 후 종료
        echo "상태 : $(systemctl is-active "$service")"
        ;;

    status)
        systemctl status "$service" --no-pager      # --no-pager: 페이저 없이 즉시 전체 출력
        ;;

    *)
        echo "Unknown Action : $action"
        echo "Usage : $0 {install|start|stop|restart|status} <service-name>"
        exit 1
        ;;
esac
```

```bash
./script/positional_example03.sh                      # 사용법 출력 후 종료
./script/positional_example03.sh run httpd            # Unknown Action : run
./script/positional_example03.sh install httpd        # 패키지 설치
./script/positional_example03.sh start httpd          # 서비스 시작
./script/positional_example03.sh status httpd         # 상태 확인
```

```bash
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; preset: disabled)
     Active: active (running) since Wed 2026-07-29 16:05:44 KST; 13s ago
   Main PID: 3100 (httpd)
```

> `systemctl status` 는 서비스가 정지 상태일 때 종료 코드 `3` 을 반환한다. 따라서 status 결과를 `$?` 로 성공/실패 판정에 쓰면 안 되고, 활성 여부는 `systemctl is-active` 로 확인한다.

### 3-4. 함수 + local 변수 + $PATH 활용

```bash
#!/bin/bash
# 함수 안에서 local로 지역 변수를 선언해 전역 오염 방지

backup_path() {
    local src="$1"                      # local : 함수 내부 전용 변수 선언
    local dest="$2"
    local base
    base=$(basename "$src")             # 경로 끝 이름만 추출
    echo "$dest/$base"
}

# PATH 환경변수 — 명령어 탐색 경로 확인
echo "현재 PATH : $PATH"

# date +%F_%T — 날짜_시각 형식 (콜론 포함)
today=$(date +%F_%T)                   # %F: YYYY-MM-DD; %T: HH:MM:SS (콜론 포함)
echo "타임스탬프 : $today"             # 예: 2026-07-30_14:30:00

# 타임스탬프 파일명으로 사용할 때 (콜론 제거 형식이 안전)
ts=$(date +%F_%H%M%S)                  # %H%M%S: 콜론 없는 형식 — 파일명에 안전
echo "파일명용 타임스탬프 : $ts"
```

### 3-5. 백업 스크립트 (경로 인자 + tar)

```bash
#!/bin/bash
# $1 : 백업 대상 디렉터리 , $2 : 백업 파일 저장 디렉터리
# 백업 파일명 : 디렉터리명_YYYY-MM-DD_HHMMSS.tar.gz

if [ $# -ne 2 ]
then
    echo "Usage : $0 <source_dir> <destination_dir>"
    exit 1
fi

src="$1"
dest="$2"

if [ ! -d "$src" ]                          # ! -d: 디렉터리가 없으면 참
then
    echo "백업 대상 디렉터리가 없습니다 : $src"
    exit 1
fi

if [ ! -d "$dest" ]
then
    echo "백업 저장 디렉터리가 없습니다 : $dest"
    exit 1
fi

base=$(basename "$src")                     # basename: 경로 끝 이름만 추출 (/temp/sqlDB/ → sqlDB)
parent=$(dirname "$src")                    # dirname: 부모 디렉터리 경로 추출 → -C 옵션에 사용
today=$(date +%F_%H%M%S)                    # 콜론(:) 없는 형식 — 파일명에 안전
backup_file="${dest}/${base}_${today}.tar.gz"

echo "백업을 시작한다 : $backup_file"

tar -czf "$backup_file" -C "$parent" "$base"
# -czf: 새 아카이브 생성·gzip 압축·파일명 지정
# -C "$parent": 상위 디렉터리 기준으로 상대경로 압축 → 절대경로 경고 방지

if [ $? -eq 0 ]                             # $?: 직전 명령 종료 코드; 0이면 성공
then
    echo "백업 성공 : $(du -h "$backup_file" | awk '{print $1}')"
    tar -tzf "$backup_file" | head -n 5      # -t: 목록 출력; head -n 5: 앞 5개만 확인
else
    echo "백업 실패"
    exit 1
fi
```

```bash
mkdir -p /temp/sqlDB /backup/mysqlDB_backup
cp -r /etc/a* /temp/sqlDB/

chmod +x ./script/positional_example04.sh
./script/positional_example04.sh /temp/sqlDB/ /backup/mysqlDB_backup/

ls -l /backup/mysqlDB_backup/                # 결과 확인
```

```bash
-rw-r--r-- 1 root root 8437  7월 29 17:13 sqlDB_2026-07-29_171339.tar.gz
```

> 저장 디렉터리(`$2`)를 백업 대상(`$1`) 하위에 지정하면 자기 자신을 다시 압축하려는 재귀가 발생한다. 대상과 저장 경로는 반드시 서로 독립된 위치로 지정한다.

---

## 4. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 4-1. 필수 검증 명령어

```bash
bash -n <스크립트>                     # 문법 사전 검사
./<스크립트>                            # 인자 없이 실행해 Usage 분기 동작 확인
echo $?                                # 종료 코드 확인 (검증 실패 시 1)
bash -x ./<스크립트> arg1 arg2          # 인자 전개 과정을 추적 (가장 강력한 진단)
echo "$#" ; echo "$@"                  # 스크립트 내부에서 인자 수·값 직접 확인
```

### 4-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. 10번째 이후 인자가 이상한 값으로 출력됨

- **증상**: `$10` 이 10번째 인자가 아니라 `첫 번째 인자 + 0` 으로 출력됨.
- **원인**: 두 자리 이상 위치 매개변수는 중괄호 없이 해석되지 않는다.
- **해결**: `${10}`, `${11}` 형태로 중괄호를 사용한다.

#### 🚨 시나리오 2. 공백이 포함된 파일명을 인자로 넘겼는데 두 개로 쪼개짐

- **증상**: `./bak.sh "my file.txt"` 실행 시 `my` 와 `file.txt` 로 분리되어 처리됨.
- **원인**: 스크립트 내부에서 `$1` 또는 `$@` 를 따옴표 없이 사용해 단어 분리 발생.
- **해결**: 인자는 항상 `"$1"`, 전체 전달은 `"$@"` 로 감싼다.

```bash
cp -- "$1" "$2"
```

#### 🚨 시나리오 3. `[[ "$1" =~ ^[0-9]+$ ]]` 가 오류를 발생시킴

- **증상**: syntax error 또는 조건이 항상 거짓.
- **원인 후보**: ① `sh script.sh` 로 실행해 Bash 확장 문법(`[[ ]]`, `=~`)이 지원되지 않음, ② 정규식을 따옴표로 감싸 문자열 리터럴로 비교됨.
- **해결**: shebang `#!/bin/bash` + `./script.sh` 실행, 정규식은 따옴표 없이 작성한다. 음수 허용이 필요하면 `^-?[0-9]+$`.

#### 🚨 시나리오 4. 백업 파일명에 콜론이 들어가 이후 처리에서 문제 발생

- **증상**: `date +%T` 로 만든 `sqlDB_2026-07-29_17:13:39.tar.gz` 파일이 `scp`·`tar` 옵션 해석이나 타 OS 전송 과정에서 오류를 유발.
- **원인**: 콜론은 리눅스 파일명으로는 유효하지만 `host:path` 문법과 충돌하고 Windows/일부 파일시스템에서 사용 불가.
- **해결**: `date +%F_%H%M%S` 처럼 콜론 없는 형식을 사용한다.

#### 🚨 시나리오 5. tar 실행 시 `Removing leading '/' from member names` 경고

- **원인**: 절대경로를 그대로 아카이브에 담아 tar가 경로를 상대경로로 변환한다는 안내.
- **해결**: `-C 상위경로 대상이름` 형태로 상대경로 기준으로 압축한다. 복원 위치를 명확히 통제할 수 있어 실무 표준이다.

```bash
tar -czf "$backup_file" -C "$(dirname "$src")" "$(basename "$src")"
```

#### 🚨 시나리오 6. `dnf install -y "$pkg" > /dev/null 2>&1` 로 출력을 숨겼는데 실패를 눈치채지 못함

- **원인**: 오류 메시지까지 함께 버려져 실패 원인이 사라짐.
- **해결**: 표준 출력만 숨기고 오류는 로그로 남기며 `$?` 를 반드시 확인한다.

```bash
dnf install -y "$pkg" > /dev/null 2>> /var/log/myscript_err.log
[ $? -ne 0 ] && { echo "설치 실패 : $pkg"; exit 1; }
```

#### 🚨 시나리오 7. 함수 안에서 `$1` 이 스크립트 인자가 아닌 값으로 나옴

- **원인**: 함수 내부의 `$1` 은 함수에 전달된 인자를 의미한다.
- **해결**: 스크립트 인자를 함수 안에서 쓰려면 전역 변수에 먼저 저장하거나 명시적으로 넘긴다.

```bash
src="$1"
do_backup "$src"
```

#### 🚨 시나리오 8. 인자 검증 없이 실행해 빈 경로로 파괴적 명령이 수행됨

- **증상**: `rm -rf "$1"/*` 에서 `$1` 이 비어 있어 `/` 하위가 대상이 됨.
- **원인**: 인자 개수·존재 검증 누락.
- **해결/예방**: 스크립트 최상단에 개수 검증과 `[ -d "$1" ]` 검증을 배치하고, 파괴적 명령 전에는 `printf '%s\n' "$1"/*` 로 대상을 먼저 확인한다.

```bash
[ $# -eq 1 ] && [ -d "$1" ] || { echo "Usage : $0 <dir>"; exit 1; }
```

---


> 📌 **핵심 요약**
> - `$0`=스크립트 이름, `$1~$9`=인자, `${10}` 이상은 중괄호 필수, `$#`=개수(`$0` 제외)
> - `"$@"`=개별 전개(다른 명령에 넘길 때 표준), `"$*"`=문자열 결합(메시지용)
> - 스크립트 첫 로직은 항상 **개수 → 형식 → 존재** 3단 검증 + Usage 출력 + `exit 1`
> - 가변 인자는 `for arg in "$@"` 또는 `shift` + `while [ $# -gt 0 ]` 로 전체를 처리
> - 인자 참조는 예외 없이 `"$1"` 형태로 따옴표를 사용해 공백·빈 값 사고를 막는다
> - 관련: 🔀 Shell Script - 조건문 (if · case) · 🔁 Shell Script - 반복문 (for · while · until) · 📦 Shell Script - 배열(Array)과 RANDOM · 🚑 Shell Script - 트러블슈팅 치트시트
