# 🐚 Shell Script - 변수와 환경변수 (커널·쉘·쉘스크립트 개념 포함)

> **Tag:** #Linux #ShellScript #Bash #변수 #환경변수 #지역변수 #전역변수 #PATH #export
> **핵심 요약:** 커널은 하드웨어를 직접 제어하는 OS의 핵심이고, 쉘은 사용자의 명령을 커널에 전달하는 해석기(Command Interpreter)이며, 쉘 스크립트는 이 명령들을 파일로 묶어 자동 실행하는 프로그램이다. Bash 변수는 기본적으로 **모든 값을 문자열로 저장**하며, `export` 로 승격해야 **환경 변수**가 되어 자식 프로세스에 전달된다.

---

## 1. 📖 개요 (Overview)

커널은 CPU·메모리·디스크·네트워크 같은 하드웨어를 직접 제어하는 OS의 심장이고, 쉘은 사용자의 입력을 받아 커널에 전달하는 번역기 역할이다. 쉘 스크립트는 쉘이 실행할 명령들을 파일로 묶어 자동화한 것이다. 전체 동작 흐름은 `사용자 입력 → 쉘 → 커널 → 하드웨어 → 결과를 쉘이 화면에 출력` 이다. 대표 쉘은 4종으로, `sh`(POSIX 표준, 단순·호환성 높음), `bash`(리눅스 기본, 기능 풍부), `zsh`(자동완성·플러그인 강력), `ksh`(상업용 유닉스 계열, sh과 bash 중간 기능)가 있다. `/bin` 과 `/etc` 는 역할이 다른데, `/bin`은 실제 실행되는 쉘 프로그램(`/bin/bash`, `/bin/csh` 등)이고, `/etc`는 쉘의 환경을 설정하는 파일(`/etc/profile`, `/etc/bashrc`, `/etc/shells`)이다.

사용자 로그인 시에는 `사용자 로그인 → /etc/passwd 확인 → 지정된 쉘(예: /bin/bash) 실행 → /etc/profile 읽음 → /etc/bashrc 읽음 → 사용자 명령 입력` 순서로 진행된다. `/etc/passwd` 의 7번째 필드가 로그인 시 적용되는 기본 쉘이다. `chsh <user>` 로 기본 쉘을 변경할 수 있으나, **이미 로그인된 세션의 `$SHELL` 값은 즉시 바뀌지 않고 재로그인 시 반영**된다. 즉시 적용하려면 `exec /bin/bash` 를 사용한다. 추가 쉘 설치는 `dnf install -y csh zsh tcsh` 후 `/etc/shells` 에 자동 등록된다.

Bash는 기본적으로 모든 변수 값을 **문자열(String)** 로 저장하기 때문에 `num1=10; num2=20; echo $num1+$num2` 를 실행하면 `30`이 아니라 `10+20`이 출력된다. 실제 산술 계산을 하려면 `$(( ))`, `expr`, `let` 같은 별도 구문이 필요하다. 변수명 규칙은 문자(a-z, A-Z)·숫자(0-9)·언더바(_)만 사용, 첫 글자는 숫자 불가, 공백·특수문자(`!@$%` 등) 불가, 대/소문자 구분, if/then/fi 같은 예약어는 변수명 사용 불가이다. `=` 앞뒤에 공백이 있으면 오류가 난다(`name = "kim"` 은 오류, `name="kim"` 은 정상). 값에 공백이 포함되면 반드시 겹따옴표로 감싸야 한다 (`arr="A B C D"`). 겹따옴표 없이 `arr=A B C D` 를 실행하면 B가 명령어로 잘못 해석되어 오류가 발생한다.

지역변수(일반 변수)는 **현재 쉘에서만 유효**하고 자식 프로세스에 전달되지 않는다. 반면 전역변수(환경 변수)는 `export` 로 환경에 등록된 변수로, **현재 쉘과 그 쉘이 실행하는 모든 자식 프로세스에 자동으로 전달**된다. 이를 검증하려면 Bash에서 변수를 만든 뒤 `tcsh` 같은 자식 쉘을 실행해 `echo $변수명` 으로 확인하면 되는데, 일반 변수는 `Undefined variable` 오류가 나고, 환경 변수는 값이 그대로 출력된다. 환경 변수 삭제는 `unset <변수명>` 으로 한다. 대표 환경 변수로는 `PATH`(명령어 탐색 경로), `HOME`(홈 디렉터리), `USER`(로그인 사용자), `SHELL`(로그인 시 기본 쉘), `LANG`/`LC_*`(언어·인코딩)가 있다.

`PATH` 는 쉘이 명령어(실행 파일)를 찾는 디렉터리 목록이다. `PATH=/temp` 처럼 기존 경로를 덮어쓰면 `ls`, `date`, `cp` 등 대부분의 명령어를 **찾을 수 없다(command not found)** 는 오류가 발생한다. 복구 방법은 반드시 **절대 경로로 직접 재입력**해야 한다. `export PATH=/root/.local/bin:/root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin` 형태로 원래 값을 알고 있어야만 복구 가능하므로, 실무에서는 변경 전 `echo $PATH` 로 반드시 백업해 둔다. `PATH=값` 은 현재 쉘에서만 적용되고, `export PATH=값` 은 자식 프로세스에도 전달된다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 변수선언 및 참조 기본 문법

```bash
# 변수선언 (= 앞뒤 공백 금지)
<변수명>=<값>

# 변수 참조
echo $<변수명>
echo "$<변수명>"          # 공백 포함 값은 겹따옴표 권장

# 변수명과 뒤 문자열을 명확히 구분해야 할 때
echo "${<변수명>}추가문자열"
```

### 2-2. 명령 결과를 변수에 저장 (명령 치환)

```bash
# 옛날 방식(백틱) - 비권장
result=`<명령어>`

# 현재 방식 - 권장
result=$(<명령어>)

echo "$result"

# 실전 예제
today=$(date)              # date 명령 결과 저장 → "2026. 07. 30. (수) ..."
today=$(date +%F)          # 날짜만 저장          → "2026-07-30"
to_cal=$(cal)              # 달력 출력 저장
cnt=$(ls -l | wc -l)      # ls -l 줄 수 저장 (파일 개수 계산 등에 활용)

echo "$today"              # 저장된 날짜 출력
echo "$cnt"                # 저장된 줄 수 출력
```

> - `date` 만 입력하면 전체 날짜·시간 출력. `date +%F` → `YYYY-MM-DD`, `date +%T` → `HH:MM:SS`
> - `cal` : 현재 달의 달력 출력. `cal 8 2026` 처럼 특정 월·연도 지정 가능
> - `wc -l` : 줄 수(lines) 계산. `wc -w` : 단어 수, `wc -c` : 바이트 수

### 2-3. 변수 계산 (문자열이 아닌 산술 연산)

```bash
a=10
b=20

echo $((a + b))          # 산술 확장 : 30 출력
sum=$((a + b))            # 변수에 저장
echo "합계 : $sum"
```

### 2-4. 환경 변수 등록/해제

```bash
# 방법 1 : 변수 선언 후 export
NAME=lee
export NAME

# 방법 2 : 선언과 동시에 export
export NAME=lee

# 확인
env | grep NAME

# 삭제
unset NAME

# 읽기 전용(변경·삭제 불가)
readonly CONST_VAR=값
declare -r CONST_VAR=값   # 동일한 효과
```

### 2-5. PATH / HOME 등 주요 환경 변수 조작

```bash
echo $PATH                 # 현재 PATH 확인 (반드시 사전에 백업)
export PATH=<절대경로1>:<절대경로2>:...   # 변경 시 기존 경로 반드시 포함

echo $HOME                 # 현재 홈 디렉터리
export HOME=<새 경로>       # 홈 디렉터리 변경 (cd 단독 입력 시 이동 위치가 바뀜)

echo $USER                 # 현재 로그인한 사용자 이름
echo $LANG                 # 현재 언어 및 인코딩 설정 (예: ko_KR.UTF-8)
```

### 2-6. 실전 변수 선언 패턴 예제

```bash
# 기본 변수 선언 및 출력
name="kim"
age=25
dir="/home/user1"
msg="안녕하세요, $name 님"

echo $name             # kim
echo $age              # 25
echo $dir              # /home/user1
echo "$msg"            # 안녕하세요, kim 님

# 다른 명칭 변수 사용 예
NAME="lee"
CITY="seoul"
PATH2="/usr/local/bin"      # 별도 경로 변수 (PATH와 구분)
arr="apple banana cherry"  # 공백 포함 값은 겹따옴표

echo $NAME
echo $CITY
echo $PATH2

# to_cal : cal 명령 결과 변수에 저장 후 사용
to_cal=$(cal)
echo "$to_cal"         # 달력 출력
```

### 2-7. PS1 프롬프트 커스터마이징

```bash
echo $PS1                  # 기본값 예 : [\u@\h \W]\$

# 임시 변경 (현재 세션만)
PS1='[\t \u@\h \W]\$ '

# 영구 반영 (.bashrc 에 저장)
vi ~/.bashrc
# PS1='[\u@\h \W]\$ '   라인 추가
source ~/.bashrc
```

**PS1 이스케이프 시퀀스 표**

| 시퀀스 | 의미 |
|---|---|
| `\u` | 현재 사용자 이름 |
| `\h` | 호스트 이름 첫 부분 |
| `\W` | 현재 작업 디렉터리의 마지막 이름 |
| `\w` | 현재 작업 디렉터리 전체 경로 |
| `\$` | root는 `#`, 일반 사용자는 `$` |
| `\t` | 현재 시간 (HH:MM:SS) |
| `\d` | 현재 날짜 |

### 2-8. date 명령 형식 지정자 요약

```bash
date +%Y            # 연도 4자리
date +%y            # 연도 2자리
date +%m             # 월
date +%d             # 일
date +%F             # YYYY-MM-DD (ISO 표준)
date +%T             # HH:MM:SS
date +"%Y-%m-%d %H:%M:%S"
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
echo $0                     # 현재 사용 중인 쉘 확인
echo $SHELL                 # 로그인 시 적용된 기본 쉘
echo $PATH                  # 명령어 탐색 경로
echo $HOME                  # 홈 디렉터리
env                          # 전체 환경 변수 조회
env | grep <변수명>          # 특정 환경 변수 확인
cat /etc/passwd | grep <user>   # 계정의 기본 쉘 확인
cat /etc/shells              # 시스템에 등록된 쉘 목록

set                          # 현재 쉘의 모든 변수·함수 목록 출력
set -e                       # 오류 발생 즉시 스크립트 종료
set -x                       # 실행 명령을 화면에 추적 출력 (디버그 모드)
set +x                       # 디버그 모드 해제
bash -x ./script.sh          # 각 명령 실행 전 라인 출력 (전체 추적)
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. 변수 계산 결과가 문자열로 이어붙여 출력됨

- **증상:** `num1=10; num2=20; echo $num1+$num2` → `10+20` 출력 (30이 아님).
- **원인:** Bash 변수는 기본적으로 모든 값을 문자열로 취급하기 때문.
- **해결:**
  ```bash
  echo $((num1 + num2))     # 산술 확장 사용 → 30
  ```

#### 🚨 시나리오 2. 값에 공백이 있는 변수를 겹따옴표 없이 선언하여 오류 발생

- **증상:** `arr=A B C D` 실행 시 `bash: B: 명령을 찾을 수 없습니다` 오류.
- **원인:** 쉘이 공백을 기준으로 `arr=A` 를 실행한 뒤, `B`를 새로운 명령어로 인식하고 `C D`를 그 인자로 전달.
- **해결:**
  ```bash
  arr="A B C D"              # 값에 공백이 있으면 반드시 겹따옴표로 감쌈
  ```

#### 🚨 시나리오 3. 환경 변수로 등록했는데 `env` 에서 확인되지 않음

- **증상:** `NAME=lee` 선언 후 `env | grep NAME` 결과가 비어 있음.
- **원인:** `export` 를 하지 않아 일반(로컬) 변수 상태로만 존재.
- **해결:**
  ```bash
  export NAME          # 이미 선언된 변수를 환경 변수로 승격
  # 또는
  export NAME=lee      # 선언과 동시에 환경 변수 등록
  ```

#### 🚨 시나리오 4. `PATH` 변경 후 대부분의 명령어가 실행되지 않음

- **증상:** `PATH=/temp` 실행 후 `ls`, `date`, `cp` 등 모든 명령에서 `명령을 찾을 수 없습니다` 오류.
- **원인:** 기존 PATH 값이 통째로 덮어써져 명령어 탐색 경로가 사라짐.
- **해결:**
  ```bash
  export PATH=/root/.local/bin:/root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
  ```
  ⚠️ 사전에 `echo $PATH` 로 원본 값을 반드시 백업해 두어야 복구할 수 있다. PATH 변경 실습은 테스트 환경에서만 진행할 것.

#### 🚨 시나리오 5. `HOME` 변경 후 `cd` 단독 입력 시 예상과 다른 경로로 이동

- **증상:** `HOME=/etc/ssh` 로 변경 후 `cd` 만 입력하면 `/root` 가 아니라 `/etc/ssh` 로 이동.
- **원인:** `cd` 는 `HOME` 환경 변수 값을 기준으로 이동하며, `HOME` 값이 바뀌면 이동 위치도 바뀐다.
- **해결:**
  ```bash
  export HOME=/root        # 원래 값으로 복구
  ```

#### 🚨 시나리오 6. 자식 쉘(tcsh 등)에서 일반 변수를 참조하면 오류

- **증상:** Bash에서 `SOL=desk` 선언 후 `tcsh` 실행, `echo $SOL` 입력 시 `SOL: Undefined variable.` 오류.
- **원인:** 일반 변수는 현재 쉘에서만 유효하고 자식 프로세스에 전달되지 않는다.
- **해결:**
  ```bash
  export SOL=desk           # 환경 변수로 등록해야 자식 쉘에서도 참조 가능
  ```

---

> 📌 **핵심 요약**
> - 커널 → 쉘 → 쉘 스크립트 흐름을 구분해서 이해할 것
> - Bash 변수는 기본적으로 **문자열**, 계산은 `$(( ))` / `expr` / `let` 사용
> - 일반 변수는 현재 쉘 전용, **환경 변수(`export`)만 자식 프로세스에 전달**
> - `PATH`, `HOME` 변경은 되돌릴 값을 반드시 사전에 확보해 둘 것
> - 관련: Metacharacters (메타문자) · expr · let (산술 연산) · exit 상태와 test 명령
