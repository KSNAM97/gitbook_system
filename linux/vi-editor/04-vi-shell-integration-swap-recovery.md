# 🔗 VI Shell 연동 & Swap 파일 복구

> **Tag:** #Linux #Vi #Vim #Shell #Recovery #SwapFile  
> **핵심 요약:** VI/Vim 내부에서 `:!`로 외부 명령을 실행하고, `:read !`로 결과를 삽입하며, `:[범위]!`로 버퍼 내용을 외부 명령으로 필터링할 수 있다. Swap 경고가 발생하면 동시 편집 여부를 먼저 확인하고 복구 후에만 잔존 Swap을 삭제한다.

---

## 1. 📖 개요 (Overview)

VI/Vim 내부에서 셸 명령을 실행하는 이유는 편집 화면을 유지하면서 파일 목록을 확인하고, 다른 파일 내용을 참고하고, 문법 검사를 실행하고, 백업 파일과 현재 내용을 비교하고, 로그 및 서비스 상태를 확인하고, 외부 명령 결과를 문서에 삽입하고, 선택한 줄을 정렬·필터링할 수 있기 때문이다.

셸 연동 명령은 각각 동작이 다르다. `:!명령`은 외부 명령 실행 후 화면으로 복귀하고, `:read !명령`은 외부 명령의 출력을 버퍼에 삽입하며, `:[범위]!명령`은 범위의 내용을 명령 입력으로 보내고 그 출력으로 교체하고, `:write !명령`은 버퍼 내용을 외부 명령의 표준 입력으로 전달한다.

| 명령 | 동작 |
|---|---|
| `:!명령` | 외부 명령 실행 후 화면으로 복귀 |
| `:read !명령` | 외부 명령의 출력을 버퍼에 삽입 |
| `:[범위]!명령` | 범위의 내용을 명령 입력으로 보내고 출력으로 교체 |
| `:write !명령` | 버퍼 내용을 외부 명령의 표준 입력으로 전달 |

Swap 파일 경고는 몇 가지 원인으로 발생할 수 있다. 다른 Vim 세션이 같은 파일을 편집 중이거나, 이전 Vim 세션이 비정상 종료되었거나, 원격 호스트에서 생성된 Swap이 공유 파일시스템에 남아 있거나, 복구 후 Swap 파일이 정리되지 않은 경우다. Swap은 복구를 돕는 임시 데이터이며 완전한 백업이 아니므로, 확인하지 않고 즉시 삭제하면 저장하지 않은 편집 내용을 잃거나 동시 편집 충돌을 놓칠 수 있다.

---

## 2. 🛠️ 표준 사용 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. `:!` — 외부 명령 실행

```vim
:!ls -l /root
:!date
:!hostname
:!wc -l %
```

`%`는 현재 버퍼의 파일명을 의미한다.

```vim
:!diff /etc/passwd %
```

> **참고:** `:!diff backup %`는 디스크에 저장된 현재 파일과 백업을 비교한다. 저장하지 않은 버퍼 변경은 `%` 파일에 아직 반영되지 않았으므로 비교되지 않는다.

저장하지 않은 현재 버퍼를 비교:

```vim
:w !diff -u /tmp/file.bak -
```

명령 출력 확인 후 복귀:

```text
Press ENTER or type command to continue
```

### Step 2. `:read !` — 명령 결과 삽입

현재 줄 다음에 삽입:

```vim
:read !date
:read !ls -l /root
:read !cat /etc/hosts
```

축약형:

```vim
:r !date
```

10번째 줄 다음에 삽입:

```vim
:10read !ls -l /root
```

파일 마지막에 삽입:

```vim
:$read !ls -l /root
```

파일 첫 줄 앞에 삽입:

```vim
:0read !date
```

> **참고:** `:10!ls -l /root`는 10번째 줄을 `ls` 명령의 입력으로 전달한 후 그 출력으로 교체한다. 10번째 줄 다음에 결과를 삽입하려면 `:10read !명령`을 사용한다.

### Step 3. `:[범위]!` — 외부 필터

현재 줄을 외부 명령으로 필터링:

```vim
:.!tr 'a-z' 'A-Z'
```

1~10번째 줄 정렬:

```vim
:1,10!sort
```

파일 전체 정렬:

```vim
:%!sort
```

파일 전체 정렬 후 중복 제거:

```vim
:%!sort -u
```

공백 기준 열 정렬:

```vim
:%!column -t
```

선택한 범위의 앞 공백 제거 예시:

```vim
:1,10!sed 's/^[[:space:]]*//'
```

> **참고:** 외부 필터가 실패하거나 예상과 다른 출력을 생성하면 해당 범위가 손상될 수 있다. 먼저 일부 범위에서 테스트하고 `u`로 되돌릴 수 있는 상태를 유지한다.

### Step 4. 실습 환경 준비

Root 계정에서 `/backup` 생성:

```bash
mkdir -p /backup
```

기존 실습 파일만 정리할 경우 삭제 대상을 먼저 확인한다.

```bash
find /backup -mindepth 1 -maxdepth 1 -print
```

확인 후:

```bash
rm -rf -- /backup/*
```

> **주의:** 위 Glob은 숨김파일을 포함하지 않는다. 실습 디렉터리 전체 초기화가 필요하면 별도 문서의 안전 삭제 절차를 따른다.

파일 복사:

```bash
cp \
  /etc/NetworkManager/system-connections/ens160.nmconnection \
  /etc/passwd \
  /etc/login.defs \
  /backup/
```

검증:

```bash
ls -l /backup
```

### Step 5. VI 내부에서 실습

`passwd` 파일 복사:

```bash
cp /backup/passwd /home/guest/passwd
```

파일 열기:

```bash
vi /home/guest/passwd
```

줄 번호 표시:

```vim
:set number
```

11~35번째 줄 삭제:

```vim
:11,35delete
```

`/root` 조회:

```vim
:!ls -l /root
```

조회 결과를 파일 끝에 삽입:

```vim
:$read !ls -l /root
```

10번째 줄 다음에 삽입:

```vim
:10read !ls -l /root
```

원본 `/etc/passwd` 내용으로 버퍼 전체 교체:

```vim
:%delete
:0read /etc/passwd
```

외부 `cat`을 사용할 수도 있다.

```vim
:%delete
:0read !cat /etc/passwd
```

> **참고:** 외부 명령이 필요하지 않다면 `:read /etc/passwd`가 더 단순하다.

현재 버퍼와 원본 비교:

```vim
:w !diff -u /etc/passwd -
```

백업 파일로 저장하되 현재 편집 대상 유지:

```vim
:w /home/guest/passwd.bak
```

확인:

```vim
:!ls -l /home/guest/passwd*
```

> **참고:** 원본 조건은 `/home/guest/passwd.bak`이다. `/home/passwd.bak`로 저장하면 다른 디렉터리에 생성된다.

### Step 6. 다른 이름 저장과 파일 전환

```vim
:w /tmp/passwd.bak
:saveas /tmp/passwd.new

:e /etc/hosts
:e!

:split /etc/hosts
:vsplit /etc/passwd
```

창 이동:

```text
Ctrl+w w
```

### Step 7. 권한이 없는 파일 저장

처음부터 권장하는 방식:

```bash
sudoedit /etc/example.conf
```

이미 일반 사용자 Vim에서 편집한 경우, 현재 파일의 절대경로를 셸 안전하게 처리한다.

```vim
:execute 'write !sudo tee ' . shellescape(expand('%:p')) . ' >/dev/null'
```

명령이 성공하고 현재 버퍼 내용과 디스크 내용이 같다면 수정 표시를 해제할 수 있다.

```vim
:set nomodified
```

또는 디스크에서 다시 읽는다.

```vim
:edit!
```

> **참고:** `:edit!`는 현재 버퍼의 저장하지 않은 내용을 버린다. `tee` 저장 성공 여부와 디스크 내용을 확인한 후 사용한다.

단순한 경로에서 흔히 사용하는 축약형:

```vim
:w !sudo tee % > /dev/null
```

주의사항:

- 파일명에 공백 또는 셸 특수문자가 있으면 단순 `%` 사용이 위험할 수 있음
- `tee` 저장은 Vim의 일반적인 원자적 저장 방식과 다름
- 새 파일 생성 시 권한·소유권·SELinux 컨텍스트 확인 필요
- 가능하면 처음부터 `sudoedit` 사용

### Step 8. 문법 검사 워크플로

백업:

```vim
:w /tmp/nginx.conf.bak
```

현재 버퍼 저장:

```vim
:w
```

문법 검사:

```vim
:!nginx -t
```

저장하지 않은 현재 버퍼를 임시 파일로 검사해야 한다면 서비스별 지원 방식에 맞는 별도 절차가 필요하다.

현재 버퍼와 백업 비교:

```vim
:w !diff -u /tmp/nginx.conf.bak -
```

이상이 없으면 reload:

```vim
:!sudo systemctl reload nginx
```

로그 확인:

```vim
:!tail -n 20 /var/log/nginx/error.log
```

### Step 9. Swap 파일 확인

외부 셸:

```bash
ls -la /backup3
```

일반적인 Swap 예시:

```text
.passwd.swp
.passwd.swo
.passwd.swn
```

Vim 설정에 따른 Swap 경로 확인:

```vim
:set directory?
:set swapfile?
```

열린 프로세스 확인:

```bash
lsof /backup3/passwd
fuser -v /backup3/passwd
pgrep -af '(^|/)(vi|vim|nvim)( |$)'
```

> **참고:** `lsof`에 파일이 나오지 않는다고 무조건 편집 세션이 없다고 단정할 수는 없다. Swap 경고에 표시된 PID, 사용자, 호스트, 파일 수정 시간을 함께 확인한다.

### Step 10. Swap에서 복구

복구 가능한 파일 목록:

```bash
vim -r
```

특정 파일 복구:

```bash
vim -r /backup3/passwd
```

복구된 내용을 즉시 원본에 덮어쓰기 전에 별도 파일로 저장한다.

```vim
:w /tmp/passwd.recovered
```

원본과 비교:

```vim
:!diff -u /backup3/passwd /tmp/passwd.recovered
```

필요한 내용을 확인한 후 원본으로 반영한다.

### Step 11. Swap 삭제

다음 조건을 모두 확인한다.

- 다른 편집 세션이 없음
- 복구할 내용이 없음 또는 복구 완료
- Swap의 사용자·호스트·PID 확인
- 원본과 복구본 비교 완료

그 후 외부 셸에서 삭제:

```bash
rm -- /backup3/.passwd.swp
```

Vim의 E325 화면에서 확실한 경우 `D`를 선택할 수도 있다.

> **주의:** 다른 세션이 편집 중인 상태에서 Swap을 삭제하면 충돌 경고 기능을 잃을 수 있다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. Vim 내부 확인

```vim
:pwd
:file
:buffers
:changes
:undolist
:set modified?
:set readonly?
:set swapfile?
:set directory?
```

### 3-2. 외부 셸 확인

```bash
lsof <파일>
fuser -v <파일>
ls -la <디렉터리>
vim -r
find /home -type f -name '.*.sw?' 2>/dev/null
```

> **참고:** Swap이 `/tmp`, 홈 디렉터리의 전용 Swap 경로 등에 저장되면 원본 디렉터리만 검색해서는 찾지 못할 수 있다.

### 3-3. 대표 트러블슈팅

#### 🚨 시나리오 1. E325 Swap 경고가 발생한다

경고 예:

```text
E325: ATTENTION
Found a swap file by the name ".nginx.conf.swp"
(1) Another program may be editing the same file.
(2) An edit session for this file crashed.
```

처리 절차:

1. 경고의 사용자, 호스트, PID, 수정 시간 확인
2. `O`로 읽기 전용 열기 또는 종료
3. 프로세스 확인
4. 활성 세션이 없으면 복구
5. 복구본을 다른 이름으로 저장
6. 원본과 비교
7. 필요할 때만 Swap 삭제

```bash
lsof /etc/nginx/nginx.conf
fuser -v /etc/nginx/nginx.conf
vim -r /etc/nginx/nginx.conf
```

#### 🚨 시나리오 2. `E212: Can't open file for writing`

가능한 원인을 확인한다.

```bash
namei -l /path/to/file
findmnt -T /path/to/file
df -h /path/to/file
df -i /path/to/file
```

현재 버퍼 저장 시도:

```vim
:execute 'write !sudo tee ' . shellescape(expand('%:p')) . ' >/dev/null'
```

성공 확인 후:

```vim
:set nomodified
```

#### 🚨 시나리오 3. `:%!sort` 후 전체 내용이 재정렬되었다

즉시 실행 취소:

```vim
u
```

일부 범위에서 테스트:

```vim
:1,10!sort
```

사전 백업:

```vim
:w /tmp/file.bak
```

현재 버퍼 비교:

```vim
:w !diff -u /tmp/file.bak -
```

#### 🚨 시나리오 4. SSH 연결이 종료되었다

SSH 연결이 끊겨도 Vim 프로세스가 즉시 종료된다고 단정할 수 없다. `tmux`, `screen`, 셸의 SIGHUP 처리 방식 등에 따라 프로세스가 남아 있을 수 있다.

프로세스 확인:

```bash
pgrep -af '(^|/)(vi|vim|nvim)( |$)'
lsof /path/to/file
```

활성 세션이 없다면 복구:

```bash
vim -r /path/to/file
```

복구본 저장:

```vim
:w /tmp/file.recovered
```

#### 🚨 시나리오 5. `:10!ls -l /root`를 실행했더니 10번째 줄이 사라졌다

`:[범위]!명령`은 해당 범위를 명령 출력으로 교체한다.

실행 취소:

```vim
u
```

10번째 줄 다음에 명령 결과를 삽입하려면:

```vim
:10read !ls -l /root
```

#### 🚨 시나리오 6. `:!diff backup %`에 저장하지 않은 변경이 보이지 않는다

`%`는 현재 파일의 디스크 경로로 확장된다. 아직 `:w`하지 않은 버퍼 변경은 디스크 파일에 없다.

현재 버퍼를 직접 비교:

```vim
:w !diff -u /tmp/backup -
```

> 📌 **핵심 요약**
> - 셸 명령: `:!명령`
> - 결과 삽입: `:read !명령`
> - 범위 필터: `:[범위]!명령`
> - 현재 버퍼 전달: `:write !명령`
> - 안전한 관리자 저장: `sudoedit`
> - 복구: `vim -r 파일`
> - Swap 삭제 전 반드시 동시 편집과 복구 필요 여부 확인
> - 관련: VI 3-Mode 아키텍처 & 파일 열기 · VI 편집 명령어 (커서·삭제·복사·Ex Mode 범위 조작) · VI 치환
