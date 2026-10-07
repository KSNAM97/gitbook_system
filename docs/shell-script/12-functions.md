# Shell Script - 함수(Functions) 심화

> **Tag:** #Linux #ShellScript #Bash #Functions #local #DynamicScoping #FUNCNAME #exportf
> **핵심 요약:** Shell 함수는 `{ ; }`(현재 쉘) 또는 `()`(subshell)로 명령 그룹에 이름을 붙인 것이다. 함수는 실행 전에 반드시 정의되어 있어야 하고, 인자는 매개변수 없이 `$1`, `$2`…로 자동 할당되며, `return`은 값을 반환하는 것이 아니라 **종료 상태 값**을 지정하는 데 쓰인다. 변수는 기본적으로 전역(global)이며 **Dynamic Scoping** 을 따른다.

---

## 1. 개요 (Overview)

`{ 명령들; }` 이나 `( 명령들 )` 로 명령 그룹을 만들면 그룹 전체가 하나의 명령처럼 실행되는데, `{ ;}` 는 **현재 쉘**에서, `()` 는 **subshell**에서 실행된다는 차이가 있다. `echo hello world | read var; echo "$var"` 는 파이프 뒤의 `read` 가 subshell에서 실행되어 `var` 값이 표시되지 않지만, `echo hello world | { read var; echo "$var" ;}` 처럼 명령 그룹으로 묶으면 `read`·`echo` 가 같은 컨텍스트에서 실행되어 값이 정상적으로 표시된다. 이 명령 그룹에 이름을 붙이면 함수가 되어 일반 명령과 동일하게 사용할 수 있으며, 보통 현재 쉘에서 실행되는 `{ ;}` 방식으로 정의한다.

**정의 문법**은 `function_name () { commands; }` 또는 bash 전용 문법인 `function function_name() { commands; }` 두 가지가 있다. `function` 키워드를 붙이면 함수명이 외부 명령이나 alias와 겹칠 때 발생할 수 있는 syntax error를 방지할 수 있다. 함수명에는 메타문자나 quotes를 사용할 수 없다. Shell은 함수에 전달된 인자를 특수 변수(`$1`, `$2`, `$3`…)에 자동으로 할당하므로, **함수 정의 시 매개변수 목록을 적지 않는다**(다른 언어의 `function foo(a, b)` 같은 문법이 없다). 정의된 함수는 `declare -f <name>`(내용 확인), `declare -F`/`compgen -A function`(전체 함수명 목록), `declare -F <name>`(존재 여부 확인), `unset -f <name>`(삭제)로 조회·관리한다.

**함수는 반드시 호출 전에 정의되어 있어야 한다.** `foo1` 함수 안에서 `foo2` 를 호출하더라도, `foo1` 이 실제로 실행되는 시점에는 `foo2` 가 이미 정의되어 있어야 한다 — 즉 `foo1`과 `foo2`를 모두 정의한 뒤에 `foo1`을 호출해야 오류 없이 동작한다. 조건문 안에서 함수를 다르게 정의하는 것도 가능하다. 예를 들어 `KSH_VERSION` 변수가 설정되어 있는지에 따라 서로 다른 `puts()` 함수를 정의해 ksh·bash 양쪽에서 동작하도록 만들 수 있다.

**인자 전달**은 외부 명령을 호출할 때 괄호를 쓰지 않는 것처럼 함수 호출 시에도 괄호를 쓰지 않는다(`myfunc arg1 arg2`). 스크립트 파일을 직접 실행할 때 `$0`은 파일 이름이지만, 함수 내부에서 `$0`은 `bash`가 된다. `$@`와 `$*`는 함수에 전달된 인자 전체를 포함하는데, quote하지 않으면 둘의 차이가 없지만 quote하면 `"$@"`는 `"$1" "$2" "$3"...`(개별 인용), `"$*"`는 `"$1c$2c$3..."`(c는 `$IFS`의 첫 글자로 하나의 문자열로 결합)로 동작이 달라진다. 함수 내부에서 자신의 이름은 `$FUNCNAME` 변수로 알 수 있다.

**결과 반환 방식**이 일반 프로그래밍 언어와 다르다. `return` 명령은 연산 결과를 반환하는 것이 아니라 `exit`처럼 **함수의 종료 상태 값**(0~255)을 지정하는 용도다. 실제 연산 결과는 `echo`로 표준 출력에 내보내고, 이를 외부 명령처럼 명령 치환(`$(함수명 인자)`)으로 받는다. `func() { expr $1 + $2; return 5; }` 를 호출하면 `expr` 결과인 `3`이 stdout으로 출력되고, `$?`로 확인하는 종료 상태 값은 `5`가 된다. 참/거짓 판단에도 종료 상태 값이 쓰인다 — `func() { return $1; }` 정의 후 `if func 0; then echo true; else echo false; fi`는 `func 0`이 종료 상태 0(참)을 반환하므로 `true`를 출력한다.

**함수 내부에서 nesting(중첩 정의)** 도 가능하다. 함수 안에 또 다른 함수를 정의할 수 있는데, shell에서 함수는 모두 전역(global) 함수가 되지만, nesting된 내부 함수는 외부 함수가 **먼저 한 번 실행되어야만** 실제로 정의된 상태가 되어 호출할 수 있다.

**변수는 기본적으로 global**이며, `source` 한 파일을 포함해 현재 스크립트 전체에서 유효하다. `local` 명령으로 함수 안에서 지역변수를 선언할 수 있으며, `local`과 `declare`는 사실상 같은 기능이지만 `local`은 함수(local scope) 밖에서는 사용할 수 없다는 차이가 있다(반대로 함수 내부에서 `declare`를 쓰면 `local`과 동일하게 동작한다). `local` 변수를 `unset` 하면 그 이름의 global 변수가 다시 보이게 된다. scope을 유지한 채 변수값만 초기화하려면 `local var=""` 처럼 빈 문자열로 초기화해야 한다. Shell은 **Dynamic Scoping**을 사용한다 — 함수 f1이 실행 중일 때 f1이 호출한 함수 f2에서 f1의 지역변수에 접근하고 값을 바꿀 수 있으며, 그 변경 사항이 f1에도 그대로 반영된다(대부분의 프로그래밍 언어가 쓰는 Lexical/Static Scoping과 반대로, f2가 f1의 지역 변수에 자유롭게 접근 가능). sh, bash, PowerShell, Emacs Lisp 등이 Dynamic Scoping을 사용한다.

새 프로세스(exec으로 생성되는 프로세스 등)에서도 함수를 쓰려면 `export -f 함수명` 으로 함수를 export해야 한다. subshell에서는 별도 설정 없이도 부모 쉘에서 정의한 함수를 그대로 사용할 수 있다.

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** Bash 기반 Linux 셸 환경 (RHEL 계열 기본 `/bin/bash`).

### Step 1. 함수 정의 표준 문법

```bash
# 기본 문법
function_name () {
    commands
}

# bash 전용 (function 키워드로 이름 충돌 방지)
function function_name() {
    commands
}
```

### Step 2. 함수 조회/삭제 명령

```bash
declare -f my_func       # 함수 정의 내용 확인
declare -F                # 전체 함수명 목록
compgen -A function       # 전체 함수명 목록 (다른 방법)
declare -F my_func         # 특정 함수 정의 여부 확인
unset -f my_func           # 함수 삭제
```

### Step 3. 인자 전달과 특수 변수

```bash
greet() {
    echo "함수 이름: $FUNCNAME"
    echo "첫 번째 인자: $1"
    echo "전체 인자(개별 인용): $@"
    echo "인자 개수: $#"
}
greet Alice Bob
```

### Step 4. return vs 연산 결과 반환

```bash
add() {
    echo $(( $1 + $2 ))   # 결과는 stdout으로 출력
    return 5               # 종료 상태 값은 5 (연산 결과와 무관)
}

result=$(add 3 4)
echo "$result"      # 7
echo "$?"            # 5
```

### Step 5. local 변수와 Dynamic Scoping

```bash
outer() {
    local msg="outer-message"
    inner
    echo "outer 종료 후: $msg"     # inner에서 바뀐 값이 반영됨
}

inner() {
    msg="inner-changed"    # local 아님 → outer의 local 변수를 그대로 덮어씀
}

outer
```

### Step 6. 함수 export (자식 프로세스에서 사용)

```bash
greet() { echo "Hello, $1"; }
export -f greet

bash -c 'greet World'   # 새 프로세스에서도 함수 사용 가능
```

### Step 7. 조건에 따라 다른 함수 정의

```bash
if [ -n "$KSH_VERSION" ]; then
    puts() { print -r -- "$*"; }
else
    puts() { printf '%s\n' "$*"; }
fi
```

### Step 8. 함수 nesting (중첩 정의)

```bash
outer() {
    inner() { echo "outer에서 정의된 inner 실행"; }
    echo "outer 실행"
    inner        # outer가 실행된 뒤에야 inner를 호출할 수 있다
}

outer      # outer 실행 → inner 정의됨 → inner 실행
inner      # outer가 이미 실행되었으므로 여기서는 정상 호출 가능
```

### Step 9. 파이프와 명령 그룹으로 컨텍스트 유지하기

```bash
# subshell에서 실행되어 var 값이 표시되지 않음
echo hello world | read var; echo "$var"

# 명령 그룹으로 묶어 같은 컨텍스트에서 실행 → 값이 표시됨
echo hello world | { read var; echo "$var"; }
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
declare -F              # 정의된 함수 목록 확인
declare -f <함수명>       # 함수 정의 내용 확인
echo $?                  # 함수의 종료 상태 값(return) 확인
```

### 3-2. 트러블슈팅 시나리오

#### 시나리오 1. 함수를 호출했는데 `command not found` 오류

- **원인:** 함수가 호출되는 시점에 아직 정의되지 않았다(정의 코드가 호출 코드보다 뒤에 있음).
- **해결:** 함수 정의를 스크립트 상단으로 옮기거나, 모든 함수 정의를 마친 뒤에 실행 로직을 배치한다.

#### 시나리오 2. 함수에서 계산한 값을 받으려 했는데 빈 값이거나 상태 코드만 들어옴

- **원인:** `return`은 종료 상태 값(0~255) 전용이라 연산 결과를 담을 수 없다는 점을 착각함.
- **해결:** 연산 결과는 `echo`로 출력하고, 호출 측에서 `result=$(함수명 ...)` 형태로 명령 치환을 사용해 받는다.

#### 시나리오 3. 새로 띄운 서브 프로세스(`bash -c`, 백그라운드 스크립트 등)에서 함수를 호출하니 `command not found`

- **원인:** 함수는 기본적으로 현재 쉘에만 존재하며 자식 프로세스로 자동 상속되지 않는다.
- **해결:** `export -f 함수명` 으로 함수를 export한 뒤 자식 프로세스를 실행한다.

#### 시나리오 4. 함수 안에서 만든 변수가 스크립트 전체에 영향을 줌

- **원인:** `local`을 붙이지 않으면 함수 내부 변수도 기본적으로 global이다.
- **해결:** 함수 내부 전용 변수는 반드시 `local var=값` 으로 선언한다.

#### 시나리오 5. nesting된 내부 함수를 스크립트 시작 부분에서 바로 호출했더니 오류가 남

- **원인:** nesting된 함수는 외부 함수가 최소 한 번 실행되어야 실제로 정의된 상태가 된다.
- **해결:** 외부 함수를 먼저 한 번 호출해 내부 함수가 정의되게 한 뒤, 그다음부터 내부 함수를 직접 호출한다.

#### 시나리오 6. 파이프 뒤에서 변수를 채웠는데 파이프 앞뒤로 값이 사라짐

- **원인:** 파이프로 연결된 명령은 각각 별도의 subshell에서 실행되므로, subshell 안에서 바뀐 변수 값은 파이프 밖으로 전달되지 않는다.
- **해결:** `{ ; }` 명령 그룹으로 파이프 뒤의 명령들을 묶어 같은 컨텍스트에서 실행되게 한다.

---

>  **핵심 요약**
> - 함수 정의는 매개변수 없이 `name() { ...; }`, 인자는 `$1`,`$2`...로 자동 할당
> - `return`은 종료 상태 값 전용 — 연산 결과는 `echo` + 명령 치환(`$(func ...)`)으로 받는다
> - 변수는 기본 global, `local`로 지역화, Shell은 Dynamic Scoping(자식 함수가 부모 local 변수 접근·수정 가능)
> - 새 프로세스에서 함수를 쓰려면 `export -f`, 함수는 호출 전에 반드시 정의돼 있어야 함
> - 관련: Shell Script - 변수와 환경변수 (커널·쉘 개념 포함) · Shell Script - exit 상태와 test 명령 · Shell Script - 위치 매개변수 (Positional Parameters)
