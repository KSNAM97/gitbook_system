# Shell Script - 기초 문법 재정리와 read 입력

> **Tag:** #Linux #ShellScript #Bash #Basics #WordSplitting #Globbing #read #FileDescriptor
> **핵심 요약:** Shell은 공백으로 명령과 인자를 구분하고, 참/거짓 판단은 오직 종료 상태 값(`$?`, 0=참)으로만 이루어진다. `[` 는 인수로 `]`를 요구하는 명령이라 앞뒤 공백이 필수이며, 대입 연산(`AA=10`)만 예외적으로 공백을 두면 안 된다. `read`는 사용자 키보드 입력과 파일 내용을 변수로 읽어들이는 표준 명령으로, `while read line < file` 패턴으로 파일을 한 줄씩, 여러 변수를 지정해 컬럼 단위로도 읽을 수 있다.

---

## 1. 개요 (Overview)

Shell의 기본 역할은 사용자에게 명령을 입력받아 실행하는 것이며, 실제 명령이 실행되기 전에 **해석(파싱)** 단계를 거친다. 이 해석 단계에서 파일명·공백·따옴표·메타문자 등이 처리된다. 파일명 규칙은 매우 관대해서, 리눅스 파일시스템에서는 `NUL` 문자와 `/`(경로 구분자) 두 가지만 제외하면 사실상 모든 문자가 파일명이 될 수 있다.

명령은 기본적으로 공백으로 분리해서 작성한다(`command arg1 arg2 arg3`) — 맨 앞이 명령어, 이후가 공백으로 구분된 매개변수다. **`[` 는 특별한 명령이 아니라 `if` 문에서 조건 검사에 쓰이는 명령 그 자체**이며, 마지막 인자로 반드시 `]`를 필요로 한다. 이 때문에 인수 앞뒤 공백을 빠뜨리면 오류가 난다.

```bash
$ [10 -eq 10 ]; echo $?
[10: command not found      # '[10'이 통째로 하나의 명령 이름으로 해석됨

$ [ 10 -eq 10]; echo $?
bash: [: missing ']'         # '10]'이 하나의 인수로 붙어버림

$ [ a=b ]; echo $?
0                             # a=b라는 문자열 자체가 존재하므로 항상 참

$ [ a = b ]; echo $?
1                             # 공백을 모두 두어야 '=' 가 비교 연산자로 인식됨
```

반대로 **대입 연산은 공백을 절대 두면 안 된다.** 이는 Shell 문법에서 거의 유일하게 "공백을 넣으면 안 되는" 예외적인 규칙이다.

```bash
$ AA = 10
AA: command not found        # AA가 명령어로, =·10이 각각 인수로 해석됨

$ AA=10
$ echo $AA
10                            # 공백 없이 붙여 써야 정상적인 대입
```

**참/거짓 판단**은 오직 명령의 **종료 상태 값**(`$?`)으로만 이루어진다. 0이면 참이자 정상 종료, 0 이외의 값은 거짓이자 오류를 뜻한다. 함수에서 `return`으로 지정하는 값도 연산 결과가 아니라 바로 이 참/거짓 판정에 쓰이는 종료 상태 값이다.

```bash
$ func() { return $1; }                       # 전달받은 인수를 그대로 종료 상태로 반환
$ if func 0; then echo true; else echo false; fi
true                                            # 종료 상태 0 → 참

$ if func 1; then echo true; else echo false; fi
false                                           # 종료 상태 1 → 거짓
```

`return`은 값을 반환하는 일반 프로그래밍 언어의 `return`과 달리 종료 상태 값 전용이며, 연산 결과 자체는 표준 출력(stdout)으로 내보낸다.

```bash
$ func() { expr $1 + $2; return 5; }
$ func 1 2
3                    # stdout으로 출력된 연산 결과
$ echo $?
5                    # $?로 확인하는 것은 return이 지정한 종료 상태 값
```

**명령 종료 문자**로는 개행(newline)이 기본이며, 개행 없이 한 줄에 명령을 이어 쓸 때만 `;`가 필요하다. 특히 `{ ;}` 로 명령을 grouping할 때는 마지막 명령 뒤에도 `;`를 붙여야 인수(`}`)와 명확히 구분된다. 단, `()`로 만드는 subshell은 공백이나 `;`를 신경 쓰지 않아도 된다.

```bash
$ for i in {1..3} do echo $i done
> 오류                                     # 개행 없이 이어 썼는데 ; 를 안 붙임

$ for i in {1..3}; do echo $i; done       # 정상
1
2
3

$ { echo 1 }              # '}' 까지 하나의 문자열로 붙어버림
1 }
$ { echo 1 ;}              # 정상 : '}' 앞에 ; 로 인수 구분
1
```

**단어 분리(Word Splitting)와 Filename Expansion(Globbing)** 은 변수·명령 치환을 quote하지 않았을 때 발생한다. 변수 값을 공백으로 나눠 여러 인수로 쪼개는 것이 단어 분리이고, 값에 포함된 glob 문자(`*`, `?`, `[ ]`)가 실제 파일명과 매칭되어 뜻하지 않게 확장되는 것이 globbing이다.

```bash
$ AA="User-Agent: *"
$ echo "$AA"              # quote하면 globbing 없이 그대로 출력
User-Agent: *
$ echo $AA                 # quote하지 않으면 '*'가 현재 디렉터리 파일 목록으로 확장될 수 있음
```

**명령 옵션과 `--`**: 옵션은 보통 `-`나 `--`로 시작하는데, 검색하려는 문자열 자체가 `-`로 시작하면 옵션으로 오인될 수 있다. `--`를 쓰면 그 이후는 옵션이 아니라는 것을 명시할 수 있다.

```bash
$ grep -r '-n'          # '-n'이 검색어가 아니라 grep의 옵션으로 해석될 위험
$ grep -r -- '-n'       # -- 이후는 옵션이 아님을 명시, 안전
```

스크립트에서 명령 인수로 변수를 사용할 때는 이 `--` 관용구를 습관적으로 넣는 것이 오류를 줄이는 방법이다.

**`cd` 명령은 반드시 종료 상태를 확인해야 한다.** `cd`가 실패했는데 그 사실을 모른 채 다음 명령이 실행되면, 의도한 디렉터리가 아닌 곳에서 파괴적인 명령이 실행될 위험이 있다.

```bash
# 위험한 패턴 : cd 실패 시 현재 디렉터리에서 rm -rf * 가 실행됨
cd ~/tempdir
rm -rf *

# 안전한 패턴 : cd 성공 시에만 다음 명령 실행
cd ~/tempdir && rm -rf *
```

**주석**은 `#`로 표시하지만, 명령문 자체에 `#`가 포함될 수 있어(예: URL, 파일명) `#` 앞에 공백이 있어야만 주석으로 처리된다. 주석 처리를 막고 싶다면 escape하거나 quote한다.

**`read` 명령**은 파일 디스크립터에서 값을 읽어 변수에 저장하는 명령이다. 사용자로부터 키보드 입력을 받을 때도, 파일 내용을 한 줄씩 읽을 때도 동일하게 쓰인다.

```bash
#!/bin/bash
echo "name: "
read NAME
echo "Your name is $NAME"
```

파일을 읽을 때는 리다이렉션(`<`)으로 파일 디스크립터를 연결한다. 한 줄만 읽으려면 `read line < $FILE`, 파일 전체를 줄 단위로 읽으려면 `while read line; do ...; done < $FILE` 패턴을 쓴다.

```bash
FILE=user.sh
read line < "$FILE"
echo "$line"

FILE=user.sh
while read line
do
    echo "$line"
done < "$FILE"
```

`read`는 한 줄을 통째로 읽을 수도 있지만, **여러 변수를 나열하면 공백/탭 기준으로 컬럼 단위로 나눠 읽는다.** 파일이 `passwd 16`, `tistory 27`, `ubuntu 30`처럼 "이름 나이" 형태의 두 컬럼으로 구성되어 있다면, `read name age`로 각 컬럼을 서로 다른 변수에 바로 담을 수 있다.

```bash
# user.log
# passwd 16
# tistory 27
# ubuntu 30

FILE=user.log
while read name age
do
    echo "$name : $age"
done < "$FILE"
```

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** Bash 기반 Linux 셸 환경 (RHEL 계열 기본 `/bin/bash`).

### Step 1. `[` 명령 공백 규칙 정리

```bash
[ 10 -eq 10 ]        # 정상 : [, 인수들, ] 모두 공백으로 구분
[10 -eq 10 ]         # 오류 : '[10'이 명령 이름이 됨
[ 10 -eq 10]         # 오류 : '10]'이 하나의 인수가 됨
[ a = b ]            # 정상 : '=' 양쪽 공백 필수
```

### Step 2. 대입 연산 vs 일반 명령 공백 규칙 비교

```bash
AA=10          # 대입 연산: 공백 금지 (표준)
AA = 10        # 오류: AA가 명령어로 해석됨

echo $AA       # 일반 명령: 공백으로 인수 구분 (표준)
```

### Step 3. 명령 종료 문자와 grouping

```bash
for i in {1..3}; do echo "$i"; done     # 개행 없이 한 줄이면 ; 필수

{ echo 1; echo 2; }                     # 명령 grouping: 마지막 명령 뒤에도 ; 필수
(echo 1; echo 2)                        # subshell: ; 규칙에 덜 민감
```

### Step 4. Word Splitting / Globbing 방지

```bash
AA="User-Agent: *"
echo "$AA"       # quote: 단어 분리·globbing 방지 (안전)
echo $AA          # no quote: 단어 분리·globbing 위험
```

### Step 5. `--`로 옵션·인수 경계 명시

```bash
grep -r -- "$pattern" .
rm -- "$filename"
```

### Step 6. `cd` 성공 확인 표준 패턴

```bash
cd "$target_dir" && rm -rf ./old_cache/*
# 또는
cd "$target_dir" || { echo "cd 실패: $target_dir"; exit 1; }
rm -rf ./old_cache/*
```

### Step 7. read 표준 패턴 모음

```bash
# 사용자 입력
read -p "이름을 입력하세요: " name
echo "안녕하세요, $name"

# 한 줄만 파일에서 읽기
read line < config.txt

# 파일 전체를 줄 단위로 순회
while read -r line
do
    echo "$line"
done < config.txt

# 컬럼(다중 변수) 단위로 읽기
while read -r name age
do
    echo "$name 은(는) $age 살"
done < user.log
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
type [                  # [ 가 명령(builtin)임을 확인
echo $?                 # 직전 명령의 참/거짓(종료 상태) 확인
printf '[%s]\n' $AA      # quote 없이 참조했을 때 실제 분리 결과 확인
printf '[%s]\n' "$AA"    # quote 했을 때와 비교
```

### 3-2. 트러블슈팅 시나리오

#### 시나리오 1. `[10 -eq 10 ]` 실행 시 `command not found`

- **원인:** `[`와 첫 번째 인수 사이에 공백이 없어 `[10` 전체가 하나의 명령 이름으로 해석된다.
- **해결:** `[` 뒤와 `]` 앞 모두 공백을 반드시 넣는다(`[ 10 -eq 10 ]`).

#### 시나리오 2. `AA = 10` 실행 시 `AA: command not found`

- **원인:** 대입 연산에 공백을 넣어 `AA`가 명령어로, `=`와 `10`이 각각 인수로 해석됐다.
- **해결:** 대입 연산은 공백 없이 `AA=10` 형태로만 작성한다.

#### 시나리오 3. 파일 목록에 없는 이상한 값이 출력됨

- **원인:** 변수 값에 `*` 같은 glob 문자가 있는데 quote 없이 참조해 파일명 확장(globbing)이 발생했다.
- **해결:** `echo "$변수"` 처럼 항상 quote해서 참조한다.

#### 시나리오 4. `grep -r '-n' .` 실행 시 검색어가 옵션으로 오인됨

- **원인:** `-`로 시작하는 검색 문자열이 grep의 옵션으로 해석됐다.
- **해결:** `grep -r -- '-n' .` 처럼 `--`로 옵션 구간의 끝을 명시한다.

#### 시나리오 5. `cd` 실패 후 의도치 않은 위치에서 `rm -rf *`가 실행됨

- **원인:** `cd`의 성공 여부를 확인하지 않고 다음 명령을 그대로 실행했다.
- **해결:** `cd "$dir" && rm -rf *` 또는 `cd "$dir" || exit 1` 패턴으로 반드시 성공을 확인한 뒤 진행한다.

#### 시나리오 6. `read name age`로 읽었는데 컬럼이 예상과 다르게 나뉨

- **원인:** 파일의 구분자가 공백/탭(IFS 기본값)이 아니라 콤마 등 다른 문자다.
- **해결:** `IFS=',' read -r name age` 처럼 `IFS`를 해당 구분자로 임시 변경한 뒤 읽는다.

---

>  **핵심 요약**
> - `[`는 `]`를 인수로 요구하는 명령이므로 앞뒤 공백 필수, 반대로 대입 연산(`AA=10`)은 공백 금지
> - 참/거짓 판단은 오직 종료 상태 값(`$?`, 0=참)으로만 이루어짐 — `return`도 값이 아닌 종료 상태 지정용
> - quote 없이 변수·명령 치환을 참조하면 단어 분리·globbing 위험, `--`로 옵션/인수 경계 명시, `cd`는 항상 성공 여부 확인
> - `read`는 사용자 입력과 파일 입력에 동일하게 쓰이며, 변수를 여러 개 나열하면 컬럼 단위로 분할해 읽음
> - 관련: Shell Script - 변수와 환경변수 (커널·쉘 개념 포함) · Shell Script - Metacharacters (메타문자) · Shell Script - exit 상태와 test 명령 · Shell Script - Quotes와 Escape Sequences 심화
