# 파일 조회·처리·검색 통합 정리

파일 작업은 조회, 기록, 검색의 세 영역으로 구분한다. 짧은 파일은 `cat`, 대용량 파일은 `less`, 실시간 로그는 `tail -F`, 기존 파일 추가는 `>>`, 정밀 검색은 `find`를 사용한다.

## 목차

1. [개요 (Overview)](#개요-overview)
2. [표준 개념 정리 (Configuration)](#표준-개념-정리-configuration)
3. [검증 및 트러블슈팅 (Verification & Troubleshooting)](#검증-및-트러블슈팅-verification--troubleshooting)
4. [요약](#요약)

---

## 개요 (Overview)

파일 조회·기록·검색의 기본 흐름은 다음과 같다.

```text
파일 형식·크기 확인
        ↓
목적에 맞는 조회 명령 선택
        ↓
변경 전 백업
        ↓
리다이렉션 또는 편집 도구로 기록
        ↓
내용·문법 검증
        ↓
find로 검색 및 관리
```

파괴적 동작을 막는 공통 원칙은 다음과 같다: 대용량 파일을 무조건 `cat`으로 출력하지 않고, `>`를 사용하기 전에 기존 파일 존재 여부를 확인하며, 설정 파일 변경 전 백업하고, Heredoc의 변수 치환 여부를 확인하며, `find -delete` 전에 `-print`로 검증하고, 검색 시작 경로와 파일 종류를 제한하며, 변수에 저장된 경로를 정규화하고 허용 범위를 검사한다.

`tail -f`와 `tail -F`의 선택 기준은, 단순히 열린 파일을 계속 읽을 때는 `tail -f`를 사용하고, logrotate 등으로 파일이 교체될 수 있을 때는 `tail -F`를 사용하는 것이다.

```bash
tail -F /var/log/app.log
```

---

## 표준 개념 정리 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 조회 명령어 선택

| 상황 | 명령어 | 설명 |
|---|---|---|
| 짧은 파일 전체 | `cat` | 전체 출력 |
| 앞부분 | `head -n` | 앞에서 지정한 줄 |
| 뒷부분 | `tail -n` | 뒤에서 지정한 줄 |
| 대용량 파일 | `less` | 페이지 조회 및 검색 |
| 실시간 로그 | `tail -F` | 파일 교체 대응 |
| 줄 번호 | `nl`, `cat -n` | 줄 번호 출력 |
| 특수문자 확인 | `cat -A` | 탭·CR·줄 끝 표시 |

조회 전 확인:

```bash
ls -lh <파일>
stat <파일>
file <파일>
wc -l <파일>
```

### Step 2. 조회 명령어

```bash
cat /etc/passwd
cat -n /etc/passwd
cat -A file.txt

head -n 5 /etc/passwd
tail -n 20 /var/log/messages
tail -F /var/log/app.log

less -N /var/log/messages
less -S /var/log/messages

nl file.txt
nl -ba file.txt
```

### Step 3. 리다이렉션

```bash
command > file                 # 생성 또는 덮어쓰기
command >> file                # 기존 파일 끝에 추가
command 2> error.log           # 표준 오류
command > all.log 2>&1         # 표준 출력과 오류
command &> all.log             # Bash 축약형
```

파일 내용 비우기:

```bash
: > file
```

`noclobber` 활성화:

```bash
set -o noclobber
```

강제 덮어쓰기:

```bash
command >| file
```

### Step 4. Heredoc

변수와 명령 치환 허용:

```bash
cat > /tmp/system-info <<EOF
Host: $(hostname)
User: $USER
EOF
```

치환 억제:

```bash
cat > /tmp/app.conf <<'EOF'
location / {
    try_files $uri $uri/ =404;
}
EOF
```

기존 파일 끝에 추가:

```bash
cat >> /tmp/app.conf <<'EOF'
option=value
EOF
```

### Step 5. Root 권한 파일 기록

잘못된 예:

```bash
sudo echo 'value' >> /etc/example.conf
```

권장:

```bash
printf '%s\n' 'value' |
sudo tee -a /etc/example.conf > /dev/null
```

파일 전체 생성:

```bash
sudo tee /etc/example.conf > /dev/null <<'EOF'
option1=value1
option2=value2
EOF
```

### Step 6. `find` 기본 골격

```bash
find [시작 경로] [조건] [액션]
```

```bash
find /etc -type f -name '*.conf' -print
find /var/log -type f -mtime +7 -print
find /tmp -type f -mmin -30 -print
find /var/log -type f -size +100M -print
find / -type f -perm -4000 2>/dev/null
find /home \( -nouser -o -nogroup \) -print
```

### Step 7. `find` 액션

```bash
find /etc -type f -name '*.conf' -print
find /etc -type f -name '*.conf' -ls

find /tmp -type f -name '*.tmp' \
  -exec ls -l -- {} +

find /tmp -type f -name '*.tmp' -print0 |
xargs -0 -r ls -l --
```

삭제 전:

```bash
find /tmp/myapp -type f -mtime +7 -print
```

확인 후:

```bash
find /tmp/myapp -type f -mtime +7 -delete
```

### Step 8. 표준 안전 패턴

#### 대용량 파일 조회

```bash
ls -lh /var/log/app.log
less /var/log/app.log
```

#### 설정 변경

```bash
cp -a /etc/example.conf \
  "/etc/example.conf.bak.$(date +%F-%H%M%S)"
```

변경 후 문법 검증:

```bash
<서비스별 문법 검사 명령>
```

#### 검색 후 삭제

```bash
TARGET=/tmp/myapp

find "$TARGET" -type f -name '*.tmp' -mtime +7 -print
find "$TARGET" -type f -name '*.tmp' -mtime +7 -delete
```

---

## 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 공통 검증 명령어

```bash
ls -lh <파일>                    # 논리적 크기
du -h <파일>                     # 디스크 사용량
wc -l <파일>                     # 줄 수
file <파일>                      # 파일 형식
stat <파일>                      # 메타데이터
tail -n 10 <파일>                # 마지막 내용
diff -- <원본> <사본>            # 내용 비교
sha256sum <원본> <사본>          # 체크섬
```

검색 결과 검증:

```bash
find <경로> <조건> -print
find <경로> <조건> -print | wc -l
find <경로> <조건> -ls
```

### 3-2. 대표 함정

| 함정 | 문제 | 올바른 접근 |
|---|---|---|
| `cat huge.log` | 터미널 출력 폭주 | `less`, `tail -n` |
| `tail -f` | 파일 교체 시 새 로그 누락 가능 | `tail -F` |
| `echo ... > config` | 기존 설정 제거 | 백업 후 정확한 변경 |
| `cat file > file` | 원본 파일이 비워짐 | `cp` 또는 임시 파일 |
| `<<EOF` | 변수·명령 치환 | 원문은 `<<'EOF'` |
| `sudo echo >> file` | 셸 권한으로 리다이렉션 | `sudo tee -a` |
| `find -name *.log` | 셸 Glob 선확장 | `-name '*.log'` |
| `find / -delete` | 전체 시스템 삭제 위험 | 제한된 경로 + `-print` |
| `-perm 4000` | 정확히 4000만 검색 | SUID 포함은 `-perm -4000` |
| `-nouser -o -nogroup` | 우선순위 혼동 | 괄호 사용 |

### 3-3. 장애 대응 흐름

```text
1. 작업 중단
2. 대상 파일·경로 확인
3. 파일 크기·형식·권한 확인
4. 명령 기록과 로그 확인
5. 백업·스냅샷 확인
6. 안전한 방식으로 복구
7. 서비스 문법 검사
8. 서비스 reload 또는 restart
9. 재발 방지 절차 문서화
```

---

## 요약

- 짧은 파일: `cat`
- 대용량 파일: `less`
- 실시간 로그: `tail -F`
- 추가 기록: `>>`
- 원문 Heredoc: `<<'EOF'`
- Root 파일 기록: `sudo tee`
- 검색: `find`
- 삭제: `-print` 검증 후 `-delete`
- 관련: **파일 내용 출력 6종 (cat · head · tail · more · less · nl)** · **cat 리다이렉션 & Heredoc** · **파일·디렉터리 검색 (find)**
