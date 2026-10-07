# cat 리다이렉션 & Heredoc

> **Tag:** #Linux #cat #Redirection #Heredoc #Shell #Automation  
> **핵심 요약:** `>`는 표준 출력을 파일에 덮어쓰고, `>>`는 기존 파일 끝에 추가한다. Heredoc은 여러 줄의 입력을 명령에 전달하며, 종료 구분자를 따옴표로 감싸면 변수·명령 치환을 억제할 수 있다.

---

## 1. 개요 (Overview)

`>`와 `>>`의 차이는 다음과 같다.

| 연산자 | 동작 |
|---|---|
| `>` | 파일 생성 또는 기존 파일 내용 제거 후 덮어쓰기 |
| `>>` | 파일이 없으면 생성하고, 있으면 마지막에 추가 |
| `<` | 파일을 명령의 표준 입력으로 전달 |
| `2>` | 표준 오류를 파일로 전달 |
| `&>` | Bash에서 표준 출력과 표준 오류를 함께 전달 |

```bash
command > file
command >> file
```

> **참고:** `>`와 `>>`는 `cat` 전용 기능이 아니라 셸이 처리하는 리다이렉션 연산자이다. `echo`, `printf`, `ls`, `find` 등 다른 명령에도 사용할 수 있다.

`>`가 위험한 이유는 셸이 명령 실행 전에 목적지 파일을 먼저 열고 기존 내용을 비우기 때문이다.

```bash
echo "new line" > existing.conf
```

기존 `existing.conf` 내용은 사라지고 `new line`만 남는다. 특히 다음 명령은 원본 파일을 먼저 비우기 때문에 안전한 자기 복사가 되지 않는다.

```bash
cat file > file
```

파일 내용이 제거될 수 있으므로 실행하면 안 된다.

Ctrl+D와 Heredoc의 `EOF`가 같은 것인지에 대해서는, 관련은 있지만 동일한 개념은 아니다. `Ctrl + D`는 터미널 드라이버가 현재 입력의 끝을 프로그램에 알리도록 하는 키 입력으로, 보통 빈 줄에서 누르면 `cat`에 EOF 상태가 전달되어 종료되며 입력 중인 문자가 있다면 해당 입력을 먼저 전달할 수 있다. `EOF`는 Heredoc에서 흔히 사용하는 **종료 구분자 이름**으로, 예약어가 아니므로 다른 문자열을 사용해도 된다.

```bash
cat <<END
hello
END
```

다음도 동일하게 동작한다.

```bash
cat <<MY_TEXT
hello
MY_TEXT
```

---

## 2. 표준 사용 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. `>` — 파일 생성 및 덮어쓰기

파일 내용을 다른 파일로 출력한다.

```bash
cat /etc/passwd > /home/guest/newfile
```

미리 `touch`로 생성할 필요는 없다.

```bash
cat /etc/passwd > /home/guest/newfile
```

다음 두 단계를 사용해도 되지만 첫 번째 명령은 필수가 아니다.

```bash
touch /home/guest/newfile
cat /etc/passwd > /home/guest/newfile
```

복사 결과 확인:

```bash
ls -l /etc/passwd /home/guest/newfile
diff -- /etc/passwd /home/guest/newfile &&
echo "내용 일치"
```

> **참고:** 파일 크기가 같다는 사실만으로 내용까지 동일하다고 확정할 수 없다. `diff` 또는 `sha256sum`으로 검증한다.

### Step 2. 키보드 입력으로 파일 생성

```bash
cat > /home/guest/newfile2
```

내용 입력:

```text
hello world~~!!
hello soldesk~~!!
```

입력을 종료하고 저장:

```text
Ctrl + D
```

결과 확인:

```bash
cat /home/guest/newfile2
```

같은 명령을 다시 실행하면 기존 내용이 삭제된다.

```bash
cat > /home/guest/newfile2
```

```text
리다이렉션 연산자를 사용합니다.
Ctrl + D
```

확인:

```bash
cat /home/guest/newfile2
```

### Step 3. 빈 파일 생성 또는 파일 내용 비우기

빈 파일 생성:

```bash
> **참고:** /tmp/empty-file
```

기존 파일의 크기를 0으로 만든다.

```bash
> **참고:** /var/log/app.log
```

명령을 명확하게 표시하려면:

```bash
: > /var/log/app.log
```

> **참고:** 운영 로그를 직접 비우기 전에 서비스 동작, logrotate 정책 및 감사 요구사항을 확인해야 한다.

### Step 4. `>>` — 기존 파일 끝에 내용 추가

```bash
cat > /home/guest/newfile3
```

```text
이것은 newfile3의 원본 내용입니다.
Ctrl + D
```

내용 추가:

```bash
cat >> /home/guest/newfile3
```

```text
새로운 내용을 추가합니다.
이 내용은 기존 내용 다음에 추가됩니다.
Ctrl + D
```

결과 확인:

```bash
cat /home/guest/newfile3
```

간단한 한 줄 추가에는 `printf`가 더 명확하다.

```bash
printf '%s\n' '새로운 내용을 추가합니다.' >> /home/guest/newfile3
```

### Step 5. 여러 파일 병합

실습 파일 생성:

```bash
cat > /home/guest/a
```

```text
동해물과 백두산이 마르고 닳도록
하느님이 보우하사 우리나라 만세
Ctrl + D
```

```bash
cat > /home/guest/b
```

```text
무궁화 삼천리 화려강산
대한 사람 대한으로 길이 보전하세
Ctrl + D
```

두 파일을 화면에 이어서 출력:

```bash
cat /home/guest/a /home/guest/b
```

두 파일을 `c`로 병합:

```bash
cat /home/guest/a /home/guest/b > /home/guest/c
```

확인:

```bash
cat /home/guest/c
```

검증:

```bash
cat /home/guest/a /home/guest/b |
diff - /home/guest/c
```

### Step 6. 여러 명령의 결과를 한 파일에 저장

덮어쓰기:

```bash
{
    echo "===== SYSTEM REPORT ====="
    date
    hostnamectl
    df -h
    free -m
} > /tmp/system-report.txt
```

기존 파일 끝에 추가:

```bash
{
    echo "===== REPORT: $(date) ====="
    df -h
    free -m
} >> /var/log/system-report.log
```

### Step 7. Heredoc 기본 사용

변수와 명령 치환 허용:

```bash
cat > /tmp/motd.example <<EOF
Welcome to $(hostname)
Current user: $USER
Kernel: $(uname -r)
EOF
```

확인:

```bash
cat /tmp/motd.example
```

### Step 8. Heredoc 변수 치환 억제

종료 구분자를 따옴표로 감싸면 변수, 명령 및 산술 치환이 실행되지 않는다.

```bash
cat > /tmp/app.conf <<'EOF'
server {
    listen 80;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

`$uri`가 셸 변수로 치환되지 않고 그대로 저장된다.

### Step 9. Heredoc으로 기존 파일에 내용 추가

```bash
cat >> /home/guest/testfile <<'EOF'
새로운 내용을 추가합니다.
Heredoc과 >>를 사용하면 기존 내용 뒤에 추가됩니다.
EOF
```

확인:

```bash
tail -n 5 /home/guest/testfile
```

### Step 10. `<<-EOF` — 선행 탭 제거

```bash
cat > /tmp/hello.sh <<-'EOF'
	#!/bin/bash
	echo "Hello, World"
	echo "Today is $(date)"
EOF
```

- `<<-`는 각 줄 앞의 **리터럴 탭 문자**를 제거한다.
- 공백 문자는 제거하지 않는다.
- 위 예시는 구분자를 따옴표로 감쌌기 때문에 `$(date)`가 파일 생성 시 실행되지 않고 원문으로 저장된다.

### Step 11. 표준 오류 리다이렉션

표준 출력:

```bash
command > /tmp/stdout.log
```

표준 오류:

```bash
command 2> /tmp/stderr.log
```

표준 출력과 오류를 같은 파일에 저장:

```bash
command > /tmp/all.log 2>&1
```

Bash 전용 축약형:

```bash
command &> /tmp/all.log
```

표준 출력은 파일, 표준 오류는 별도 파일:

```bash
command > /tmp/out.log 2> /tmp/err.log
```

### Step 12. `tee` — 화면 출력과 파일 저장

덮어쓰기:

```bash
command | tee /tmp/output.log
```

추가:

```bash
command | tee -a /tmp/output.log
```

Root 권한이 필요한 파일에 추가:

```bash
printf '%s\n' 'net.ipv4.ip_forward=1' |
sudo tee -a /etc/sysctl.conf > /dev/null
```

Root 권한으로 파일 전체 생성:

```bash
sudo tee /etc/example.conf > /dev/null <<'EOF'
option1=value1
option2=value2
EOF
```

### Step 13. `noclobber`로 우발적 덮어쓰기 방지

현재 셸에서 활성화:

```bash
set -o noclobber
```

기존 파일에 `>`를 사용하면 실패한다.

```bash
echo "test" > existing-file
```

의도적으로 강제 덮어쓰기:

```bash
echo "test" >| existing-file
```

해제:

```bash
set +o noclobber
```

> **참고:** `noclobber`는 모든 형태의 파일 변경을 막는 보안 기능이 아니다. `>>`, `rm`, 편집기, `>|` 등으로 기존 파일은 여전히 변경될 수 있다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
ls -lh <파일>                    # 논리적 크기
wc -l <파일>                     # 줄 수
tail -n 10 <파일>                # 추가된 마지막 부분
file <파일>                      # 파일 형식
stat <파일>                      # 메타데이터
diff -- <원본> <사본>            # 내용 비교
sha256sum <원본> <사본>          # 체크섬 비교
```

### 3-2. 대표 트러블슈팅

#### 시나리오 1. `>` 사용 후 설정 파일 내용이 사라졌다

원인:

```bash
echo "설정 한 줄" > /etc/example.conf
```

`>`가 기존 파일을 먼저 비웠기 때문이다.

변경 전 백업:

```bash
cp -a /etc/example.conf \
  "/etc/example.conf.bak.$(date +%F-%H%M%S)"
```

단순 추가가 목적이라면:

```bash
printf '%s\n' '설정 한 줄' >> /etc/example.conf
```

> **참고:** 설정 파일에 같은 지시문을 반복해서 추가하면 서비스에 따라 첫 번째 값 또는 마지막 값이 적용되거나 문법 오류가 발생할 수 있다. 가능하면 전용 설정 도구, drop-in 파일 또는 정확한 편집 방식을 사용한다.

예를 들어 SSH 설정은 별도 drop-in 파일을 사용할 수 있다.

```bash
cat > /etc/ssh/sshd_config.d/99-local.conf <<'EOF'
PermitRootLogin no
MaxAuthTries 3
EOF

sshd -t
systemctl reload sshd
```

#### 시나리오 2. Heredoc의 `$uri`가 빈 문자열로 바뀌었다

잘못된 예:

```bash
cat > /tmp/app.conf <<EOF
try_files $uri $uri/ =404;
EOF
```

수정:

```bash
cat > /tmp/app.conf <<'EOF'
try_files $uri $uri/ =404;
EOF
```

#### 시나리오 3. `sudo echo ... >> 파일`이 권한 오류로 실패한다

잘못된 예:

```bash
sudo echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```

`echo`만 Root 권한으로 실행되고 `>>`는 현재 일반 사용자 셸이 처리한다.

권장 방법:

```bash
printf '%s\n' 'net.ipv4.ip_forward=1' |
sudo tee -a /etc/sysctl.conf > /dev/null
```

대안:

```bash
sudo sh -c \
  'printf "%s\n" "net.ipv4.ip_forward=1" >> /etc/sysctl.conf'
```

#### 시나리오 4. `cat file > file` 실행 후 파일이 비었다

셸이 `cat` 실행 전에 목적지 파일을 비운 것이 원인이다.

파일 복사는 다음 명령을 사용한다.

```bash
cp file file.copy
```

필터 결과로 같은 파일을 교체해야 한다면 임시 파일을 사용한다.

```bash
tmp=$(mktemp) || exit 1

sed 's/old/new/g' file > "$tmp" &&
cat "$tmp" > file

rm -f "$tmp"
```

더 안전하게 동일 파일시스템에서 교체:

```bash
tmp=$(mktemp ./file.XXXXXX) || exit 1

if sed 's/old/new/g' file > "$tmp"; then
    chmod --reference=file "$tmp"
    chown --reference=file "$tmp"
    mv -- "$tmp" file
else
    rm -f -- "$tmp"
    exit 1
fi
```

#### 시나리오 5. 패키지를 재설치했지만 설정 파일이 복구되지 않는다

패키지 관리자는 수정된 설정 파일을 보호하기 위해 기존 파일을 유지하거나 `.rpmnew`, `.rpmsave`를 생성할 수 있다. 재설치가 항상 기본 설정 복원을 의미하지는 않는다.

관련 파일 확인:

```bash
find /etc -type f \
  \( -name '*.rpmnew' -o -name '*.rpmsave' \) -print
```

패키지 파일 목록 확인:

```bash
rpm -ql openssh-server
```

패키지 검증:

```bash
rpm -V openssh-server
```

가장 안전한 복구 순서:

1. 별도 SSH 세션 또는 VMware 콘솔 확보
2. 현재 손상 파일 백업
3. 기존 백업·스냅샷 확인
4. `.rpmnew`, `.rpmsave` 확인
5. 패키지 기본 파일과 비교
6. 문법 검사 후 서비스 다시 로드

SSH 문법 확인:

```bash
sshd -t
```

#### 시나리오 6. `Ctrl + D`를 눌렀는데 `cat`이 즉시 종료되지 않는다

현재 줄에 입력된 문자가 있다면 `Ctrl + D`가 해당 입력을 먼저 프로그램에 전달할 수 있다. 새 빈 줄에서 다시 `Ctrl + D`를 누른다.

강제 중단이 필요하면:

```text
Ctrl + C
```

>  **핵심 요약**
> - `>`: 생성 또는 덮어쓰기
> - `>>`: 기존 파일 끝에 추가
> - `Ctrl + D`: 터미널 입력 종료 전달
> - `<<EOF`: 치환 허용 Heredoc
> - `<<'EOF'`: 치환 억제 Heredoc
> - Root 파일 기록: `sudo tee`
> - 덮어쓰기 방지: `set -o noclobber`
> - 관련: 파일 내용 출력 6종 (cat · head · tail · more · less · nl) · 파일·디렉터리 검색 (find)
