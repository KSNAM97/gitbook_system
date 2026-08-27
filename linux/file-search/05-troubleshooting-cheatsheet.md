# 파일 조회·처리·검색 트러블슈팅 치트시트

파일 조회, 리다이렉션, Heredoc, `find` 사용 중 발생하는 문제를 증상·원인·조치 순서로 빠르게 확인한다.

## 목차

1. [개요 (Overview)](#개요-overview)
2. [증상별 즉시 대응표 (Configuration)](#증상별-즉시-대응표-configuration)
3. [트러블슈팅 시나리오 (Verification & Troubleshooting)](#트러블슈팅-시나리오-verification--troubleshooting)
4. [요약](#요약)

---

## 개요 (Overview)

이 영역에서 가장 주의해야 할 동작은 다음과 같다: 1) 대용량 파일을 터미널에 전체 출력, 2) `>`로 기존 설정 파일 덮어쓰기, 3) `cat file > file`로 자기 자신 덮어쓰기, 4) Heredoc에서 의도하지 않은 변수 치환, 5) 검증하지 않은 `find -delete`, 6) 잘못된 검색 시작 경로, 7) Root 권한 리다이렉션 오해.

`sudo echo ... >> 파일`이 실패하는 이유는 파이프와 리다이렉션을 현재 셸이 구성하기 때문이다.

```bash
sudo echo 'value' >> /etc/example.conf
```

위 명령에서 `echo`만 Root 권한이고 `>>`는 일반 사용자 셸이 처리한다.

수정:

```bash
printf '%s\n' 'value' |
sudo tee -a /etc/example.conf > /dev/null
```

---

## 증상별 즉시 대응표 (Configuration)

### 2-1. 파일 조회

| 증상 | 원인 | 조치 |
|---|---|---|
| `cat` 후 화면 출력 폭주 | 대용량 파일 전체 출력 | `Ctrl+C`, `less`, `tail -n` |
| 로그 회전 후 새 내용 없음 | 기존 파일 추적 | `tail -F` |
| 설정이 정상처럼 보이지만 파싱 실패 | CRLF, BOM, 탭 등 | `file`, `cat -A`, `xxd` |
| 바이너리 출력 후 터미널 깨짐 | 바이너리 제어문자 출력 | `reset`, `file`, `strings` |
| 파일 크기와 디스크 사용량 불일치 | 희소 파일·블록 할당 | `ls -lh`, `du -h` 비교 |

### 2-2. 리다이렉션과 Heredoc

| 증상 | 원인 | 조치 |
|---|---|---|
| 설정 파일 내용이 사라짐 | `>` 사용 | 백업 복구, 이후 정확한 편집 |
| `cat file > file` 후 빈 파일 | 셸이 목적지를 먼저 비움 | `cp` 또는 임시 파일 |
| Heredoc `$변수`가 변경됨 | `<<EOF` 사용 | `<<'EOF'` |
| `sudo echo >>` 권한 거부 | 일반 셸이 `>>` 처리 | `sudo tee -a` |
| `noclobber`인데 덮어쓰기 됨 | `>|` 또는 다른 도구 사용 | 명령 기록 확인 |
| Heredoc가 종료되지 않음 | 종료 구분자 불일치 | 구분자를 단독 행에 작성 |

### 2-3. `find`

| 증상 | 원인 | 조치 |
|---|---|---|
| `-name *.log` 결과 이상 | 현재 셸이 Glob 확장 | `-name '*.log'` |
| `Permission denied` 폭주 | 접근 불가 영역 | `2>/dev/null`, `-prune` |
| 다른 파일시스템까지 검색 | 마운트 경계 통과 | `-xdev` |
| 시작 디렉터리도 결과에 포함 | 깊이 제한 없음 | `-mindepth 1` |
| 오래된 파일 조건이 예상과 다름 | `-mtime`의 24시간 반올림 | `-newermt` |
| SUID 검색 결과가 적음 | `-perm 4000` 사용 | `-perm -4000` |
| 소유자 없는 파일 검색 이상 | OR 우선순위 | 괄호 사용 |
| `-delete` 과다 삭제 | 범위·조건 미검증 | `-print` 리허설 |
| `xargs`가 입력 없이 실행 | 빈 입력 처리 | GNU `xargs -r` |

### 2-4. 핵심 진단 명령어

```bash
ls -lh <파일>
du -h <파일>
wc -l <파일>
file <파일>
stat <파일>
cat -A <파일> | less
xxd -l 64 <파일>
lsof <파일>

find <경로> <조건> -print
find <경로> <조건> -print | wc -l
find <경로> <조건> -ls
```

---

## 트러블슈팅 시나리오 (Verification & Troubleshooting)

### 🚨 시나리오 1. 대용량 파일 출력 후 SSH 화면이 멈췄다

현재 세션에서:

```text
Ctrl + C
```

다른 세션에서 프로세스 확인:

```bash
pgrep -a cat
```

해당 프로세스만 종료:

```bash
kill -TERM <PID>
```

안전하게 재조회:

```bash
less huge.log
tail -n 500 huge.log
```

> `pkill cat`은 다른 사용자의 정상 프로세스까지 종료할 수 있으므로 가능한 한 PID를 확인한다.

### 🚨 시나리오 2. 로그 회전 후 `tail -f`에 새 내용이 표시되지 않는다

```bash
tail -F /var/log/nginx/access.log
```

파일과 열린 디스크립터 확인:

```bash
ls -li /var/log/nginx/access.log*
lsof /var/log/nginx/access.log*
```

### 🚨 시나리오 3. `>`로 설정 파일을 덮어썼다

현재 파일을 추가로 변경하지 않는다.

```bash
cp -a /etc/example.conf \
  "/root/example.conf.damaged.$(date +%F-%H%M%S)"
```

복구 자료 확인:

```bash
find /etc -type f \
  \( -name '*.rpmnew' -o -name '*.rpmsave' -o -name '*.bak' \) \
  -print
```

패키지 소속 확인:

```bash
rpm -qf /etc/example.conf
```

패키지 검증:

```bash
rpm -V <패키지명>
```

복구 후 서비스별 문법 검사를 실행한다.

```bash
sshd -t
nginx -t
apachectl configtest
```

> 패키지 재설치가 수정된 설정 파일을 자동으로 기본값으로 되돌린다고 보장할 수 없다. 백업, `.rpmnew`, `.rpmsave`, 패키지 원본을 비교해야 한다.

### 🚨 시나리오 4. `cat file > file` 후 파일이 비었다

원인:

```bash
cat file > file
```

셸이 `cat` 실행 전에 목적지 파일을 비웠다.

복사는 다음 명령을 사용한다.

```bash
cp file file.copy
```

필터 결과로 교체할 때:

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

### 🚨 시나리오 5. Heredoc의 `$uri`가 치환되었다

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

### 🚨 시나리오 6. `sudo echo >>`가 권한 오류로 실패한다

```bash
printf '%s\n' 'net.ipv4.ip_forward=1' |
sudo tee -a /etc/sysctl.conf > /dev/null
```

또는:

```bash
sudo sh -c \
  'printf "%s\n" "net.ipv4.ip_forward=1" >> /etc/sysctl.conf'
```

### 🚨 시나리오 7. Heredoc가 끝나지 않는다

종료 구분자는 다음 조건을 만족해야 한다.

- 시작할 때 지정한 문자열과 정확히 일치
- 일반 `<<EOF`에서는 줄 앞에 공백이 없어야 함
- 종료 구분자 뒤에 다른 문자가 없어야 함
- 대소문자가 일치해야 함

정상 예:

```bash
cat > /tmp/file <<'EOF'
hello
EOF
```

강제 중단:

```text
Ctrl + C
```

### 🚨 시나리오 8. `find /home -name *.log` 결과가 이상하다

```bash
find /home -name '*.log'
```

현재 셸의 Glob 결과 확인:

```bash
printf '<%s>\n' *.log
```

### 🚨 시나리오 9. `find`의 `Permission denied`가 너무 많다

간단히 표준 오류를 숨긴다.

```bash
find / -name 'passwd' 2>/dev/null
```

가상 파일시스템을 제외한다.

```bash
find / \
  \( -path /proc -o -path /sys -o -path /dev -o -path /run \) \
  -prune \
  -o -name 'passwd' -print
```

현재 파일시스템만 검색:

```bash
find / -xdev -name 'passwd' 2>/dev/null
```

### 🚨 시나리오 10. `find -delete` 대상이 너무 많다

즉시 삭제를 실행하지 않고 검색 조건부터 다시 확인한다.

```bash
TARGET=/var/log/myapp

: "${TARGET:?TARGET must be set}"

REAL_TARGET=$(realpath -e -- "$TARGET") || exit 1

case "$REAL_TARGET" in
  /var/log/myapp|/var/log/myapp/*)
    ;;
  *)
    echo "허용되지 않은 경로: $REAL_TARGET" >&2
    exit 1
    ;;
esac
```

대상 확인:

```bash
find "$REAL_TARGET" \
  -type f -name '*.log' -mtime +30 -print
```

개수 확인:

```bash
find "$REAL_TARGET" \
  -type f -name '*.log' -mtime +30 -print |
wc -l
```

용량 확인:

```bash
find "$REAL_TARGET" \
  -type f -name '*.log' -mtime +30 \
  -exec du -ch -- {} + |
tail -n 1
```

검증 후에만 삭제한다.

```bash
find "$REAL_TARGET" \
  -type f -name '*.log' -mtime +30 -delete
```

### 🚨 시나리오 11. `-mtime +7`이 생각한 달력 날짜와 다르다

`-mtime`은 현재 시각 기준 완료된 24시간 단위로 계산한다.

달력 날짜 기준으로 검색:

```bash
find /var/log -type f \
  ! -newermt '2026-07-01 00:00:00'
```

특정 기간:

```bash
find /var/log -type f \
  -newermt '2026-07-01 00:00:00' \
  ! -newermt '2026-07-08 00:00:00'
```

---

## 요약

- 대용량 파일: `less`, `tail -n`
- 로그 회전 대응: `tail -F`
- 덮어쓰기 전 백업
- 원문 Heredoc: `<<'EOF'`
- Root 파일: `sudo tee`
- `find -name` 패턴은 따옴표
- 삭제 전 `-print`, 개수, 용량 확인
- 관련: **파일 내용 출력 6종 (cat · head · tail · more · less · nl)** · **cat 리다이렉션 & Heredoc** · **파일·디렉터리 검색 (find)**
