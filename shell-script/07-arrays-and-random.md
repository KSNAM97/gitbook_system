# 📦 Shell Script - 배열(Array)과 RANDOM

> **Tag:** #Linux #ShellScript #Bash #Array #배열 #RANDOM #declare #unset
> **핵심 요약:** 배열은 여러 개의 값을 하나의 변수에 인덱스 순서대로 저장하는 자료구조이며, Bash의 인덱스는 **0번부터** 시작한다. 전체 요소는 `${arr[@]}`, 인덱스 목록은 `${!arr[@]}`, 요소 개수는 `${#arr[@]}` 로 조회한다. `unset` 으로 요소를 삭제하면 개수만 줄고 인덱스는 재정렬되지 않아 중간에 구멍(sparse array)이 생긴다. `RANDOM` 은 호출할 때마다 0~32767 범위의 정수를 반환하는 Bash 특수 변수로, 나머지 연산과 오프셋을 조합해 원하는 범위를 만든다.

---

## 1. 📖 개요 (Overview)

일반 변수는 값 하나만 저장하므로 `fruit1`, `fruit2`, `fruit3` 처럼 변수 개수를 늘려야 하고, 값이 늘어날 때마다 코드를 수정해야 한다. 배열은 하나의 변수에 여러 값을 순서대로 담고 **인덱스와 반복문으로 일괄 처리**할 수 있어 값의 개수가 변해도 로직을 고치지 않는다는 장점이 있다. 선언 형식은 `배열명=("값1" "값2" "값3")` 이며 요소는 **공백으로 구분**하고, 값에 공백이 포함되면 반드시 겹따옴표로 감싼다. 인덱스를 지정하지 않고 `arr="apple"` 로 대입하면 `arr[0]` 에 저장되므로, 즉 Bash에서 일반 변수는 사실상 요소가 하나인 배열처럼 동작한다. 명시적 선언은 `declare -a arr`(인덱스 배열), `declare -A map`(연관 배열, Bash 4.0 이상)이며, 연관 배열은 **반드시** `declare -A` 로 먼저 선언해야 한다.

`"${arr[@]}"` 는 각 요소를 **독립된 값**으로 전개하고, `"${arr[*]}"` 는 모든 요소를 `IFS` 의 첫 문자(기본 공백)로 이어붙인 **하나의 문자열**로 전개한다. `${#arr[@]}` 는 **요소 개수**, `${#arr[0]}` 는 **0번 요소 문자열의 길이**로 의미가 완전히 다르다. 따옴표 없이 `echo ${arr[@]}` 와 `echo ${arr[*]}` 를 실행하면 화면 출력이 같아 보여 차이를 놓치기 쉬운데, 차이는 **따옴표로 감쌌을 때** 드러난다. 반복문 순회는 예외 없이 `"${arr[@]}"` 를 사용해야 하며, `"${arr[*]}"` 로 순회하면 반복이 1회만 실행된다.

```bash
arr=("a b" "c")
printf '[%s]\n' "${arr[@]}"      # [a b] / [c]   → 요소 2개로 전개
printf '[%s]\n' "${arr[*]}"      # [a b c]       → 문자열 1개로 결합
echo ${#arr[@]}                   # 2  (요소 개수)
echo ${#arr[0]}                   # 3  ("a b" 의 문자 길이)
```

배열 요소의 추가는 `arr[인덱스]="값"` 또는 `arr+=("값")`, 수정은 기존 인덱스에 재대입, 삭제는 `unset '배열명[인덱스]'` 로 처리한다. 다만 `unset` 은 해당 인덱스만 제거할 뿐 뒤 요소를 앞으로 끌어당기지 않기 때문에 인덱스가 `0 1 3 4` 처럼 끊긴 **희소 배열(sparse array)** 이 되는 함정이 있다. `arr+=("값")` 는 항상 **현재 최대 인덱스 + 1** 위치에 추가되며, 중간을 `unset` 한 상태에서도 빈 인덱스를 메우지 않는다. 삭제 시 대괄호가 글롭 메타문자로 해석되는 것을 막기 위해 `unset 'name[2]'` 처럼 **작은따옴표**로 감싸는 것이 안전하다. 인덱스를 다시 촘촘하게 정리하려면 배열을 재생성하면 되고(`arr=("${arr[@]}")`), 배열 전체 삭제는 `unset arr` 로 한다.

배열을 반복문으로 순회할 때는 값만 필요하면 `for v in "${arr[@]}"`, 인덱스가 함께 필요하면 `for i in "${!arr[@]}"` 를 사용하는 것이 표준 패턴이다. `for ((i=0; i<${#arr[@]}; i++))` 방식은 **인덱스가 연속이라는 전제**가 깨지는 순간(즉 `unset` 이후) 빈 값을 읽거나 마지막 요소를 건너뛰므로 위험하다.

```bash
name=("kim" "lee" "park" "ryu")
unset 'name[2]'

echo ${#name[@]}                 # 3
echo ${!name[@]}                 # 0 1 3   ← 인덱스 2가 비어 있음

for ((i=0; i<${#name[@]}; i++)); do echo "$i : ${name[i]}"; done
# 0 : kim / 1 : lee / 2 :        ← 빈 값, ryu 누락

for i in "${!name[@]}"; do echo "$i : ${name[i]}"; done
# 0 : kim / 1 : lee / 3 : ryu    ← 안전
```

`RANDOM` 은 참조할 때마다 **0~32767** 사이의 정수를 반환한다. `min ~ max` 범위는 `$(( RANDOM % (max - min + 1) + min ))` 공식으로 만든다. 단 나머지 연산 방식은 32768이 나눗수의 배수가 아닐 때 앞쪽 값이 미세하게 더 자주 나오는 **모듈로 편향(modulo bias)** 이 있으므로, 통계적 균일성이 중요한 용도에는 `shuf` 를 사용한다. 대표적인 범위 공식으로 `0~100` 은 `$(( RANDOM % 101 ))`, `30~100` 은 `$(( (RANDOM % 71) + 30 ))`, `1~6` 은 `$(( RANDOM % 6 + 1 ))` 이 있다. `RANDOM=42` 처럼 값을 대입하면 **시드가 고정**되어 같은 난수 순서가 재현되므로 테스트 스크립트 디버깅에 유용하다. 균일 난수·중복 없는 추출이 필요하면 `shuf -i 1-100 -n 1`, 암호학적 난수가 필요하면 `/dev/urandom`(예: `od -An -N2 -tu2 /dev/urandom`)을 사용하며, `RANDOM` 은 보안 용도로 쓰지 않는다. 실습용 점수·테스트 데이터 생성에는 `RANDOM` 이 충분하지만, 임시 비밀번호나 토큰 생성에 `RANDOM` 을 쓰는 것은 예측 가능성 때문에 **금지 사항**으로 취급한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** Bash 기반 Linux 셸 환경 (RHEL 계열 기본 `/bin/bash`).

### Step 1. 배열 선언 및 조회 문법

```bash
arr=("값1" "값2" "값3")      # 배열 선언 (요소는 공백 구분)
declare -a arr                # 인덱스 배열 명시적 선언
declare -A map                # 연관 배열 선언 (Bash 4.0+, 필수)

echo "${arr[0]}"              # 0번 요소 (인덱스는 0부터)
echo "${arr[@]}"              # 모든 요소 (개별 값으로 전개) ← 반복문 표준
echo "${arr[*]}"              # 모든 요소 (하나의 문자열로 결합)
echo "${!arr[@]}"             # 인덱스 목록
echo "${#arr[@]}"             # 요소 개수
echo "${#arr[0]}"             # 0번 요소의 문자열 길이 (개수 아님)
echo "${arr[@]:1:2}"          # 1번부터 2개 슬라이싱
```

**조회 문법 요약표**

| 문법 | 의미 | 예시 결과 (`name=(kim lee park ryu)`) |
| --- | --- | --- |
| `${name[@]}` | 모든 요소(개별 전개) | `kim lee park ryu` |
| `${name[*]}` | 모든 요소(문자열 결합) | `kim lee park ryu` |
| `${!name[@]}` | 인덱스 목록 | `0 1 2 3` |
| `${#name[@]}` | 요소 개수 | `4` |
| `${#name[1]}` | 1번 요소 길이 | `3` |
| `${name[-1]}` | 마지막 요소(Bash 4.3+) | `ryu` |

### Step 2. 요소 추가 · 수정 · 삭제

```bash
name=("kim" "lee" "park" "ryu")

name[4]="hong"                # 방법 1) 인덱스 직접 지정하여 추가
name+=("choi")                # 방법 2) 최대 인덱스 다음 위치에 추가 (권장)

name[3]="jung"                # 수정 : 개수와 다른 인덱스는 변하지 않음

unset 'name[2]'               # 특정 요소 삭제 (인덱스는 재정렬되지 않음)
name=("${name[@]}")           # 인덱스 재정렬 (구멍 제거)
unset name                    # 배열 전체 삭제
```

### Step 3. 배열 순회 표준 패턴

```bash
# 값만 필요할 때 (공백 포함 값도 안전)
for item in "${arr[@]}"
do
    echo "$item"
done

# 인덱스가 함께 필요할 때 (희소 배열에도 안전)
for i in "${!arr[@]}"
do
    echo "$i : ${arr[i]}"
done
```

### Step 4. RANDOM 범위 지정 템플릿

```bash
echo $(( RANDOM ))                        # 0 ~ 32767
echo $(( RANDOM % 101 ))                  # 0 ~ 100
echo $(( (RANDOM % 71) + 30 ))            # 30 ~ 100
echo $(( RANDOM % (max - min + 1) + min ))  # min ~ max 일반식

RANDOM=42                                 # 시드 고정 (동일 순서 재현)
shuf -i 1-100 -n 1                        # 편향 없는 균일 난수
shuf -i 1-45 -n 6                         # 중복 없이 6개 추출
```

### Step 5. 배열 + RANDOM 조합 표준 골격

```bash
#!/bin/bash

num=()                                    # 빈 배열 선언

for (( i=0; i<10; i++ ))                  # 요소 10개 생성
do
    num[i]=$(( RANDOM % 101 ))            # (( )) 안에서는 i 앞에 $ 불필요
done

echo "생성된 값 : ${num[@]}"
echo "요소 개수 : ${#num[@]}"

for value in "${num[@]}"                  # 값 순회
do
    (( value % 2 == 0 )) && echo "$value : 짝수"
done
```

---

## 3. 📝 실습 예제 모음 (EX)

### 3-1. 배열 기본 조작 실습

```bash
# EX1) 배열 선언 후 요소·인덱스·개수를 각각 확인
name=("kim" "lee" "park" "ryu")
echo "${name[2]}"        # park — 인덱스 2번 요소 (0부터 시작)
echo "${name[@]}"        # kim lee park ryu — 전체 요소 개별 전개
echo "${!name[@]}"       # 0 1 2 3 — 인덱스 목록
echo "${#name[@]}"       # 4 — 요소 개수 (${#name}은 0번 요소 문자 길이이므로 혼동 주의)

# EX2) 요소 추가 → 수정 → 삭제 → 재정렬 흐름 확인
name+=("hong")           # kim lee park ryu hong — 최대 인덱스 다음에 추가
name[3]="jung"           # kim lee park jung hong — 인덱스 3 수정
unset 'name[2]'          # kim lee jung hong  (인덱스 : 0 1 3 4) — 삭제 후 구멍 발생
echo "${!name[@]}"       # 0 1 3 4 — 인덱스에 구멍이 생긴 것 확인
name=("${name[@]}")      # 재정렬 — 구멍 없애기
echo "${!name[@]}"       # 0 1 2 3
```

### 3-2. 인덱스별 요소 접근 + 병렬 배열 인덱스 패턴

```bash
# 인덱스별 직접 접근 (0번, 1번)
name=("kim" "lee" "park")
echo "${name[0]}"        # kim (첫 번째 요소)
echo "${name[1]}"        # lee (두 번째 요소)

# 학생 점수 배열 — 인덱스 i로 접근하는 패턴
scores=(85 92 70 45 61)

for i in "${!scores[@]}"
do
    echo "$((i+1))번 학생 : ${scores[i]}점"
done
```

### 3-3. 배열에서 짝수만 출력 (배열 + RANDOM + 조건문)

```bash
#!/bin/bash
# 랜덤 정수 10개를 배열에 저장한 뒤 짝수만 출력

num=()                          # 빈 배열 초기화

for (( i=0; i<10; i++ ))
do
    num[i]=$(( RANDOM ))        # RANDOM: 0~32767 난수; (( )) 안에서 $ 없이 RANDOM 사용 가능
done

echo "생성된 값 : ${num[@]}"   # 전체 요소 출력
echo

echo "===== 짝수 값 ====="
for number in "${num[@]}"
do
    if (( number % 2 == 0 ))   # 나머지가 0이면 짝수
    then
        echo "$number"
    fi
done
```

```bash
chmod +x ./script/arr_example01.sh
./script/arr_example01.sh
```

### 3-4. 짝수 · 홀수 개수 집계

```bash
#!/bin/bash
# 랜덤 정수 100개를 저장한 뒤 짝수/홀수 개수를 집계

num=()
even=0
odd=0

for (( i=0; i<100; i++ ))
do
    num[i]=$RANDOM              # $RANDOM: (( )) 밖에서는 $ 필요
done

for number in "${num[@]}"
do
    if (( number % 2 == 0 ))
    then
        (( even++ ))            # (( )): 변수 앞 $ 없이 증감 가능
    else
        (( odd++ ))
    fi
done

echo "배열 요소 개수 : ${#num[@]}개"   # ${#배열[@]}: 요소 개수
echo "짝수 개수 : ${even}개"
echo "홀수 개수 : ${odd}개"
```

```bash
배열 요소 개수 : 100개
짝수 개수 : 45개
홀수 개수 : 55개
```

### 3-5. 랜덤 점수의 합격 · 불합격 판정

```bash
#!/bin/bash
# 0~100 랜덤 점수 10개를 저장한 뒤 60점 기준으로 합격 여부 출력

scores=()

for (( i=0; i<10; i++ ))
do
    scores[i]=$(( RANDOM % 101 ))          # 0 ~ 100
done

for score in "${scores[@]}"
do
    if (( score >= 60 ))
    then
        echo "점수 ${score}점 : 합격"
    else
        echo "점수 ${score}점 : 불합격"
    fi
done
```

### 3-6. 학생별 3과목 평균 · 과락 판정 (병렬 배열)

```bash
#!/bin/bash
# 국어/영어/수학을 각각 배열로 관리하고 같은 인덱스를 같은 학생으로 처리
# 평균 60점 이상 + 모든 과목 40점 이상일 때만 합격 (한 과목이라도 40점 미만이면 과락)

kor=(); eng=(); math=()
pass=0
fail=0

for (( i=0; i<10; i++ ))
do
    kor[i]=$(( (RANDOM % 71) + 30 ))       # 30 ~ 100
    eng[i]=$(( (RANDOM % 71) + 30 ))
    math[i]=$(( (RANDOM % 71) + 30 ))
done

echo "-----------------------------"
for i in "${!kor[@]}"                       # 인덱스 기준 순회 (병렬 배열 표준)
do
    total=$(( kor[i] + eng[i] + math[i] ))
    average=$(( total / 3 ))                # 정수 나눗셈 (소수점 절삭)

    echo "$(( i + 1 ))번 학생"
    echo "국어 점수 : ${kor[i]}점"
    echo "영어 점수 : ${eng[i]}점"
    echo "수학 점수 : ${math[i]}점"
    echo "총점 : ${total}점"
    echo "평균 : ${average}점"

    if (( average >= 60 && (kor[i] < 40 || eng[i] < 40 || math[i] < 40) ))
    then
        echo "결과 : 과락 불합격"
        (( fail++ ))
    elif (( average >= 60 ))
    then
        echo "결과 : 합격"
        (( pass++ ))
    else
        echo "결과 : 불합격"
        (( fail++ ))
    fi
    echo "-----------------------------"
done

echo "합격 : ${pass}명"
echo "불합격 : ${fail}명"
```

> **참고:** 평균은 `$(( total / 3 ))` 로 계산되어 소수점이 버려진다. 소수점 평균이 필요하면 `echo "scale=2; $total/3" | bc` 를 사용한다.

---

## 4. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 4-1. 필수 검증 명령어

```bash
bash -n <스크립트>                 # 문법 사전 검사
declare -p <배열명>                 # 배열 전체 구조(인덱스+값)를 한 번에 확인
echo "${!arr[@]}"                  # 인덱스 연속성 확인 (희소 배열 진단)
echo "${#arr[@]}"                  # 요소 개수 확인
printf '[%s]\n' "${arr[@]}"        # 요소 경계 확인 (공백 포함 값 진단)
echo $BASH_VERSION                 # 연관 배열(4.0+) 지원 여부 확인
```

### 4-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. `unset` 으로 요소를 삭제했는데 인덱스가 어긋나 반복이 깨짐

- **증상**: `unset 'name[2]'` 이후 `for ((i=0;i<${#name[@]};i++))` 로 순회하면 빈 값이 출력되고 마지막 요소가 누락됨.
- **원인**: `unset` 은 개수만 줄이고 뒤 요소를 앞으로 이동시키지 않아 인덱스에 구멍이 생긴다.
- **해결**:

```bash
for i in "${!name[@]}"; do echo "${name[i]}"; done   # 인덱스 목록 기준 순회
name=("${name[@]}")                                   # 또는 인덱스 재정렬
```

#### 🚨 시나리오 2. `for item in "${arr[*]}"` 실행 시 반복이 1회만 동작

- **원인**: `"${arr[*]}"` 는 배열 전체를 하나의 문자열로 결합한다.
- **해결**: 요소별 순회는 `"${arr[@]}"` 를 사용한다.

#### 🚨 시나리오 3. `${#arr[@]}` 대신 `${#arr}` 를 써서 개수가 이상하게 나옴

- **증상**: 요소가 4개인데 `echo ${#arr}` 결과가 `3` 으로 출력됨.
- **원인**: `${#arr}` 는 `${#arr[0]}` 과 같아 0번 요소의 문자열 길이를 반환한다.
- **해결**: 요소 개수는 반드시 `${#arr[@]}` 로 조회한다.

#### 🚨 시나리오 4. 공백이 포함된 값이 여러 요소로 쪼개짐

- **증상**: `arr=(hello world "kim lee")` 를 따옴표 없이 전개했더니 요소가 4개로 계산됨.
- **원인**: 선언 시 따옴표 누락, 또는 참조 시 `${arr[@]}` 를 따옴표 없이 사용해 단어 분리(word splitting) 발생.
- **해결**: 선언과 참조 양쪽 모두 겹따옴표를 사용한다.

```bash
arr=("hello world" "kim lee")
printf '[%s]\n' "${arr[@]}"
```

#### 🚨 시나리오 5. `declare -A` 없이 연관 배열을 사용해 값이 덮어써짐

- **증상**: `map[apple]=1; map[banana]=2` 후 `echo ${map[apple]}` 이 `2` 를 출력.
- **원인**: `declare -A` 선언이 없으면 인덱스 배열로 취급되어 문자열 첨자가 산술 평가(대부분 0)로 처리된다.
- **해결**:

```bash
declare -A map
map[apple]=1; map[banana]=2
for key in "${!map[@]}"; do echo "$key : ${map[$key]}"; done
```

#### 🚨 시나리오 6. `RANDOM % 101` 결과 분포가 균일하지 않다는 지적

- **원인**: `RANDOM` 의 값 범위(32768개)가 101의 배수가 아니라 앞쪽 값이 약간 더 자주 선택되는 모듈로 편향이 존재한다.
- **해결**: 실습·테스트 데이터에는 그대로 사용해도 무방하지만, 균일성이 필요하면 `shuf -i 0-100 -n 1` 을 사용한다. 보안 목적에는 `RANDOM` 을 사용하지 않는다.

---


> 📌 **핵심 요약**
> - 인덱스는 **0부터**, 전체 요소는 `"${arr[@]}"`, 인덱스는 `${!arr[@]}`, 개수는 `${#arr[@]}`
> - `${#arr[0]}`(문자 길이)과 `${#arr[@]}`(요소 개수)를 혼동하지 말 것
> - `unset` 은 인덱스를 재정렬하지 않는다 → `for i in "${!arr[@]}"` 또는 `arr=("${arr[@]}")`
> - 순회는 항상 `"${arr[@]}"`, `"${arr[*]}"` 는 문자열 1개로 결합됨
> - RANDOM 범위식 = `$(( RANDOM % (max-min+1) + min ))`, **보안 용도 사용 금지**
> - 관련: 🔁 Shell Script - 반복문 (for · while · until) · 🎯 Shell Script - 위치 매개변수 (Positional Parameters) · 🔢 Shell Script - expr · let (산술 연산) · 🧩 Shell Script - 통합 정리
