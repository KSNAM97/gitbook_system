# 👀 파일 내용 출력 6종 (cat · head · tail · more · less · nl)

> **Tag:** #Linux #cat #head #tail #more #less #nl #LogAnalysis  
> **핵심 요약:** 파일 크기와 조회 목적에 따라 `cat`, `head`, `tail`, `more`, `less`, `nl`을 구분해서 사용한다. 짧은 파일은 `cat`, 일부 범위는 `head`·`tail`, 대용량 파일은 `less`, 실시간 로그는 `tail -F`가 적합하다.

---

## 1. 📖 개요 (Overview)

파일 조회 명령어는 상황에 따라 다음과 같이 구분해서 사용한다.

| 상황 | 권장 명령어 | 특징 |
|---|---|---|
| 짧은 파일 전체 출력 | `cat` | 전체 내용을 표준 출력으로 전달 |
| 파일 앞부분 확인 | `head` | 기본 앞 10줄 |
| 파일 뒷부분 확인 | `tail` | 기본 뒤 10줄 |
| 실시간 로그 확인 | `tail -F` | 파일 교체·로그 회전에 대응 |
| 페이지 단위 조회 | `more` | 단순한 페이저 |
| 대용량 파일 조회·검색 | `less` | 양방향 이동과 검색 지원 |
| 줄 번호와 함께 출력 | `nl` | 기본적으로 내용이 있는 줄에 번호 부여 |

대용량 로그에 `cat` 사용을 피해야 하는 이유는, `cat`이 파일을 한 번에 메모리에 전부 적재하는 명령은 아니지만 파일 전체를 빠르게 터미널로 출력하기 때문이다. 수 GB 크기의 로그에 실행하면 SSH 네트워크 트래픽 증가, 터미널 렌더링 지연, 스크롤 버퍼 증가, 필요한 내용을 찾기 어려움, 세션이 멈춘 것처럼 보이는 현상 같은 문제가 발생할 수 있다. 대용량 파일은 다음 명령을 사용한다.

```bash
less /var/log/messages
tail -n 500 /var/log/messages
tail -F /var/log/app.log
```

파일 크기를 먼저 확인한다.

```bash
ls -lh /var/log/messages
stat /var/log/messages
du -h /var/log/messages
```

> `ls -lh`와 `stat`은 파일의 논리적 크기를 보여주고, `du -h`는 파일이 실제로 차지하는 디스크 블록 사용량을 보여준다. 희소 파일에서는 두 값이 크게 다를 수 있다.

`tail -f`와 `tail -F`의 차이를 보면, `tail -f`는 GNU `tail`의 기본 동작에서 열린 파일 디스크립터를 따라가며 파일 이름이 변경되어도 기존 열린 파일을 계속 추적할 수 있지만, 로그 파일이 삭제된 뒤 같은 이름으로 새로 만들어지면 새 파일을 놓칠 수 있다. `tail -F`는 `--follow=name --retry`와 유사하게 동작하며 파일 이름을 기준으로 다시 열기를 시도하므로 logrotate 등으로 파일이 교체되는 운영 로그에 적합하다.

```bash
tail -f /var/log/app.log
tail -F /var/log/app.log
```

> 모든 로그 회전 방식에서 `tail -f`가 반드시 실패하는 것은 아니다. `copytruncate` 방식에서는 동일 파일이 유지될 수 있다. 일반적인 파일 교체 방식에 대응하려면 `tail -F`가 더 안전하다.

---

## 2. 🛠️ 표준 사용 템플릿 (Configuration)

### 2-1. `cat` — 짧은 파일 전체 출력

```bash
cat /etc/passwd
cat /etc/hosts
cat /etc/group
```

여러 파일을 이어서 출력한다.

```bash
cat /etc/hostname /etc/hosts
```

줄 번호를 표시한다.

```bash
cat -n /etc/passwd       # 빈 줄을 포함하여 번호 표시
cat -b file.txt          # 내용이 있는 줄에 번호 표시
```

보이지 않는 문자를 확인한다.

```bash
cat -A file.txt
```

대표적인 출력:

```text
^I  탭 문자
^M  CR(Carriage Return)
$   줄 끝
```

> `cat`은 일반 파일뿐 아니라 표준 입력과 일부 특수 파일도 읽을 수 있다. 디렉터리를 일반 파일처럼 출력하려고 하면 오류가 발생한다.

### 2-2. `head` — 파일 앞부분 출력

기본적으로 앞 10줄을 출력한다.

```bash
head /etc/passwd
```

앞 5줄:

```bash
head -n 5 /etc/passwd
```

GNU 환경에서는 다음 축약형도 사용할 수 있다.

```bash
head -5 /etc/passwd
```

앞 100바이트:

```bash
head -c 100 file.bin
```

마지막 5줄을 제외하고 출력:

```bash
head -n -5 file.txt
```

### 2-3. `tail` — 파일 뒷부분 출력

기본적으로 뒤 10줄을 출력한다.

```bash
tail /etc/passwd
```

뒤 20줄:

```bash
tail -n 20 /var/log/messages
```

GNU 환경에서 사용할 수 있는 축약형:

```bash
tail -20 /var/log/messages
```

뒤 100바이트:

```bash
tail -c 100 file.bin
```

11번째 줄부터 파일 끝까지 출력:

```bash
tail -n +11 file.txt
```

### 2-4. 실시간 로그 추적

파일 디스크립터 기준 추적:

```bash
tail -f /var/log/messages
```

파일 이름 기준 재시도 및 추적:

```bash
tail -F /var/log/nginx/access.log
```

여러 로그 동시 추적:

```bash
tail -F \
  /var/log/nginx/access.log \
  /var/log/nginx/error.log
```

최근 100줄부터 실시간 추적:

```bash
tail -n 100 -F /var/log/app.log
```

종료:

```text
Ctrl + C
```

### 2-5. `more` — 단순 페이지 조회

```bash
more /etc/ssh/sshd_config
```

대표 조작 키:

```text
Enter   한 줄 아래
Space   다음 페이지
b       이전 페이지(입력과 구현 환경에 따라 제한될 수 있음)
q       종료
/문자열  문자열 검색
```

파이프 출력 조회:

```bash
ls -l /etc | more
```

> 파이프 입력에서는 이전 페이지 이동이 제한될 수 있다. 양방향 이동과 검색이 필요하면 `less`를 사용한다.

### 2-6. `less` — 대용량 파일 조회

```bash
less /var/log/messages
```

대표 조작 키:

```text
j / 아래 방향키    한 줄 아래
k / 위 방향키      한 줄 위
Space              다음 페이지
b                  이전 페이지
g 또는 gg          파일 처음
G                  파일 끝
/문자열             아래 방향 검색
?문자열             위 방향 검색
n                  같은 방향의 다음 검색 결과
N                  반대 방향의 검색 결과
q                  종료
```

줄 번호 표시:

```bash
less -N /etc/passwd
```

긴 줄 자동 줄 바꿈 억제:

```bash
less -S /var/log/messages
```

> `-S`는 긴 줄의 데이터를 삭제하거나 실제로 자르는 옵션이 아니다. 화면에서 자동 줄 바꿈을 하지 않으며 좌우 방향키로 가로 이동할 수 있다.

파일 끝에서 열기:

```bash
less +G /var/log/messages
```

실시간 추적 모드로 열기:

```bash
less +F /var/log/app.log
```

`less +F` 사용 중:

```text
Ctrl + C   추적을 잠시 중단하고 일반 less 모드로 전환
F          다시 추적 모드로 전환
q          종료
```

### 2-7. `nl` — 줄 번호 출력

기본적으로 내용이 있는 줄에 번호를 표시한다.

```bash
nl /etc/passwd
```

모든 줄에 번호를 표시한다.

```bash
nl -ba file.txt
```

번호 너비 지정:

```bash
nl -ba -w 4 file.txt
```

다른 명령의 결과에 줄 번호를 표시한다.

```bash
ls -l /etc | nl
```

`cat -n`과 비교:

```bash
cat -n file.txt      # 기본적으로 모든 줄에 번호
nl file.txt          # 기본적으로 비어 있지 않은 본문 줄에 번호
nl -ba file.txt      # 모든 줄에 번호
```

### 2-8. 파이프 연동

특정 문자열이 포함된 줄을 페이지 단위로 조회:

```bash
grep 'ERROR' /var/log/app.log | less
```

실시간 로그에서 특정 문자열 조회:

```bash
tail -F /var/log/app.log |
grep --line-buffered 'ERROR'
```

목록에 번호를 붙여 페이지 단위로 조회:

```bash
ls -l /etc | nl | less
```

앞부분만 조회:

```bash
ls -l /var/log | head -n 20
```

뒷부분만 조회:

```bash
ls -l /var/log | tail -n 20
```

최근 수정된 항목 조회:

```bash
ls -lt /etc | head
```

> `ls -l`의 첫 줄에 출력되는 `합계`까지 포함해 정확한 파일 개수를 제한하려면 출력 형식과 로케일을 고려해야 한다. 자동화에서는 `find`와 `sort` 사용을 검토한다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
ls -lh <파일>                    # 논리적 파일 크기
du -h <파일>                     # 디스크 블록 사용량
wc -l <파일>                     # 줄 수
file <파일>                      # 파일 형식과 개행 방식
stat <파일>                      # 크기·시간·권한 등 상세 정보
```

### 3-2. 대표 트러블슈팅

#### 🚨 시나리오 1. `cat huge.log` 실행 후 SSH 세션이 멈춘 것처럼 보인다

현재 세션에서 중단:

```text
Ctrl + C
```

다른 세션에서 `cat` 프로세스를 확인한다.

```bash
pgrep -a cat
```

해당 PID만 종료한다.

```bash
kill -TERM <PID>
```

> `pkill cat`은 다른 사용자가 실행 중인 정상적인 `cat`까지 종료할 수 있으므로 PID를 확인한 후 종료하는 것이 안전하다.

다시 조회:

```bash
less huge.log
tail -n 500 huge.log
```

#### 🚨 시나리오 2. 로그 회전 후 새 로그가 보이지 않는다

```bash
tail -F /var/log/nginx/access.log
```

파일과 inode 상태 확인:

```bash
ls -li /var/log/nginx/access.log*
lsof /var/log/nginx/access.log*
```

#### 🚨 시나리오 3. 설정 문법은 맞아 보이지만 파싱 오류가 발생한다

보이지 않는 문자 확인:

```bash
cat -A /etc/httpd/conf/httpd.conf | less
file /etc/httpd/conf/httpd.conf
```

CRLF를 LF로 변환하기 전에 백업한다.

```bash
cp -p /etc/httpd/conf/httpd.conf \
  /etc/httpd/conf/httpd.conf.bak
```

`dos2unix`가 설치되어 있다면:

```bash
dos2unix /etc/httpd/conf/httpd.conf
```

`sed`를 사용할 경우:

```bash
sed -i 's/\r$//' /etc/httpd/conf/httpd.conf
```

서비스별 문법 검사:

```bash
sshd -t
nginx -t
apachectl configtest
```

> 실제 설치된 서비스에 해당하는 검사 명령만 사용한다.

#### 🚨 시나리오 4. `more`에서 위로 이동할 수 없다

파이프로 전달된 입력은 이전 내용을 자유롭게 다시 읽지 못할 수 있다.

```bash
command | less
```

또는 출력을 임시 파일로 저장한 뒤 조회한다.

```bash
command > /tmp/output.txt
less /tmp/output.txt
```

#### 🚨 시나리오 5. 바이너리 파일을 출력해 터미널 문자가 깨졌다

파일 유형 확인:

```bash
file <파일>
```

텍스트 문자열만 조회:

```bash
strings <파일> | less
```

터미널이 깨졌다면:

```bash
reset
```

> 📌 **핵심 요약**
> - 짧은 파일 전체: `cat`
> - 앞·뒤 일부: `head`, `tail`
> - 대용량 조회: `less`
> - 실시간 로그: `tail -F`
> - 줄 번호: `nl`, `cat -n`
> - 특수문자 확인: `cat -A`
> - 관련: cat 리다이렉션 & Heredoc · 파일·디렉터리 검색 (find)

---
