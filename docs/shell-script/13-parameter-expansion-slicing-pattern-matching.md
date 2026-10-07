# Shell Script - 변수 기본값 · 문자열/배열 슬라이싱 · Pattern Matching 심화

> **Tag:** #Linux #ShellScript #Bash #ParameterExpansion #StringSlicing #ArraySlicing #GlobPattern #CharacterClass #ExitStatus
> **핵심 요약:** 변수는 `${VAR:-value}` 계열 확장자로 미선언/NULL 상태에 따라 기본값을 반환·대입·에러 처리할 수 있고, `${string:idx:len}` 문법으로 문자열·배열을 슬라이싱할 수 있다. Glob 패턴(`*`, `?`, `[...]`)과 POSIX Character Class(`[[:alpha:]]` 등)는 파일명 매칭·case문·매개변수 확장 전반에서 재사용되며, pipe로 연결된 명령의 종료 상태 값은 기본적으로 **마지막 명령 것만** 반영된다.

---

## 1. 개요 (Overview)

### 변수 기본값 설정 (Parameter Expansion Defaults)

쉘 변수는 세 가지 상태를 가질 수 있다: ① **선언되지 않음(unset)**, ② **선언되었지만 값이 NULL**, ③ **NULL이 아닌 값으로 선언됨**. 이 상태에 따라 기본값을 다르게 처리하는 확장자가 제공된다.

`-`/`:-` 계열은 변수가 선언되지 않았을 때(또는 `:`가 붙으면 NULL일 때도) 대체값을 **반환만** 하고 원본 변수는 바꾸지 않는다. `${VAR-value}`는 `VAR`이 선언되지 않은 경우에만 `value`를 반환하고, `${VAR:-value}`는 선언되지 않았거나 NULL인 경우 모두 `value`를 반환한다. `=`/`:=` 계열은 반환과 동시에 **변수 자체를 그 값으로 초기화**한다는 점이 `-`/`:-`와의 핵심 차이다. `${VAR=value}`는 미선언 시, `${VAR:=value}`는 미선언 또는 NULL일 때 `VAR`을 `value`로 대입하고 `value`를 반환한다.

`+`/`:+` 계열은 반대로 변수가 **선언된 경우에** 값을 반환한다. `${VAR+value}`는 `VAR`이 선언만 되어 있으면(NULL이어도) `value`를 반환하고, `${VAR:+value}`는 `VAR`이 NULL이 아닌 값으로 선언된 경우에만 `value`를 반환한다. `?`/`:?` 계열은 기본값을 채우는 대신 **에러를 발생**시킨다. `${VAR?ERR}`은 `VAR`이 선언되지 않은 경우 표준 에러로 `ERR`을 출력하고, `${VAR:?ERR}`은 선언되지 않았거나 NULL인 경우에도 동일하게 에러를 출력한다(스크립트 초반에 필수 환경변수가 비어있는지 검증하는 용도로 유용).

### 문자열/배열 슬라이싱 (Parameter Slicing)

`${변수:인덱스:길이}` 문법으로 문자열이나 배열을 슬라이싱한다. 인덱스와 길이 모두 음수를 허용하며, 음수는 뒤에서부터 센 위치를 의미한다(단, `${변수: -N}`처럼 콜론 뒤 공백을 넣어야 `${변수:-기본값}` 문법과 헷갈리지 않는다). 예를 들어 `string=01234567890abcdefgh`일 때 `${string:7}`은 인덱스 7부터 끝까지(`7890abcdefgh`), `${string:7:2}`은 인덱스 7부터 2글자(`78`), `${string:7:-2}`은 인덱스 7부터 뒤에서 2글자를 제외한 나머지(`7890abcdef`), `${string: -7}`은 뒤에서 7번째부터 끝까지(`bcdefgh`)를 반환한다.

함수·스크립트에 전달된 매개변수 배열(`$@`)은 IFS로 구분된 진짜 배열로 취급되지만(공백으로 합쳐진 하나의 문자열로 취급하는 `$*`와 다름), `$@` 자체를 직접 슬라이싱하기는 어렵기 때문에 다른 배열 변수에 복사한 뒤 슬라이싱해야 한다: `var_all_arr=("$@")` 로 복사한 뒤 `${var_all_arr[@]:2}` 처럼 배열 슬라이싱 문법을 적용한다.

### JOIN 함수 구현

Shell에는 배열 요소를 구분 기호로 결합하는 내장 `join` 함수가 없어 아래처럼 직접 구현해서 사용한다.

```bash
join_ws() { local d=$1 s=$2; shift 2 && printf %s "$s${@/#/$d}"; }
```

동작 원리는 다음과 같다. ① `local d=$1 s=$2`로 첫 번째 인자(구분자)를 `d`, 두 번째 인자(첫 배열 요소)를 `s`에 저장한다. ② `shift 2`로 이미 저장한 두 인자를 제거해 나머지 배열 요소만 남긴다. ③ `${@/#/$d}`는 패턴 치환으로 남은 각 인자의 맨 앞(`#`)에 구분자 `$d`를 붙인다. ④ `"$s${@/#/$d}"`는 첫 요소 `$s`를 맨 앞에 붙이고 그 뒤에 "구분자가 붙은 나머지 요소들"을 이어 붙이는 방식으로, 맨 첫 요소에만 구분자가 붙지 않도록 만든다.

```bash
arr=("apple" "banana" "cherry")
echo -e "$(join_ws ',' "${arr[@]}")"      # apple,banana,cherry
echo -e "$(join_ws ' || ' "${arr[@]}")"   # apple || banana || cherry
```

### Pattern Matching (Glob 문자와 Character Class)

`*`(0글자 이상), `?`(정확히 1글자), `[...]`(대괄호 안 문자 집합 중 1글자) 세 글롭 문자는 파일명 매칭뿐 아니라 case문, 매개변수 확장(`${var#pattern}` 등) 전반에서 재사용된다. Bracket 표현식은 `[XYZ]`(X, Y, Z 중 하나), `[X-Z]`(범위), `[^...]`/`[!...]`(NOT, 괄호 안 문자를 제외한 1글자)로 세분화된다.

**POSIX Character Class**는 `[[:class:]]` 형태로 비슷한 의미의 문자를 그룹화한 것이다. 대표적으로 `[[:alnum:]]`=`[A-Za-z0-9]`(영문자+숫자), `[[:alpha:]]`=`[A-Za-z]`(영문자), `[[:digit:]]`=`[0-9]`(숫자), `[[:lower:]]`/`[[:upper:]]`(소문자/대문자), `[[:space:]]`(모든 공백 문자), `[[:punct:]]`(문장부호), `[[:xdigit:]]`=`[0-9a-fA-F]`(16진수 자릿수)가 있다. `[[:alnum:]]`처럼 클래스를 bracket 표현식 안에 넣어 사용한다(`[[:alnum:]]` 자체가 이미 대괄호 한 겹을 포함하므로 실제로는 이중 대괄호처럼 보인다).

**Extended Pattern(확장 패턴)**은 bash 전용 기능으로, 프롬프트나 쉘 함수에서는 기본 사용 가능하지만 스크립트 파일 실행 시에는 기본적으로 비활성화되어 있어 `shopt -s extglob`로 켜야 한다. `?(패턴)`(0회 또는 1회 매칭), `*(패턴)`(0회 이상), `+(패턴)`(1회 이상), `@(패턴)`(정확히 1회), `!(패턴)`(패턴과 일치하지 않는 것과 매칭)로 구성된다.

### test / `[[ ]]` 데이터 타입 심화

Shell에는 숫자·문자열을 구분하는 데이터 타입이 따로 없다 — 모든 것은 문자열이고, 산술 연산은 확장(`$(( ))`)이나 별도 명령(`expr`, `let`)으로만 제공된다. `test` 와 `[` 는 완전히 같은 명령이다(`test -d /etc`와 `[ -d /etc ]`는 동일). 주의할 점 세 가지: ① 변수 비교 시 quote 처리가 필요하다 — `[ -n $AA ]`처럼 quote 없이 빈 변수를 비교하면 `-n`이 값 없는 인수로 취급되어 예상과 다르게 항상 참이 될 수 있으므로 `[ -n "$AA" ]`로 quote해야 한다. ② 숫자 비교에는 문자열 연산자(`<`, `>`)를 쓸 수 없다 — `[ 100 \> 2 ]`는 사전순 문자열 비교라 원하는 결과가 안 나올 수 있고, 숫자 크기 비교는 `-gt`, `-lt` 등을 써야 한다. ③ 배열 비교 시 `${arr[*]}`를 써야 한다 — quote하지 않으면 `${arr[@]}`와 `${arr[*]}`가 동일하게 IFS로 단어 분리되지만, quote하면 `"${arr[@]}"`(개별 인용)와 `"${arr[*]}"`(하나로 결합)의 의미가 달라지므로 배열 전체를 통째로 비교할 때는 `"${arr[*]}"`를 사용해야 한다. AND/OR/NOT은 `test` 자체의 `-a`/`-o` 옵션이나 쉘 메타문자 `&&`/`||`로 표현할 수 있고, NOT은 `!`를 test 안팎 어디에든 쓸 수 있다. `[[ ]]`는 `[ ]`의 확장판으로 명령어가 아니라 **쉘 키워드**이기 때문에 명령어이기에 생기는 제약(quote 필요성 등) 없이 더 편하게 쓸 수 있다(`[[ a < b ]]` 처럼 문자열 비교 연산자를 이스케이프 없이 그대로 사용 가능).

### Exit Status 심화

`if`, `while`, `until`, `&&`, `||`는 모두 직전 명령의 종료 상태 값(`$?`, 0=성공/참, 그 외=실패/거짓)으로 참·거짓을 판단한다. **pipe로 연결된 명령**은 기본적으로 **마지막 명령의 종료 상태 값**만 결과로 쓰인다 — `command1 arg1 arg2 | sed -n '/pattern/,/^$/p'`는 `command1`이 실패하더라도 `sed`가 항상 참을 반환하므로 전체 파이프 결과는 항상 0이 된다. 파이프 중간 명령의 실패를 감지하려면 `set -o pipefail` 옵션(파이프에 연결된 명령 중 하나라도 실패하면 전체가 실패로 처리됨, `sh`에서는 사용 불가)을 켜거나, 아예 명령을 분리해서 `command1 > tmpfile; status=$?; sed ... tmpfile; echo $status` 형태로 각 단계의 상태를 따로 저장한다. 존재하지 않거나 NULL인 변수는 quote하지 않으면(예: `[ false ]`) 참으로 판단될 수 있다는 점도 주의해야 한다. 대입 연산(`=`, `+=`)의 종료 상태 값은 기본적으로 항상 0이지만, 명령 치환과 함께 쓰이면(`var=$(cmd)`) 그 명령 치환의 종료 상태 값이 그대로 반영된다.

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** Bash 기반 Linux 셸 환경 (RHEL 계열 기본 `/bin/bash`).

### Step 1. 변수 기본값 확장자 요약표

| 확장자 | 조건 | 동작 |
|---|---|---|
| `${VAR-value}` | VAR 미선언 | value 반환 (VAR 값 변경 없음) |
| `${VAR:-value}` | VAR 미선언 또는 NULL | value 반환 (VAR 값 변경 없음) |
| `${VAR=value}` | VAR 미선언 | VAR을 value로 대입 후 반환 |
| `${VAR:=value}` | VAR 미선언 또는 NULL | VAR을 value로 대입 후 반환 |
| `${VAR+value}` | VAR 선언됨(NULL 포함) | value 반환 |
| `${VAR:+value}` | VAR이 NULL 아닌 값으로 선언됨 | value 반환 |
| `${VAR?ERR}` | VAR 미선언 | STDERR로 ERR 출력 |
| `${VAR:?ERR}` | VAR 미선언 또는 NULL | STDERR로 ERR 출력 |

```bash
unset NAME
echo "${NAME:-guest}"        # guest (NAME은 그대로 unset)
echo "${NAME:=guest}"        # guest (NAME이 guest로 대입됨)
echo "$NAME"                  # guest

: "${REQUIRED:?REQUIRED 환경변수가 필요합니다}"   # 없으면 즉시 에러로 스크립트 중단
```

### Step 2. 문자열 슬라이싱 실전 패턴

```bash
string=01234567890abcdefgh
echo "${string:7}"        # 7890abcdefgh
echo "${string:7:2}"      # 78
echo "${string:7:-2}"     # 7890abcdef
echo "${string: -7}"      # bcdefgh   (콜론 뒤 공백 필수)
echo "${string: -7:2}"    # bc
```

### Step 3. `$@` 배열 슬라이싱

```bash
#!/bin/bash
var_all_arr=("$@")
echo "${var_all_arr[@]:2}"    # 인덱스 2부터 끝까지
```

### Step 4. JOIN 함수 재사용 스니펫

```bash
join_ws() { local d=$1 s=$2; shift 2 && printf %s "$s${@/#/$d}"; }

arr=("apple" "banana" "cherry")
echo -e "$(join_ws ',' "${arr[@]}")"
echo -e "$(join_ws '\n' "${arr[@]}")"
```

### Step 5. Glob / Character Class / Extended Pattern

```bash
ls file[0-9].txt            # file0.txt ~ file9.txt
ls file[!0-9]*               # 숫자로 시작하지 않는 file*
ls [[:upper:]]*               # 대문자로 시작하는 파일

shopt -s extglob
ls +([0-9]).log               # 숫자가 1회 이상 반복되는 이름의 .log 파일
```

### Step 6. test / `[[ ]]` 표준 패턴

```bash
[ -n "$AA" ]; echo $?          # quote 필수 (변수 비교)
[ 100 -gt 2 ]; echo $?          # 숫자 비교는 -gt/-lt
[[ "${AA[*]}" = "${BB[*]}" ]]   # 배열 전체 비교는 *
[[ a < b ]]; echo $?            # [[ ]] 는 키워드라 quote 부담 적음
```

### Step 7. pipefail로 파이프 오류 감지

```bash
(
    set -o pipefail
    command1 arg1 arg2 | sed -n '/pattern/,/^$/p'
)
echo $?    # command1 실패 시에도 비정상 상태 값 반영
```

### Step 8. 명령을 분리해 파이프 중간 상태 값 확보 (`sh` 호환)

```bash
# pipefail을 쓸 수 없는 sh 환경에서의 대안
command1 arg1 arg2 > tmpfile
status=$?

sed -n '/pattern/,/^$/p' tmpfile
echo "$status"     # command1 자체의 성공/실패 여부
```

### Step 9. Character Class 조합 실전 패턴

```bash
ls [[:digit:]]*.log            # 숫자로 시작하는 .log 파일
ls *[[:space:]]*                # 파일명에 공백이 포함된 항목
grep -E '^[[:alpha:]]+$' file   # 영문자로만 이루어진 줄만 필터링
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
echo "${VAR:-<미설정>}"     # 값 확인하되 없으면 표시용 기본값만
declare -p VAR              # 변수의 선언/NULL 상태 확인
echo $?                     # 직전 명령(또는 pipefail 파이프)의 종료 상태 값
```

### 3-2. 트러블슈팅 시나리오

#### 시나리오 1. `${VAR:-value}`를 썼는데 VAR에 값이 저장되길 기대했다가 안 됨

- **원인:** `-`/`:-` 계열은 값을 반환만 할 뿐 VAR 자체를 바꾸지 않는다.
- **해결:** VAR에 실제로 값을 대입하고 싶다면 `=`/`:=` 계열(`${VAR:=value}`)을 사용한다.

#### 시나리오 2. 파이프 명령 중간 것이 실패했는데 스크립트가 계속 정상 진행됨

- **원인:** pipe의 종료 상태 값은 기본적으로 마지막 명령 것만 반영된다.
- **해결:** `set -o pipefail`을 켜거나, 파이프를 분리해 중간 명령의 상태를 별도로 저장한다.

#### 시나리오 3. `${string: -3}` 처럼 음수 인덱스를 썼는데 `${string:-3}` 문법으로 오인되어 기본값 확장이 발동됨

- **원인:** 콜론 뒤에 공백 없이 마이너스를 붙이면 파서가 기본값 확장 문법으로 해석한다.
- **해결:** 슬라이싱에서 음수 인덱스를 쓸 때는 `${string: -3}` 처럼 콜론 뒤에 반드시 공백을 넣는다.

#### 시나리오 4. `[[:alpha:]]` 클래스가 매칭되지 않음

- **원인:** Character Class는 반드시 `[...]` bracket 표현식 안에서 사용해야 한다(`[[:alpha:]]` 자체가 `[` + `[:alpha:]` + `]` 구조).
- **해결:** 단독으로 `:alpha:` 만 쓰지 말고 `[[:alpha:]]` 전체를 그대로 사용한다.

#### 시나리오 5. 배열 두 개를 비교했는데 원소 순서만 같으면 항상 같다고 나옴

- **원인:** `${array[@]}`를 quote 없이 비교하면 IFS로 단어 분리되어 개별 원소끼리 비교되는 것처럼 동작해 버린다.
- **해결:** 배열 전체를 하나의 문자열로 비교하려면 `[ "${AA[*]}" = "${BB[*]}" ]` 처럼 quote한 `*`를 사용한다.

#### 시나리오 6. `shopt -s extglob` 없이 확장 패턴을 썼는데 그냥 문자 그대로 매칭됨

- **원인:** Extended Pattern(`?()`, `*()`, `+()`, `@()`, `!()`)은 bash 스크립트 파일 실행 시 기본적으로 비활성화되어 있다.
- **해결:** 스크립트 상단에서 `shopt -s extglob`으로 명시적으로 활성화한 뒤 사용한다.

---

>  **핵심 요약**
> - `-`/`:-`(반환만) vs `=`/`:=`(반환+대입) vs `+`/`:+`(선언된 경우) vs `?`/`:?`(에러) — 네 계열로 변수 기본값 처리
> - `${string:idx:len}` 슬라이싱은 인덱스/길이에 음수 허용(뒤에서부터), `$@`는 배열로 복사한 뒤 슬라이싱
> - JOIN 함수는 `${@/#/$d}` 패턴 치환으로 각 요소 앞에 구분자를 붙이는 방식으로 직접 구현
> - Glob(`*?[]`)·Character Class(`[[:alpha:]]` 등)·Extended Pattern(`shopt -s extglob`)은 파일명 매칭·case문·매개변수 확장 전반에서 재사용
> - pipe의 종료 상태 값은 기본적으로 마지막 명령 것만 반영 — 중간 명령 실패 감지에는 `pipefail`
> - 관련: Shell Script - Metacharacters (메타문자) · Shell Script - exit 상태와 test 명령 · Shell Script - 배열(Array)과 RANDOM · Shell Script - 위치 매개변수 (Positional Parameters)
