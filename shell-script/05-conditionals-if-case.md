# Shell Script - 조건문 (if · case)

> **Tag:** #Linux #ShellScript #Bash #조건문 #if #case #read
> **핵심 요약:** Bash 조건문은 TRUE/FALSE 값을 직접 판단하지 않고 **명령의 종료 코드**를 기준으로 판단한다(0=참, 0 이외=거짓). `if–then–fi` 는 순차적 조건 분기, `case–esac` 는 하나의 값에 대해 여러 패턴을 비교하는 분기에 적합하며, `read` 명령으로 사용자 입력을 받아 조건문과 결합하면 실무형 자동화 스크립트를 구성할 수 있다.

---

## 1. 개요 (Overview)

쉘 스크립트에는 별도의 boolean 타입이 없다. 대신 리눅스의 모든 명령이 실행 후 남기는 **종료 코드(exit status)** 를 참(0)/거짓(0 이외)의 기준으로 재사용한다. 이 덕분에 `if` 뒤에는 `[ 조건식 ]` 뿐 아니라 **어떤 명령이든** 올 수 있다. 예를 들어 `if ls /root` 처럼 조건식이 아닌 명령 자체를 `if` 뒤에 그대로 사용할 수 있으며, `ls` 가 성공(exit 0)하면 `then` 블록이 실행된다. 출력을 화면에 보이지 않게 하려면 `if ls /root > /dev/null 2>&1` 처럼 리다이렉션과 결합하면 된다.

`if–then–fi`, `if–else`, `if–elif–else` 는 구조적으로 차이가 있다. 단순 `if–then–fi` 는 조건이 참일 때만 실행하고 거짓이면 아무 것도 하지 않는다. `if–else` 는 참/거짓 각각의 처리를 지정한다. `if–elif–else` 는 **여러 조건을 순서대로 검사**하다가 처음으로 참이 되는 분기만 실행한다. `fi` 는 `if` 의 끝을 의미하며, Bash는 블록을 `fi`, `done`, `esac` 처럼 역순 키워드로 닫는다. 조건식은 `[ 표현식 ]` 형태로 작성하며 실제로는 `test` 명령과 동일하다. 산술 비교가 필요하면 `(( ))` 를, 문자열/파일 비교는 `[ ]` 를 사용한다(예: `if (( a == b / 2 )); then ...`).

조건문을 터미널에서 직접 입력하지 않고 스크립트 파일로 만드는 이유는, 반복 실행이 잦은 작업(서버 점검, 로그 정리, 백업)을 파일로 만들면 **재실행·자동화(cron 연동)·유지보수·공유·재현성 확보**가 모두 가능해지기 때문이다. 터미널에 직접 입력하는 조건문은 매번 다시 타이핑해야 하고, 자동화나 기록 관리가 불가능하다. 스크립트 파일은 반드시 실행 권한(`chmod +x`)이 있어야 `./script.sh` 형태로 직접 실행할 수 있으며, 권한이 없으면 `허가 거부(Permission denied)` 오류가 발생한다. 스크립트 작성 후 문법 오류를 사전에 확인하려면 `bash -n <스크립트>` 를 사용한다(실제 실행 없이 문법만 검사).

`case` 문은 **하나의 변수(또는 값)에 대해 여러 패턴을 비교(패턴매칭)** 할 때 `if–elif` 보다 코드가 짧고 가독성이 좋다. 메뉴 선택, 파일 확장자 처리, 서비스 상태(install/start/stop 등) 분기, y/n 입력 처리 등 문자열 비교가 여러 번 반복되는 상황에 특히 적합하다. 기본 형식은 `case 변수 in 패턴) 명령 ;; *) 기본실행문 ;; esac` 이며, `패턴3 | 패턴4)` 처럼 `|` 로 여러 패턴을 하나의 분기에 묶을 수 있다(예: `y|Y|yes|YES)`). `*)` 는 어떤 패턴에도 해당하지 않을 때 실행되는 else 역할이며, `esac` 는 `case` 를 거꾸로 읽은 종료 키워드다. `shopt -s nocasematch` 를 설정하면 대/소문자를 구분하지 않고 패턴을 비교할 수 있다.

`read` 는 표준 입력(키보드)에서 값을 읽어 지정한 변수에 저장하는 쉘 내장 명령이다. 스크립트 실행 중 이름, 나이, 비밀번호 등을 입력받아 조건문과 결합해 분기 처리할 때 사용한다. `read -p "메시지" 변수` 는 입력 안내 메시지를 출력한 뒤 입력받으며, `read -s -p "비밀번호 : " passwd` 는 `-s` 옵션으로 입력 내용을 화면에 표시하지 않아 비밀번호 입력에 필수적이다. `read -t 5 -p "..." 변수` 는 `-t` 옵션으로 초 단위 입력 대기 시간을 지정해 시간 초과 시 자동 종료되게 한다. 스크립트 내부에서 `exit 1` 을 호출하면 스크립트 실행을 즉시 중단하고 exit code 1을 반환하여, 잘못된 입력에 대한 조기 종료 로직을 구성할 수 있다.

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** Bash 기반 Linux 셸 환경 (RHEL 계열 기본 `/bin/bash`).

### Step 1. if 조건문 3대 형식 (동작 구조 주석 포함)

```bash
# 형식 1) 단순 if
if [ 조건 ]        # 조건식 평가 -> 종료코드 0(참)/그외(거짓) 반환
then                # 조건이 참일 때만 아래 블록으로 진입
    명령            # 참일 때 실행할 명령
fi                  # if 블록 종료 (거짓이면 then 블록을 건너뛰고 바로 여기로)

# 형식 2) if-else
if [ 조건 ]; then   # 조건 평가와 then을 한 줄로 작성 (세미콜론으로 구분)
    명령1           # 조건이 참일 때 실행
else                # 조건이 거짓일 때 진입하는 분기
    명령2           # 조건이 거짓일 때 실행
fi                  # if-else 블록 종료

# 형식 3) if-elif-else
if [ 조건1 ]        # ① 조건1을 먼저 평가
then
    명령1           # 조건1이 참이면 여기 실행 후 fi로 바로 이동 (elif/else는 검사 안 함)
elif [ 조건2 ]      # ② 조건1이 거짓일 때만 조건2 평가
then
    명령2           # 조건2가 참이면 여기 실행
else                # ③ 조건1, 조건2 모두 거짓일 때 진입
    명령3           # 어떤 조건에도 해당하지 않을 때 실행
fi                  # if-elif-else 블록 종료 (elif는 필요한 만큼 추가 가능)
```

### Step 2. 조건식 유형별 예시

```bash
# 숫자 비교
if [ "$x" -gt 5 ]; then echo "big"; fi

# 문자열 비교
if [ "$name" = "root" ]; then echo "admin user"; fi

# 파일 조건
if [ -e "/etc/hosts" ]; then echo "hosts exists"; fi

# 산술 비교 ( (( )) 사용, $ 생략 가능 )
if (( num % 2 == 0 )); then echo "even"; else echo "odd"; fi

# 명령 자체를 조건으로 사용 (출력 숨김)
if ls /root > /dev/null 2>&1; then
    echo "ls OK"
else
    echo "ls Fail"
fi
```

### Step 3. 스크립트 파일 표준 골격

```bash
vi ./script/check_hosts.sh
```
```bash
#!/bin/bash

if [ -e /etc/hosts ]
then
   echo "hosts OK"
else
   echo "hosts NOT FOUND"
fi
```
```bash
chmod +x ./script/check_hosts.sh
./script/check_hosts.sh
```

### Step 4. case 문 기본 형식 (동작 구조 주석 포함)

```bash
case 변수 in            # 검사할 대상(변수/값)을 지정, in으로 패턴 목록 시작
    패턴1)               # 변수 값이 패턴1과 일치하면 아래 명령 실행
        명령어1 ;;       # ;; = 이 분기 종료, esac으로 바로 이동 (다른 패턴 검사 안 함)
    패턴2)
        명령어2 ;;
    패턴3 | 패턴4)        # | 로 여러 패턴을 하나의 분기에 묶음 (OR 조건)
        명령어3 ;;
    *)                   # 위 패턴 중 어느 것에도 해당하지 않을 때 (else 역할)
        기본 실행문 ;;
esac                     # case 종료 키워드 (case를 거꾸로 읽은 형태)
```

### Step 5. read + case 조합 실전 템플릿

```bash
#!/bin/bash

read -p "설정할 Server(sshd, vsftpd, httpd)이름을 입력하세요 : " service
read -p "install/start/stop/restart/status 중 하나를 입력하세요 : " cmd

case "$cmd" in
    install)
        dnf install -y "$service" > /dev/null 2>&1
        echo "${service} 설치 완료"
        echo "상태 : $(systemctl is-active "$service")"
        ;;
    start)
        systemctl start "$service"
        echo "상태 : $(systemctl is-active "$service")"
        ;;
    stop)
        systemctl stop "$service"
        echo "상태 : $(systemctl is-active "$service")"
        ;;
    restart)
        systemctl restart "$service"
        echo "상태 : $(systemctl is-active "$service")"
        ;;
    *)
        echo "잘못 입력하셨습니다." ;;
esac
```

### Step 6. read 입력 검증 표준 패턴 (잘못된 입력 조기 종료)

```bash
#!/bin/bash

read -p "파일(f) / 디렉터리(d) : " type

if [ "$type" != "f" ] && [ "$type" != "d" ]
then
     echo "잘못된 입력입니다."
     exit 1                 # 스크립트 중단, exit code 1 반환
fi

read -p "생성할 경로 : " path
read -p "생성할 이름 : " name

if [ "$type" = "f" ]
then
    touch "${path}/${name}"
else
    mkdir "${path}/${name}"
fi
```

---

## 3. 실습 예제 모음 (EX)

### 3-1. if–then–fi 기본 실습

```bash
# EX0) num이 짝수인지 홀수인지 판단 (모듈로 연산 활용)
num=4
if (( $((num % 2)) == 0 ))
then
    echo "${num}은 짝수"
else
    echo "${num}은 홀수"
fi

# EX1) 변수 x=10 일 때, x가 5보다 크면 "big"을 출력하시오
x=10
if [ $x -gt 5 ]
then
    echo big
fi
# 결과 : big

# EX2) 변수 name="root" 일 때, name이 root이면 "admin user" 출력하시오
name="root"
if [ "$name" = "root" ]; then echo "admin user"; fi
# 결과 : admin user

# EX3) /home 디렉터리가 읽기 가능(-r)이면 "read ok" 출력하시오
if [ -r /home ]; then echo "read OK"; fi

# EX4) 변수 x=3 일 때, x가 0보다 크면 "positive"를 출력하시오
x=3
if [ $x -gt 0 ]; then echo "positive"; fi

# EX6) 변수 a=7, b=14 일 때 a가 b/2와 같으면 "half" 출력 (산술 비교는 (( )) 사용)
a=7; b=14
if (( a == b / 2 )); then echo "half"; fi
# 결과 : half

# EX7) 파일 /etc/hosts 가 존재하면 "hosts exists" 출력
if [ -e "/etc/hosts" ]; then echo "hosts exists"; fi

# EX9) 파일이 0바이트이면 내용을 추가하고 출력하는 스크립트
mkdir -p /backup
touch /backup/shellscript
file="/backup/shellscript"
if [ ! -s "$file" ]
then
    echo "New Data Insert" > "$file"
    cat "$file"
fi
```

### 3-2. if–then–else 실습

```bash
# EX1) /var/log/messages 파일이 존재하면 "found", 아니면 "not found" 출력
if [ -e "/var/log/messages" ]   # -e: 파일·디렉터리 존재 여부 테스트
then
    echo "found"
else
    echo "not found"
fi

# EX2) 입력받은 정수가 짝수면 "even", 홀수면 "odd" 출력
num=90348
if (( num % 2 == 0 ))           # (( )): 산술 평가; 나머지가 0이면 짝수
then
    echo "even"
else
    echo "odd"
fi

# EX3) /usr/bin/top 이 실행 가능하면 "run OK", 아니면 "no exec" 출력
if [ -x /usr/bin/top ]; then echo "run OK"; else echo "no exec"; fi   # -x: 실행 권한 있으면 참

# EX4-2) "ls /root" 성공 시 "ls OK", 실패 시 "ls Fail" (화면 출력은 숨김)
if ls /root > /dev/null 2>&1    # 명령 종료 코드로 판단; > /dev/null 2>&1 로 출력 전부 숨김
then
    echo "ls OK"
else
    echo "ls Fail"
fi

# EX7) /backup/soldesk 경로가 파일인지 디렉터리인지 없는지 판단 (중첩 if)
path="/backup/soldesk"
if [ -e "$path" ]               # 1단계: 존재 여부
then
    if [ -f "$path" ]           # 2단계: 일반 파일인지 (-f), 아니면 디렉터리
    then
        echo "$path는 파일입니다."
    else
        echo "$path는 디렉터리입니다."
    fi
else
    echo "$path 파일 또는 디렉터리가 없습니다."
fi
```

### 3-3. 스크립트 파일 + 자동화 실습

```bash
# EX2) /var/log/messages 크기가 50MB 이상이면 경고 메시지 출력
#!/bin/bash
size=$(du -m /var/log/messages | awk '{print $1}')   # du -m: MB 단위 크기; awk로 숫자 필드만 추출

if (( size >= 50 ))             # (( )): 정수 비교 산술 평가
then
   echo "LOG SIZE WARNING : ${size}MB"
else
   echo "LOG SIZE OK : ${size}MB"
fi

# EX4) SSHD 상태가 active이면 메시지 출력, 아니면 INACTIVE 출력
#!/bin/bash
status=$(systemctl is-active sshd)   # is-active: "active" / "inactive" 문자열 반환

if test "$status" = "active"    # test = [ ]; 문자열 동등 비교 (= 사용)
then
  echo "SSHD STATUS ACTIVE"
else
  echo "SSHD INACTIVE"
fi

# EX6) 루트 파일 시스템 사용률이 60% 이상이면 경고 출력
#!/bin/bash
use=$(df -h / | grep / | awk '{print $5}' | tr -d '%')   # df $5: 사용률 필드; tr -d '%': % 문자 제거

if [ "$use" -ge 60 ]; then      # -ge: 크거나 같으면 참 (greater or equal)
    echo "DISK WARNING: $use% 사용 중"
else
    echo "DISK OK: $use%"
fi

# EX7) 로그 파일 크기 확인 — $size / ${file} 패턴
#!/bin/bash
for file in /var/log/messages /var/log/secure
do
    if [ ! -f "${file}" ]       # -f: 일반 파일; !: 부정 → 파일이 없으면 참
    then
        echo "${file} 없음"
        continue
    fi
    size=$(du -m "${file}" | awk '{print $1}')   # MB 단위 크기
    if [ "$size" -ge 50 ]       # -ge: 50 이상 → 경고
    then
        echo "${file} : ${size}MB — 경고"
    elif [ "$size" -lt 10 ]     # -lt: 10 미만 → 정상
    then
        echo "${file} : ${size}MB — 정상"
    elif [ "$size" -ne 0 ]      # -ne: 0이 아니면 → 주의 (10~49 MB 범위)
    then
        echo "${file} : ${size}MB — 주의"
    fi
done

# EX8) 파일 복사 후 대상 존재 확인 — basename 언쿼트 패턴
#!/bin/bash
read -p "복사할 파일 : " src
read -p "대상 경로  : " dest

filename=$(basename $path)
target="${dest}/$(basename ${src})"   # 복사 대상 전체 경로 조합

if [ -w "$dest" ]               # -w: 쓰기 권한 있으면 참
then
    cp -r "$src" "$dest"
    echo "복사 완료 → $target"
else
    echo "쓰기 권한 없음 : $dest"
fi
```

### 3-4. read + if 조합 실습

```bash
# EX0) 아이디와 나이를 입력받아 성인 여부 판단
#!/bin/bash
read -p "아이디 : " uid
read -p "나이 : " age

if [ "$age" -ge 19 ]            # -ge: 19 이상이면 성인
then
    echo "${uid} 님은 성인입니다."
else
    echo "${uid} 님은 미성년자입니다."
fi

# EX00) /tmp 파일 수 확인 후 임계 초과 시 경고
#!/bin/bash
count=$(ls -l /tmp | wc -l)    # wc -l: 줄 수(=파일+헤더 라인) 카운트
if [ "$count" -gt 100 ]         # -gt: 100 초과이면 참 (greater than)
then
    echo "/tmp 파일 과다 : ${count}개"
fi

# EX1) 파일/디렉터리명 존재 여부를 입력받아 판단
#!/bin/bash
read -p "경로 입력 : " path
read -p "파일 또는 디렉터리명 입력 : " name

if [ -f "$path/$name" ]         # -f: 일반 파일이면 참
then
   echo "${name}는 파일입니다."
elif [ -d "$path/$name" ]       # -d: 디렉터리이면 참
then
   echo "${name}는 디렉터리입니다."
else
   echo "${name}는 존재하지 않는 파일 또는 디렉터리 입니다."
fi

# EX8) 3개의 계정 정보 중 하나와 아이디/비밀번호가 일치하면 로그인 성공
#!/bin/bash
id1="user1"; pw1="pw11"
id2="user2"; pw2="pw22"
id3="user3"; pw3="pw33"

read -p "아이디 : " input_id
read -s -p "비밀번호 : " input_pw   # -s: 입력 내용 숨김 (silent)
echo                            # -s 후 줄바꿈 강제 출력

if [ "$input_id" = "$id1" ] && [ "$input_pw" = "$pw1" ]   # &&: 두 조건 모두 참이어야 성공
then
    echo "로그인 성공"
elif [ "$input_id" = "$id2" ] && [ "$input_pw" = "$pw2" ]
then
    echo "로그인 성공"
elif [ "$input_id" = "$id3" ] && [ "$input_pw" = "$pw3" ]
then
    echo "로그인 성공"
else
    echo "아이디 또는 비밀번호를 확인하세요"
fi
```

### 3-5. case 문 실습

```bash
# EX1) 입력한 번호에 따라 다른 메시지 출력
#!/bin/bash
read -p "정수입력(1/2/3) : " num

case $num in                    # 변수 값을 패턴과 순서대로 비교
 1)
    echo "1번 선택) cmd 1 running";;   # ;;: 해당 패턴 실행 후 case 종료
 2)
    echo "2번 선택) cmd 2 running";;
 3)
    echo "3번 선택) cmd 3 running";;
 *)
    echo "잘못된 입력입니다.";;   # *): 위 패턴에 해당하지 않는 모든 경우 (기본값)
esac

# EX2) y/n 입력에 따라 진행/종료 분기 (대소문자 구분 없이 처리)
#!/bin/bash
shopt -s nocasematch            # 이후 패턴 비교에서 대/소문자 구분 무시

read -p "게임을 계속 진행하시겠습니까? (y/n) : " answer

case "$answer" in
    y|yes)                      # |: OR 패턴 — y 또는 yes 둘 다 매칭
        echo "YES 선택 - 게임을 계속 진행합니다.";;
    n|no)
        echo "NO 선택 - 게임을 종료합니다.";;
    *)
        echo "잘못 입력했습니다. y 또는 n를 입력하세요";;
esac

# EX3-1) 서비스 제어 스크립트 (install/start/stop/restart)
#!/bin/bash
read -p "install/start/stop/restart/status 중 하나를 입력하세요:" cmd

case "$cmd" in
    install)
        dnf install -y vsftpd           # -y: 설치 확인 없이 자동 진행
        echo "vsftpd 설치 완료"
        echo "상태 : $(systemctl is-active vsftpd)";;
    start)
        systemctl start vsftpd
        echo "vsftpd Start"
        echo "상태 : $(systemctl is-active vsftpd)";;
    stop)
        systemctl stop vsftpd
        echo "vsftpd Stop"
        echo "상태 : $(systemctl is-active vsftpd)";;
    restart)
        systemctl restart vsftpd
        echo "vsftpd Restart"
        echo "상태 : $(systemctl is-active vsftpd)";;
    *)
        echo "잘못 입력하셨습니다.";;
esac

# EX4) 서비스명 + add/remove 입력에 따라 방화벽 서비스를 등록/삭제
#!/bin/bash
read -p "방화벽에 적용할 서비스명을 입력하세요 (예: http, https, ftp): " servicename
read -p "서비스를 방화벽에 추가(add) 또는 삭제(remove)하세요: " action

case "$action" in
    add)
        if firewall-cmd --permanent --add-service="$servicename" > /dev/null 2>&1   # --permanent: 재부팅 후에도 유지
        then
            firewall-cmd --reload > /dev/null 2>&1   # --reload: 규칙 즉시 재적용
            echo "$servicename 서비스 방화벽 추가 완료"
        else
            echo "$servicename 서비스 방화벽 추가 실패"
        fi
        ;;
    remove)
        if firewall-cmd --permanent --remove-service="$servicename" > /dev/null 2>&1
        then
            firewall-cmd --reload > /dev/null 2>&1
            echo "$servicename 서비스 방화벽 삭제 완료"
        else
            echo "$servicename 서비스 방화벽 삭제 실패"
        fi
        ;;
    *)
        echo "지원하지 않는 명령입니다. add 또는 remove 를 입력하세요.";;
esac
```

### 3-6. case 문 + 패키지 관리 패턴

```bash
# EX5) 서비스명으로 패키지를 판별해 설치하는 case 패턴
#!/bin/bash
read -p "설치할 서비스 (sshd/vsftpd) : " service

case "$service" in
    sshd)
        package="openssh-server";;
    vsftpd)
        package="vsftpd";;
    *)
        package="$service";;
esac

dnf install -y "$package" > /dev/null 2>&1
echo "${package} 설치 완료"
echo "상태 : $(systemctl is-active "$service")"

# awk $NF 활용 — 로그에서 마지막 필드 추출
# $NF : awk의 마지막 필드 변수 (number of fields)
last_login=$(last -n 1 | awk '{print $NF}')
echo "최근 로그인 항목 : $last_login"
```

### 3-7. 파일 복사 + 대상 확인 패턴

```bash
# 파일 복사 후 대상($target) 존재 여부 확인
#!/bin/bash
read -p "복사할 파일 : " src
read -p "대상 경로  : " dest

filename=$(basename $src)
target="$dest/$filename"

cp -r "$src" "$dest"

if [ -f "$target" ]
then
    echo "${filename} 복사 완료 → $target"
else
    echo "복사 실패"
fi

# HOME 디렉터리 기반 파일 경로 사용
if [ -d "${HOME}/.ssh" ]
then
    echo "SSH 설정 디렉터리 존재 : ${HOME}/.ssh"
fi
```

### 3-8. 경로 처리 · 사용자 판별 · find 조건 심화 실습

```bash
# EX1) 현재 사용자가 root인지 판단해 다른 명령 실행
#!/bin/bash
user=$(whoami)                  # whoami: 현재 실행 사용자 이름 출력

if [ "$user" = "root" ]         # 문자열 비교 — = 연산자, 따옴표 필수
then
    echo "root 사용자 : HOME = $HOME"
    ls -l $HOME
else
    echo "일반 사용자 : pwd = $(pwd)"
fi

# EX2) 파일 복사 후 basename으로 결과 검증
#!/bin/bash
read -p "복사할 파일 경로 : " src
read -p "복사 대상 디렉터리 : " dest

filename=$(basename "$src")     # basename: 경로에서 파일명만 추출 (/tmp/test.txt → test.txt)

cp -r "$src" "$dest"

if [ -f "$dest/$filename" ]     # 복사 후 대상 파일 존재 여부로 성공 판단
then
    echo "${filename} 파일이 복사되었습니다."
else
    echo "복사 실패"
fi

# EX3) 디렉터리를 tar.gz 압축하고 파일 존재 여부 확인
#!/bin/bash
read -p "압축할 디렉터리 : " src
read -p "저장할 디렉터리 : " dest

name=$(basename "$src")         # 디렉터리 이름만 추출 (/etc/ssh → ssh)

tar -zcf "${dest}/${name}.tar.gz" -C "$(dirname "$src")" "$name"   # -C: 기준 디렉터리 변경 후 상대경로 압축

if [ -f "${dest}/${name}.tar.gz" ]   # 압축 파일 생성 여부 확인
then
    echo "${name}.tar.gz 압축 성공"
else
    echo "압축 실패"
fi

# EX4) 이름으로 파일(f) 또는 디렉터리(d) 검색 후 결과 판단
#!/bin/bash
read -p "검색할 이름 : " name
read -p "파일(f) 또는 디렉터리(d) : " type

if [ "$type" = "f" ]
then
    result=$(find / -type f -name "$name" 2>/dev/null)   # 2>/dev/null: stderr(권한 오류) 숨김
elif [ "$type" = "d" ]
then
    result=$(find / -type d -name "$name" 2>/dev/null)
else
    echo "잘못된 입력 (f 또는 d 만 허용)"
    exit 1
fi

if [ -n "$result" ]             # -n: 문자열이 비어 있지 않으면 참 → 검색 결과 있음
then
    echo "검색 결과 :"
    echo "$result"
else
    echo "${name} 을 찾을 수 없습니다."
fi
```

> - `basename 경로` : 경로에서 파일명(마지막 구성요소)만 추출. `basename /etc/ssh/sshd_config` → `sshd_config`
> - `dirname 경로` : 경로에서 디렉터리 부분만 추출. `dirname /etc/ssh/sshd_config` → `/etc/ssh`
> - `whoami` : 현재 실행 사용자 이름 출력 (`$USER` 와 동일하지만 명령으로 호출)
> - `find / -type f -name 이름 2>/dev/null` : stderr 억제하며 이름으로 파일 검색

---

## 4. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 4-1. 필수 검증 명령어

```bash
bash -n <스크립트>            # 실행 없이 문법만 검사
chmod +x <스크립트>            # 실행 권한 부여
ls -l <스크립트>               # 실행 권한(x) 부여 여부 확인
echo $?                       # 스크립트 종료 코드 확인
```

### 4-2. 트러블슈팅 시나리오

#### 시나리오 1. 스크립트 파일을 실행했는데 `허가 거부(Permission denied)` 발생

- **원인:** 파일에 실행 권한(`x`)이 없음(`rw-r--r--` 상태).
- **해결:**
  ```bash
  chmod +x ./script/check_hosts.sh
  ./script/check_hosts.sh
  ```

#### 시나리오 2. `if [ $x -gt 0 ]` 실행 시 예상과 다른 분기로 진입

- **원인 후보:** ① 변수 `$x` 가 비어 있어 조건식이 깨짐, ② 문자열 값을 숫자 비교 연산자로 비교.
- **해결:** 변수를 큰따옴표로 감싸고, 값의 성격(숫자/문자열)에 맞는 비교 연산자를 사용한다.
  ```bash
  if [ "$x" -gt 0 ]; then echo "positive"; fi
  ```

#### 시나리오 3. `case` 문에서 `*)` 분기가 항상 실행됨

- **원인 후보:** ① 패턴과 실제 입력값의 대소문자가 다름, ② 변수를 따옴표 없이 사용해 공백 등으로 패턴이 어긋남.
- **해결:**
  ```bash
  case "$answer" in       # 반드시 큰따옴표로 감싼 변수 사용
      y|Y) echo "YES";;
      n|N) echo "NO";;
      *) echo "잘못된 입력";;
  esac

  # 대소문자 구분 없이 처리하려면
  shopt -s nocasematch
  ```

#### 시나리오 4. `read -s` 로 비밀번호를 입력받았는데 다음 줄 출력이 같은 줄에 붙어 나옴

- **원인:** `-s` 옵션은 입력 내용만 숨길 뿐, 입력 후 줄바꿈까지 자동으로 처리하지 않음.
- **해결:** `read` 직후 빈 `echo` 를 추가해 줄바꿈을 강제한다.
  ```bash
  read -s -p "비밀번호 : " passwd
  echo
  echo "입력한 비밀번호 : $passwd"
  ```

#### 시나리오 5. `read -t` 로 시간 제한을 걸었는데 시간 초과 시 후속 로직이 비정상 동작

- **원인:** 시간 초과로 `read` 가 실패해도 이후 코드가 그대로 실행되어 빈 변수로 로직이 진행됨.
- **해결:** `read` 의 종료 코드를 확인해 시간 초과를 명시적으로 처리한다.
  ```bash
  if ! read -t 10 -p "입력 : " val; then
      echo "입력 시간 초과로 종료합니다."
      exit 1
  fi
  ```

#### 시나리오 6. 여러 `if` 를 중첩했더니 로직 추적이 어려워짐

- **원인:** 파일/디렉터리 존재 여부처럼 단계적 판단이 필요한 로직을 깊게 중첩(nested if)해서 가독성 저하.
- **해결/예방:** 패턴이 3개 이상으로 늘어나면 `case` 문으로 전환하거나, `elif` 체인으로 평탄화하는 것을 우선 검토한다.

---

>  **핵심 요약**
> - 조건 판단은 항상 **종료 코드(0=참)** 기준
> - `if` 뒤에는 `[ 조건식 ]` 뿐 아니라 **어떤 명령**이든 올 수 있음
> - 패턴 비교가 여러 개면 `if-elif` 보다 **`case`** 가 더 명확
> - `read` 로 입력을 받을 땐 잘못된 값에 대해 `exit 1` 로 조기 종료하는 방어 로직을 습관화
> - 관련: Shell Script - expr · let (산술 연산) · Shell Script - exit 상태와 test 명령 · Shell Script - 반복문 (for · while · until)
