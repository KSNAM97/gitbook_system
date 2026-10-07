# 파일 조회·처리·검색 명령어 퀵 레퍼런스

`cat`, `head`, `tail`, `more`, `less`, `nl`, 리다이렉션, Heredoc, `find` 문법을 빠르게 확인하는 조회용 문서.

## 목차

1. [명령어 문법 (Concept)](#명령어-문법-concept)
2. [빠른 조회표 (Configuration)](#빠른-조회표-configuration)
3. [검증 명령어 모음 (Verification & Troubleshooting)](#검증-명령어-모음-verification--troubleshooting)
4. [요약](#요약)

---

## 명령어 문법 (Concept)

### 1-1. 파일 내용 조회

```bash
cat file                       # 파일 전체 내용 출력
cat -n file                    # 모든 줄에 번호를 붙여 출력
cat -b file                    # 내용이 있는 줄에만 번호 출력
cat -A file                    # 탭·CR·줄 끝 등 특수문자 표시

head file                      # 파일 앞 10줄 출력
head -n 5 file                 # 파일 앞 5줄 출력
head -c 100 file                # 파일 앞 100바이트 출력

tail file                      # 파일 뒤 10줄 출력
tail -n 20 file                # 파일 뒤 20줄 출력
tail -c 100 file                # 파일 뒤 100바이트 출력
tail -f file                   # 열린 파일 디스크립터를 계속 추적
tail -F file                   # 파일명 기준으로 재시도하며 추적

nl file                        # 내용이 있는 줄에 기본 줄 번호 표시
nl -ba file                    # 빈 줄을 포함한 모든 줄에 번호 표시
```

### 1-2. 페이저

```bash
more file                      # 단순 페이지 단위 조회

less file                      # 앞뒤 이동 가능한 페이지 조회
less -N file                   # 줄 번호 표시
less -S file                   # 긴 줄 자동 줄 바꿈 억제
less +G file                   # 파일 끝에서 조회 시작
less +F file                   # 실시간 추적 모드로 시작
```

`less` 조작:

```text
j / k       한 줄 아래·위
Space / b   다음·이전 페이지
g / G       파일 처음·끝
/문자열      아래 방향 검색
?문자열      위 방향 검색
n / N       다음·반대 방향 검색 결과
F           실시간 추적
q           종료
```

### 1-3. 리다이렉션

```bash
command > file                 # 표준 출력을 파일에 덮어쓰기
command >> file                # 표준 출력을 파일 끝에 추가
command < file                 # 파일을 표준 입력으로 사용
command 2> error.log           # 표준 오류만 파일에 저장
command > all.log 2>&1         # 표준 출력과 오류를 같은 파일에 저장
command &> all.log             # Bash 형식으로 출력·오류 함께 저장

> file                         # 파일 생성 또는 내용 비우기
: > file                       # no-op 명령으로 파일 생성 또는 비우기

command | tee file             # 화면 출력과 파일 덮어쓰기를 함께 수행
command | tee -a file          # 화면 출력과 파일 추가를 함께 수행
```

### 1-4. Heredoc

변수·명령 치환 허용:

```bash
cat > file <<EOF               # EOF까지의 내용을 file에 덮어쓰기
User: $USER                    # 현재 셸에서 변수 치환
Host: $(hostname)              # 현재 셸에서 명령 치환
EOF
```

치환 억제:

```bash
cat > file <<'EOF'             # 구분자를 인용해 변수·명령 치환 억제
$USER
$(hostname)
EOF
```

기존 파일에 추가:

```bash
cat >> file <<'EOF'            # Heredoc 내용을 파일 끝에 추가
new line
EOF
```

선행 탭 제거:

```bash
cat > file <<-'EOF'            # 각 줄의 선행 탭을 제거하며 기록
	line1
	line2
EOF
```

### 1-5. Root 파일 기록

추가:

```bash
printf '%s\n' 'value' |
sudo tee -a /etc/example.conf > /dev/null
# root 권한으로 파일 끝에 value 추가
```

전체 생성:

```bash
sudo tee /etc/example.conf > /dev/null <<'EOF'
option=value
EOF
# root 권한으로 파일을 새 내용으로 덮어쓰기
```

### 1-6. `find` 기본 검색

```bash
find /etc -name 'passwd'       # 이름이 정확히 passwd인 항목 검색
find / -iname '*.conf'         # 대소문자를 무시하고 .conf 검색
find /etc -type f              # 일반 파일 검색
find /etc -type d              # 디렉터리 검색
find / -type l                 # 심볼릭 링크 자체 검색
find / -xtype l                # 대상이 존재하지 않는 심볼릭 링크 검색
```

### 1-7. `find` 시간 검색

```bash
find /etc -newer /etc/aliases           # aliases보다 수정 시간이 최신인 항목
find /backup -newermt '2026-07-05'      # 지정 날짜 이후 수정된 항목

find /var/log -mtime -1                 # 완료된 24시간 단위가 1보다 작은 항목
find /var/log -mtime +7                 # 완료된 24시간 단위가 7보다 큰 항목
find /tmp -mmin -30                     # 완료된 분 단위가 30보다 작은 항목
find /home -atime +30                   # 접근 후 완료 일수가 30보다 큰 항목
```

### 1-8. `find` 크기·권한·소유자

```bash
find /var/log -type f -size +100M       # 100MiB보다 큰 일반 파일 검색
find /var/log -type f -size +1G         # 1GiB보다 큰 일반 파일 검색
find /home -type f -empty               # 크기가 0인 일반 파일 검색
find /home -type d -empty               # 내부 항목이 없는 디렉터리 검색

find / -type f -perm -4000 2>/dev/null  # SUID가 설정된 파일 검색
find / -type f -perm -2000 2>/dev/null  # SGID가 설정된 파일 검색
find / -type f -perm -0002 2>/dev/null  # Other 쓰기 권한이 있는 파일 검색

find / -user nobody 2>/dev/null         # nobody 소유 항목 검색
find /home -nouser 2>/dev/null          # 유효한 소유자가 없는 항목 검색
find /home -nogroup 2>/dev/null         # 유효한 그룹이 없는 항목 검색
```

### 1-9. `find` 조건 조합

AND:

```bash
find /etc -type f -name '*.conf'        # 일반 파일이면서 .conf인 항목 검색
```

OR:

```bash
find /etc -type f \
  \( -name '*.conf' -o -name '*.cfg' \)
# 일반 파일 중 .conf 또는 .cfg 검색
```

NOT:

```bash
find /var/log -type f ! -name '*.gz'    # .gz가 아닌 일반 파일 검색
```

소유자 또는 그룹이 없는 항목:

```bash
find /home \( -nouser -o -nogroup \) -print # 소유자 또는 그룹 없는 항목 출력
```

### 1-10. `find` 액션

```bash
find /etc -type f -name '*.conf' -print # 검색 결과 경로 출력
find /etc -type f -name '*.conf' -ls    # 검색 결과 상세 정보 출력

find /tmp -type f -name '*.tmp' \
  -exec ls -l -- {} \;
# 결과마다 ls 명령을 한 번씩 실행

find /tmp -type f -name '*.tmp' \
  -exec ls -l -- {} +
# 여러 결과를 묶어 ls 명령 실행

find /tmp -type f -name '*.tmp' -print0 |
xargs -0 -r ls -l --
# 특수문자와 공백이 있는 파일명을 안전하게 전달
```

삭제 전:

```bash
find /tmp/myapp -type f -mtime +7 -print # 삭제 예정 파일 목록 확인
```

확인 후:

```bash
find /tmp/myapp -type f -mtime +7 -delete # 조건에 맞는 파일 삭제
```

### 1-11. 범위 제한

깊이 제한:

```bash
find /etc -mindepth 1 -maxdepth 2        # 시작점 제외, 최대 2단계까지 검색
```

파일시스템 경계 유지:

```bash
find / -xdev -name '*.conf' 2>/dev/null  # 다른 파일시스템으로 넘어가지 않음
```

특정 경로 제외:

```bash
find / \
  \( -path /proc -o -path /sys -o -path /dev \) -prune \
  -o -name '*.conf' -print
# /proc·/sys·/dev를 제외하고 .conf 검색
```

---

## 빠른 조회표 (Configuration)

### 1. 파일 조회 도구

| 상황 | 명령어 |
|---|---|
| 짧은 파일 전체 | `cat` |
| 앞부분 | `head -n` |
| 뒷부분 | `tail -n` |
| 대용량 파일 | `less` |
| 실시간 로그 | `tail -F` |
| 줄 번호 | `nl`, `cat -n` |
| 특수문자 | `cat -A` |

### 2. `tail -f`와 `tail -F`

| 옵션 | 기본 추적 방식 | 파일 교체 대응 |
|---|---|---|
| `-f` | 열린 파일 디스크립터 | 제한될 수 있음 |
| `-F` | 파일 이름+재시도 | 로그 로테이션에 적합 |

### 3. 리다이렉션

| 기호 | 의미 |
|---|---|
| `>` | 생성·덮어쓰기 |
| `>>` | 추가 |
| `<` | 표준 입력 |
| `2>` | 표준 오류 |
| `2>&1` | 표준 오류를 현재 표준 출력 위치로 |
| `&>` | Bash 표준 출력+오류 |
| `tee` | 화면 출력과 파일 저장 |
| `tee -a` | 화면 출력과 파일 추가 |

### 4. Heredoc

| 형식 | 치환 | 설명 |
|---|---:|---|
| `<<EOF` | O | 변수·명령 치환 |
| `<<'EOF'` | X | 원문 저장 |
| `<<-EOF` | O | 선행 탭 제거 |
| `<<-'EOF'` | X | 치환 억제+선행 탭 제거 |

### 5. `find` 파일 종류

| 조건 | 종류 |
|---|---|
| `-type f` | 일반 파일 |
| `-type d` | 디렉터리 |
| `-type l` | 심볼릭 링크 |
| `-type b` | 블록 장치 |
| `-type c` | 문자 장치 |
| `-type s` | 소켓 |
| `-type p` | 파이프 |

### 6. `find` 시간 조건

| 조건 | 의미 |
|---|---|
| `-mtime -1` | 완료된 24시간 단위가 1보다 작음 |
| `-mtime +7` | 완료된 24시간 단위가 7보다 큼 |
| `-mmin -30` | 완료된 분 단위가 30보다 작음 |
| `-atime +30` | 접근 시간 기준 완료 일수가 30보다 큼 |
| `-newer file` | 기준 파일보다 수정 시간이 최신 |
| `-newermt '날짜'` | 지정 날짜보다 수정 시간이 최신 |

### 7. `find` 권한 조건

| 조건 | 의미 |
|---|---|
| `-perm 4000` | 권한 모드가 정확히 4000 |
| `-perm -4000` | SUID 비트가 모두 설정 |
| `-perm -2000` | SGID 비트가 모두 설정 |
| `-perm /6000` | SUID 또는 SGID 하나 이상 설정 |
| `-perm -0002` | Other 쓰기 권한 설정 |

---

## 검증 명령어 모음 (Verification & Troubleshooting)

파일 조회 전:

```bash
ls -lh <파일>                   # 파일 크기와 기본 정보 확인
du -h <파일>                    # 실제 할당된 디스크 사용량 확인
wc -l <파일>                    # 파일 줄 수 확인
file <파일>                     # 실제 파일 형식 확인
stat <파일>                     # inode·시간·권한 등 상세 확인
```

내용 검증:

```bash
tail -n 10 <파일>               # 마지막 10줄 확인
diff -- <원본> <사본>           # 두 파일의 내용 차이 확인
sha256sum <원본> <사본>         # 두 파일의 SHA-256 체크섬 계산
```

검색 검증:

```bash
find <경로> <조건> -print       # 검색 결과 경로 확인
find <경로> <조건> -print | wc -l # 검색 결과 개수 확인
find <경로> <조건> -ls          # 검색 결과 상세 정보 확인
```

예상 삭제 용량:

```bash
find <경로> <조건> -exec du -ch -- {} + |
tail -n 1                       # 검색된 항목의 합계 용량 확인
```

---

## 요약

- 대용량 조회: `less`
- 실시간 로그: `tail -F`
- 내용 추가: `>>`
- 원문 Heredoc: `<<'EOF'`
- Root 파일 기록: `sudo tee`
- `find` 이름 패턴: 따옴표 사용
- SUID 검색: `-perm -4000`
- 삭제: `-print` 검증 후 `-delete`
- 관련: **파일 내용 출력 6종 (cat · head · tail · more · less · nl)** · **cat 리다이렉션 & Heredoc** · **파일·디렉터리 검색 (find)**
