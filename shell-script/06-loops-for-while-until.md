# 🔁 Shell Script - 반복문 (for · while · until)

> **Tag:** #Linux #ShellScript #Bash #반복문 #for #while #until #break #continue
> **핵심 요약:** Bash 반복문은 **for(정해진 목록을 순회)**, **while(조건이 참인 동안 반복)**, **until(조건이 참이 될 때까지 반복, while과 조건이 반대)** 3가지로 구성된다. `break` 는 반복문 자체를 종료하고, `continue` 는 현재 회차만 건너뛰고 다음 회차로 진행한다.

---

## 1. 📖 개요 (Overview)

`for` 는 **반복 대상 목록이 정해져 있을 때**(숫자 범위, 파일 목록, 배열 등) 사용한다. `while` 은 **조건이 참인 동안** 반복하며 반복 횟수가 미리 정해지지 않은 상황(로그인 재시도, 파일 대기 등)에 적합하다. `until` 은 `while` 과 정반대로 **조건이 거짓인 동안** 반복하다가 조건이 참이 되는 순간 종료한다. 반복문이 없으면 `echo 1; echo 2; echo 3 ...` 처럼 매번 명령을 나열해야 하지만, 반복문을 쓰면 코드가 짧아지고 범위 확장(예: 10 → 100)도 한 곳만 수정하면 된다. 계정 생성, 파일 순회, 로그 분석, 백업 등 반복 작업은 거의 모두 `for` 문으로 구현 가능하다.

`for` 문은 목록의 값을 하나씩 꺼내 변수에 대입하며 반복하는데, 목록은 ① 직접 나열, ② 범위 확장(`{1..10}`), ③ 간격 지정(`{1..10..2}`), ④ C언어 스타일(`((i=1;i<=5;i++))`), ⑤ 배열(`"${arr[@]}"`) 방식으로 지정할 수 있다. 기본 형식은 `for 변수 in 값1 값2 값3; do 명령; done` 이다. 범위 확장(`{1..10}`)은 **변수를 사용할 수 없다**(고정 리터럴만 가능). 변수 범위가 필요하면 C스타일 `for ((i=num1; i<=num2; i++))` 또는 `seq` 명령(`for i in $(seq "$num1" "$num2")`)을 사용해야 한다. 배열 반복 시 `"${arr[@]}"`(각 요소를 개별 인자로 전달, 공백 포함 값에 안전)와 `"${arr[*]}"`(전체를 하나의 문자열로 결합)의 동작 차이를 구분해야 한다.

`break` 와 `continue` 는 반복문의 흐름을 다르게 바꾼다. `break` 는 반복문을 만나는 즉시 **반복문 자체를 완전히 종료**한다(스크립트 전체 종료가 아님). `continue` 는 그 아래 남은 명령을 건너뛰고 **다음 반복 회차의 시작점**으로 돌아간다. `break` 는 원하는 값을 찾은 뒤 불필요한 반복을 막을 때(예: 파일 검색 성공 시) 사용하고, `continue` 는 특정 조건(예: 3의 배수)만 건너뛰고 나머지 반복은 계속 진행할 때 사용한다. 중첩 반복문에서 `break`/`continue` 는 기본적으로 **가장 안쪽 반복문**에만 영향을 준다.

`while` 문으로 로그인 재시도 로직을 만들 때는 시도 횟수를 세는 카운터 변수를 두고, `while [ "$count" -le 최대횟수 ]` 조건으로 반복하며, 성공 시 `break`(또는 `exit 0`)로 즉시 종료하고 실패가 누적되면 `exit 1` 로 스크립트를 완전히 끝내는 것이 핵심 설계 포인트다. 카운터 증가는 `((count++))` 또는 `count=$((count+1))` 로 처리한다. 로그인 성공 시 `break` 로 while 루프만 빠져나오는 경우와, `exit 0` 로 스크립트 자체를 끝내는 경우는 이후 코드 실행 여부가 다르므로 설계 의도에 맞게 선택해야 한다.

`until` 은 **특정 조건이 충족될 때까지 주기적으로 확인**하는 폴링(polling) 패턴에 자주 사용된다. 예를 들어 특정 플래그 파일이 생성될 때까지 일정 간격으로 대기하는 스크립트가 대표적이다. 기본 형식은 `until [ 조건식 ]; do 명령; done` 이며 조건이 거짓인 동안 반복하다 참이 되는 순간 종료한다. `sleep N` 과 결합해 과도한 CPU 점유 없이 일정 간격으로 상태를 재확인하도록 설계하는 것이 표준이다.

`for` 문으로 여러 계정의 홈 디렉터리 소유권/권한을 일괄 점검할 때는 ① 계정 존재 여부 확인(`id "$user"`) → ② 없으면 생성(`useradd`) → ③ 홈 디렉터리 존재 확인 → ④ 없으면 생성(`mkdir -p`) → ⑤ 소유권 확인(`stat -c "%U"`) 후 불일치 시 `chown` → ⑥ 권한 확인(`stat -c "%a"`) 후 불일치 시 `chmod` 순서로 로직을 구성하며, **각 단계마다 성공 여부(`$?`)를 확인하고 실패 시 `continue` 로 다음 계정으로 넘어가는 방어적 설계**가 표준이다. `stat -c "%U" <path>` 는 소유자 이름, `%G` 는 소유 그룹, `%a` 는 숫자 권한(예: 700), `%n` 은 경로명을 의미한다. 이 패턴은 대량의 사용자 계정을 대상으로 하는 일괄 점검·교정 자동화 스크립트의 표준 골격이다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** Bash 기반 Linux 셸 환경 (RHEL 계열 기본 `/bin/bash`).

### Step 1. for 문 5가지 목록 지정 방식 (동작 구조 주석 포함)

```bash
# (1) 기본 목록 반복
for num in 1 2 3 4 5   # 목록(1 2 3 4 5)의 값을 하나씩 순서대로 num에 대입
do                      # 반복 실행할 명령의 시작
    echo "출력 : $num"  # 매 회차마다 현재 num 값 사용
done                    # 목록의 마지막 값까지 처리하면 반복 종료

# (2) 범위(range) 표현
for num in {1..10}      # 중괄호 확장으로 1~10 목록을 쉘이 미리 생성 (고정 리터럴만 가능)
do
    echo "출력 : $num"
done

# (3) 간격(step) 반복
for num in {1..10..2}   # 1부터 10까지 2씩 건너뛰며 생성 (1,3,5,7,9)
do
    echo "출력 : $num"
done

# (4) C언어 스타일 (변수 범위 가능 - 실무에서 자주 사용)
for (( i=1; i<=5; i++ ))   # i=1(초기식) ; i<=5(조건식, 참인 동안 반복) ; i++(매 회차 후 증감식)
do
    echo "출력 : $i"
done

# (5) 배열 반복
arr=("apple" "banana" "cherry")   # 배열 선언 (인덱스 0,1,2)
for item in "${arr[@]}"           # 배열 각 요소를 개별 값으로 순회 (공백 포함 값도 안전)
do
    echo "$item"
done
```

### Step 2. while / until 기본 형식 (동작 구조 주석 포함)

```bash
# while : 조건이 참인 동안 반복
while [ 조건식 ]     # 매 회차 시작 시 조건을 재평가 -> 참(0)이면 계속, 거짓이면 종료
do
    반복할 명령       # 조건이 참인 동안 실행되는 본문 (조건을 변화시키는 코드가 반드시 포함되어야 함)
done                  # while 블록 종료 지점, done에서 다시 조건 평가로 되돌아감

# until : 조건이 거짓인 동안 반복 (참이 되면 종료)
until [ 조건식 ]      # while과 정반대 : 조건이 거짓(0 아님)인 동안 계속 반복
do
    반복할 명령       # 조건이 참이 되는 순간 반복 종료
done
```

### Step 3. break / continue 표준 사용법 (동작 구조 주석 포함)

```bash
for num in {1..10}          # 1~10을 순회
do
    if [ "$num" -eq 5 ]      # num이 5가 되는 순간 조건 참
    then
        echo "숫자 5를 만나 반복문을 종료합니다."
        break                # 반복문(for) 자체를 즉시 완전히 빠져나감 -> 6~10은 실행되지 않음
    fi
    echo "$num"              # break 되기 전까지는 정상 출력
done

for num in {1..15}
do
    if [ $(( num % 3 )) -eq 0 ]   # 3의 배수인지 산술 평가로 판단
    then
        continue                  # 아래 echo를 건너뛰고 바로 다음 회차(num 증가)로 이동
    fi
    echo "$num"               # 3의 배수가 아닐 때만 이 줄이 실행됨
done
```

### Step 4. 로그인 재시도 표준 템플릿 (while + 카운터)

```bash
#!/bin/bash

id="user1"
pw="admin1234"
count=1

while [ "$count" -le 3 ]        # count가 3 이하인 동안(즉 최대 3회 시도) 반복
do
    read -p "아이디 : " input_id
    read -s -p "비밀번호 : " input_pw
    echo

    if [ "$input_id" = "$id" ] && [ "$input_pw" = "$pw" ]
    then
        echo "로그인 성공"
        exit 0                  # 성공 시 스크립트 자체를 즉시 종료 (while 루프 밖으로도 나감)
    else
        echo "로그인 실패 ($count/3)"
    fi

    count=$((count + 1))        # 카운터 증가 (누락 시 무한 루프 위험)
done                             # count가 4가 되는 순간 조건이 거짓 -> 반복 종료

echo "3회 실패하여 종료합니다."
exit 1                           # 3회 모두 실패 시 exit code 1로 종료
```

### Step 5. until 폴링(대기) 표준 템플릿

```bash
#!/bin/bash

file="/temp/ready.flag"

echo "$file 파일이 생성될 때까지 감지합니다."

until [ -f "$file" ]          # 파일이 존재하지 않는(거짓인) 동안 계속 반복
do
    echo "$file 파일이 아직 없습니다. 대기중..."
    sleep 3                    # 3초 간격으로 재확인 (CPU 과점유 방지 필수)
done                           # 파일이 생성되어 조건이 참이 되는 순간 반복 종료

echo "${file} 파일이 생성되었습니다."
```

### Step 6. 대량 계정 점검/교정 표준 템플릿 (for + stat)

```bash
#!/bin/bash

for user in user{1..10}
do
    if ! id "$user" > /dev/null 2>&1
    then
        useradd "$user"
        if [ $? -ne 0 ]; then
            echo "$user 계정 생성 실패"
            continue
        fi
    fi

    home="/home/$user"

    if [ ! -d "$home" ]; then
        mkdir -p "$home"
    fi

    owner=$(stat -c "%U" "$home")
    if [ "$owner" != "$user" ]; then
        chown "$user:$user" "$home"
    fi

    permission=$(stat -c "%a" "$home")
    if [ "$permission" -ne 700 ]; then
        chmod 700 "$home"
    fi
done
```

### Step 7. 파일 크기/용량 점검 표준 템플릿 (for + continue)

```bash
#!/bin/bash

for file in /var/log/messages /var/log/secure /var/log/cron
do
    if [ ! -f "$file" ]
    then
        echo "$file 파일이 없습니다."
        continue
    fi

    size=$(du -k "$file" | awk '{print $1}')

    if [ "$size" -ge 10240 ]
    then
        echo "$file : 용량 초과 (${size}KB)"
    else
        echo "$file : 용량 정상 (${size}KB)"
    fi
done
```

---

## 3. 📝 실습 예제 모음 (EX)

### 3-1. for 문 기본 실습

```bash
# EX1) /etc/ 디렉터리에서 'a'로 시작하는 파일/디렉터리를 삭제하며 삭제된 이름을 출력
for filerm in /etc/a*           # 글롭 /etc/a*: a로 시작하는 모든 항목을 목록으로 확장
do
     echo "${filerm} 삭제"
     rm -rf "$filerm"           # -rf: 재귀 강제 삭제 (디렉터리 포함)
done

# EX2) for문을 사용하여 1부터 10까지의 합을 구하는 스크립트 (방법 1: 범위 확장)
#!/bin/bash
sum=0
for num in {1..10}              # {1..10}: 쉘이 파싱 시점에 1 2 3 ... 10 목록으로 확장
do
  echo "$sum + $num = $(( sum + num ))"
  sum=$(( sum + num ))          # $(( )): 산술 확장으로 합산 결과 반환
done
echo "1부터 10까지의 총 합 : ${sum}"

# 방법 2: C 스타일 for문 (동일 결과)
#!/bin/bash
sum=0
for (( num=1; num<=10; num++ ))   # C스타일: 초기식; 조건식; 증감식 — 변수 범위 가능
do
  sum=$(( sum + num ))
done
echo "1부터 10까지의 총 합 : ${sum}"

# EX2-2) 첫 번째 정수 ~ 두 번째 정수까지의 합 (입력값, 두 번째가 더 커야 함)
#!/bin/bash
read -p "첫 번째 정수 입력 : " num1
read -p "두 번째 정수 입력 : " num2

if [ "$num1" -gt "$num2" ]      # -gt: 크면 참; 입력 검증 후 조기 종료
then
    echo "두번째 정수가 더 커야합니다."
    exit 1
fi

sum=0
for (( i=num1; i<=num2; i++ ))   # {1..N} 범위 확장은 변수를 못 쓰므로 C스타일 사용
do
    sum=$((sum + i))
done
echo "$num1부터 $num2까지의 합 : $sum"
```

### 3-2. for 문 중첩 · 활용 실습

```bash
# EX4-1) 입력받은 정수의 구구단 출력
#!/bin/bash
read -p "정수를 입력하세요(1~9) : " dan
echo "============== ${dan} 단 =============="
for i in {1..9}
do
    echo "$dan x $i = $(( dan * i ))"   # $(( )): 곱셈 결과 반환
done

# EX4-2) 2~9단 전체 구구단 (이중 for문)
#!/bin/bash
for num in {2..9}
do
    echo "====== ${num} 단 ======"
    for i in {1..9}                 # 안쪽 루프: 각 단의 1~9 순회
    do
        echo "$num x $i = $(( num * i ))"
    done
    echo                            # 단 사이 빈 줄 출력
done

# EX5) 첫 번째 ~ 두 번째 정수까지 짝수/홀수 판별
#!/bin/bash
read -p "첫 번째 정수 입력 : " num1
read -p "두 번째 정수 입력 : " num2

if [ "$num1" -gt "$num2" ]
then
    echo "두번째 정수가 더 커야합니다."
    exit 1
fi

for (( i=num1; i<=num2; i++ ))
do
    if (( i % 2 == 0 ))            # (( )): 나머지가 0이면 짝수
    then
        echo "$i : 짝수"
    else
        echo "$i : 홀수"
    fi
done
```

### 3-3. break / continue 실습

```bash
# EX6) 1부터 10까지 출력하되 5를 만나면 반복 종료
for num in {1..10}
do
        if [ $num -eq 5 ]       # -eq: 값이 같으면 참
        then
                echo "숫자 5를 만나서 반복문을 종료합니다."
                break           # break: 반복문 자체를 즉시 탈출
        fi
        echo $num
done

# EX7) 루트 디렉터리 항목을 순회하며 passwd 파일 검색 후 발견 시 종료
items=(/*)                      # 글롭 /*: 루트 하위 전체 항목을 배열로 저장
for item in "${items[@]}"
do
    if [ -d "$item" ]           # -d: 디렉터리인지 확인
    then
        if [ -f "$item/passwd" ]   # -f: 해당 경로에 일반 파일 존재 여부
        then
            echo "passwd 파일 검색 완료 : $item/passwd"
            break               # 발견 즉시 종료 — 불필요한 순회 방지
        fi
    fi
done

# EX8) 1부터 15까지 출력하되 3의 배수는 건너뛰기
for num in {1..15}
do
    if [ $(( num % 3 )) -eq 0 ]   # $(( num % 3 )): 나머지 계산 후 0이면 3의 배수
    then
        echo "3의 배수인 $num 점프"
        continue                # continue: 이번 회차 나머지를 건너뛰고 다음 회차로
    fi
    echo $num
done

# EX9) 여러 로그 파일의 크기를 확인해 용량 경고/정상 판단
for file in /var/log/messages /var/log/secure /var/log/cron
do
    if [ ! -f "$file" ]         # 파일이 없으면 건너뜀
    then
        echo "$file 파일 또는 디렉터리가 없습니다."
        continue
    fi

    size=$(du -k "$file" | awk '{print $1}')   # du -k: KB 단위; awk 첫 필드(크기) 추출

    if [ "$size" -ge 10240 ]    # 10240KB = 10MB 초과 여부
    then
        echo "$file : 용량 초과 (${size}KB)"
    else
        echo "$file : 용량 정상 (${size}KB)"
    fi
done
```

### 3-4. while / until 실습

```bash
# EX3) 아이디·비밀번호를 최대 3번까지 확인 (실패 누적 시 종료)
#!/bin/bash
id="user1"
pw="admin1234"
count=1

while [ "$count" -le 3 ]
do
    read -p "아이디 : " input_id
    read -s -p "비밀번호 : " input_pw
    echo

    if [ "$input_id" = "$id" ] && [ "$input_pw" = "$pw" ]
    then
        echo "로그인 성공"
        break
    else
        echo "아이디 또는 비밀번호를 확인하세요"
        if [ "$count" -eq 3 ]
        then
            echo "아이디 또는 비밀번호를 3회 실패하여 종료합니다."
            exit 1
        fi
        ((count++))
    fi
done
echo "${id}님이 로그인 하셨습니다."

# EX4) 파일/디렉터리 생성 시 이름이 중복되면 숫자를 붙여 생성
#!/bin/bash
read -p "생성할 파일(f) 또는 디렉터리(d) 입력 : " type
read -p "생성할 전체 경로 입력 : " path

num=1
if [ -e "$path" ]
then
    while [ -e "$path$num" ]
    do
        ((num++))
    done
    path="${path}${num}"
fi

if [ "$type" = "f" ]
then
    mkdir -p "$(dirname $path)"
    touch "$path"
    echo "파일 생성 완료 ($path)"
elif [ "$type" = "d" ]
then
    mkdir -p "$path"
    echo "디렉터리 생성 완료 ($path)"
else
    echo "f또는 d만 입력해야합니다."
    exit 1
fi

# until EX1) 10부터 0까지 카운트다운
#!/bin/bash
num=10
until [ "$num" -lt 0 ]
do
    echo "카운트다운 : $num"
    num=$(( num - 1 ))
    sleep 1
done
echo "카운트다운 종료"

# until EX2) 특정 파일이 생성될 때까지 3초 간격으로 대기
#!/bin/bash
file="/temp/ready.flag"
echo "$file 파일이 생성될때까지 감지합니다."

until [ -f "$file" ]
do
    echo "$file 파일이 아직 없습니다. 대기중...."
    sleep 3
done
echo "${file} 파일이 생성되었습니다."
```

### 3-5. 파일 검사 + 용량 점검 실습 (du -h/-m, ls -l)

```bash
# 로그 파일 용량 점검 — du -m, du -h, ls -l 패턴
for file in /var/log/messages /var/log/secure /var/log/cron
do
    if [ ! -f "$file" ]
    then
        echo "$file 없음"
        continue
    fi

    # 읽기/실행 권한 확인
    if [ ! -r "$file" ]
    then
        echo "$file : 읽기 권한(-r) 없음"
        continue
    fi

    if [ -x "$file" ]
    then
        echo "$file : 실행 권한(-x) 있음 (로그 파일에 비정상)"
    fi

    size=$(du -m "$file" | awk '{print $1}')   # -m : MB 단위
    echo "$file : ${size}MB"
    du -h "$file"                               # -h : 사람이 읽기 좋은 단위 출력
done

ls -l /var/log/                                # -l : 상세 목록 출력
```

### 3-6. while 로그인 + 비밀번호 변수 패턴

```bash
# while 로그인 재시도 — $input_pass 비밀번호 변수 패턴
#!/bin/bash
id="admin"
pw="1234"
count=1

while [ "$count" -le 3 ]
do
    read -p "아이디  : " input_id
    read -s -p "비밀번호 : " input_pass
    echo

    if [ "$input_id" = "$id" ] && [ "$input_pass" = "$pw" ]
    then
        echo "로그인 성공"
        exit 0
    else
        echo "실패 ($count/3)"
    fi
    ((count++))
done
echo "3회 초과 — 종료"
exit 1
```

### 3-7. 파일 경로 처리 + seq 변수 범위 실습

```bash
# basename / dirname — 파일 경로에서 이름·상위 디렉터리 분리
config="/etc/ssh/sshd_config"
echo $(basename /etc/ssh/sshd_config)   # sshd_config
echo $(dirname /etc/ssh/sshd_config)    # /etc/ssh

# for 반복문에서 변수 범위가 필요할 때 seq 사용
read -p "시작 : " num1
read -p "끝   : " num2
for i in $(seq "$num1" "$num2")
do
    echo "$i"
done

# 파일 목록 순회에서 basename 활용 — 파일명만 추출해 비교
for f in /etc/ssh/*.conf
do
    fname=$(basename "$f")
    echo "설정 파일 : $fname"
done
```

### 3-8. 대량 계정/디렉터리 점검 실습

```bash
# EX10) user1~user10 계정과 홈 디렉터리 소유권·권한(700)을 일괄 점검/교정
#!/bin/bash
for user in user{1..10}
do
    if ! id "$user" > /dev/null 2>&1
    then
        useradd "$user"
        if [ $? -eq 0 ]
        then
            echo "$user 계정 생성 완료"
        else
            echo "$user 계정 생성 실패"
            continue
        fi
    fi

    home="/home/$user"

    if [ ! -d "$home" ]
    then
        mkdir -p "$home"
    fi

    owner=$(stat -c "%U" "$home")
    if [ "$owner" != "$user" ]
    then
        chown "$user:$user" "$home"
    fi

    permission=$(stat -c "%a" "$home")
    if [ "$permission" -ne 700 ]
    then
        chmod 700 "$home"
    fi
done
```

---

## 4. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 4-1. 필수 검증 명령어

```bash
bash -n <스크립트>              # 실행 전 문법 검사
echo $?                        # 반복문 내 각 명령 성공 여부 확인
stat -c "%U %G %a %n" <경로>    # 소유자/그룹/권한/경로 한 번에 확인
```

### 4-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. `for num in {1..$max}` 처럼 범위 확장에 변수를 사용했는데 의도대로 동작하지 않음

- **원인:** 중괄호 확장(`{1..N}`)은 **쉘이 스크립트를 파싱하는 시점**에 처리되므로 변수를 인식하지 못하고 문자 그대로 `{1..$max}` 취급된다.
- **해결:** C스타일 for문 또는 `seq` 명령으로 대체한다.
  ```bash
  for ((i=1; i<=max; i++)); do echo "$i"; done
  # 또는
  for i in $(seq 1 "$max"); do echo "$i"; done
  ```

#### 🚨 시나리오 2. `break` 를 중첩 반복문 안에서 사용했는데 바깥 루프까지 종료될 것으로 착각

- **원인:** `break` 는 기본적으로 **가장 안쪽 반복문만** 종료한다.
- **해결:** 바깥 루프까지 종료해야 한다면 `break 2` 처럼 레벨을 지정하거나, 별도의 상태 플래그 변수로 바깥 루프 종료 조건을 판단한다.

#### 🚨 시나리오 3. `while` 카운터 증가를 빠뜨려 무한 루프에 빠짐

- **증상:** 로그인 재시도 스크립트가 실패해도 계속 같은 입력을 반복 요구.
- **원인:** `count=$((count+1))` 또는 `((count++))` 누락.
- **해결:** 루프 본문 마지막에 반드시 카운터 증가 로직을 포함하고, `bash -n` 으로 사전 문법 검사를 습관화한다.

#### 🚨 시나리오 4. `until` 폴링 스크립트가 CPU를 과도하게 사용함

- **원인:** `sleep` 없이 조건만 계속 검사하여 반복문이 매우 빠르게 회전.
- **해결:** `sleep N` 을 반복문 안에 반드시 포함해 확인 간격을 둔다.
  ```bash
  until [ -f "$file" ]; do sleep 3; done
  ```

#### 🚨 시나리오 5. 대량 계정 점검 스크립트에서 `chown`/`chmod` 실패가 조용히 무시됨

- **원인:** 각 명령 뒤에 성공 여부(`$?`)를 확인하지 않고 다음 단계로 넘어감.
- **해결:** 단계별로 `$?` 를 확인하고 실패 시 로그를 남기거나 `continue` 로 다음 대상으로 넘어가는 방어 로직을 추가한다.
  ```bash
  chown "$user:$user" "$home"
  if [ $? -ne 0 ]; then
      echo "$user 소유권 변경 실패"
      continue
  fi
  ```

#### 🚨 시나리오 6. `for item in "${arr[*]}"` 사용 시 배열 요소가 하나로 합쳐져 반복이 1회만 실행됨

- **원인:** `"${arr[*]}"` 는 배열 전체를 하나의 문자열로 결합하기 때문.
- **해결:** 요소별로 개별 반복이 필요하면 `"${arr[@]}"` 를 사용한다.
  ```bash
  for item in "${arr[@]}"; do echo "$item"; done
  ```

---

> 📌 **핵심 요약**
> - `for` = 정해진 목록 순회, `while` = 참인 동안 반복, `until` = 거짓인 동안 반복(조건이 while과 반대)
> - `break` = 반복문 자체 종료, `continue` = 이번 회차만 건너뛰고 계속
> - 중괄호 확장 범위(`{1..N}`)에는 **변수 사용 불가** → C스타일 `for` 또는 `seq` 사용
> - 대량 반복 작업은 각 단계 `$?` 확인 + `continue` 방어 로직을 표준 패턴으로 사용
> - 관련: Shell Script - exit 상태와 test 명령 · Shell Script - 조건문 (if · case) · Shell Script - Metacharacters (메타문자)
