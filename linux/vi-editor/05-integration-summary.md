# 🧩 VI 통합 정리

> **Tag:** #Linux #Vi #Vim #Editor #Summary  
> **핵심 요약:** VI/Vim은 명령·입력·명령행 모드로 동작한다. 기본 편집은 카운트·오퍼레이터·모션 조합으로 수행하며, 대량 치환 전에는 백업과 매칭 검증이 필요하다. Swap 파일은 비정상 종료 복구를 돕지만 일반 백업을 대신하지 않는다.

---

## 1. 📖 개요 (Overview)

VI/Vim을 관통하는 핵심 설계는 같은 키가 현재 모드에 따라 다르게 동작하는 모달 편집 구조이다.

```text
명령 모드 ── i/a/o 등 ──▶ 입력 모드
명령 모드 ◀──── Esc ───── 입력 모드

명령 모드 ───── : ──────▶ 명령행 모드
명령 모드 ◀─ Esc/명령완료 ─ 명령행 모드
```

실행 직후에는 명령 모드이고, 실제 문자 입력은 입력 모드에서 이루어지며, 저장·종료·치환·범위 조작은 명령행 모드에서 처리한다. 모드가 혼동되면 `Esc`를 눌러 명령 모드로 복귀한다.

편집 명령의 기본 구조는 `[count] + [operator] + [motion]` 이다.

```text
[count] + [operator] + [motion]
```

예시:

```vim
dw          " w 이동 범위 삭제
3dw         " w 이동을 3번 적용한 범위 삭제
y$          " 현재 위치부터 줄 끝까지 복사
dG          " 현재 위치부터 파일 끝까지 삭제
5dd         " 현재 줄 포함 5줄 삭제
```

운영 설정 파일을 수정할 때는 다음 원칙을 지켜야 한다. 먼저 현재 파일과 권한을 확인하고, 변경 전 백업을 하며, 줄 번호를 표시하고, 좁은 범위부터 변경하며, 대량 치환은 `c` 플래그를 사용하고, 저장 전 `diff`로 확인하며, 서비스 문법 검사를 거치고, 이상이 없을 때만 reload하며, 백업·버전 관리·스냅샷을 유지한다.

Swap 파일은 비정상 종료 시 편집 내용 복구를 지원하고, 동일 파일의 동시 편집을 경고하며, 편집 중 임시 복구 데이터를 저장하고, 정상 종료 시 일반적으로 자동 삭제된다. 다만 Swap 파일은 정식 백업, 버전 관리, 스냅샷, Persistent Undo, 설정 변경 이력을 대신하지 않는다.

---

## 2. 🛠️ 표준 개념 정리 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 모드 전환

```text
명령 모드 → 입력 모드
i  현재 커서 앞
a  현재 커서 뒤
I  현재 줄 첫 비공백 문자 앞
A  현재 줄 끝
o  현재 줄 아래 새 줄
O  현재 줄 위 새 줄

입력 모드 → 명령 모드
Esc

명령 모드 → 명령행 모드
:
```

### Step 2. 파일 열기

```bash
vi file
vi +10 file
vi + file
vi +/pattern file
vi -R file
vi file1 file2 file3
```

파일이 없더라도 저장하기 전까지 실제 파일이 생성된 것은 아니다.

### Step 3. 저장과 종료

```vim
:w                      " 저장
:w /tmp/file.bak        " 다른 파일에도 저장
:q                      " 종료
:wq                     " 저장 후 종료
:x                      " 변경된 경우 저장 후 종료
:q!                     " 저장하지 않은 변경 버리기
```

명령 모드:

```vim
ZZ                      " :x와 유사
ZQ                      " :q!와 유사
```

> **주의:** `:q!`는 현재 버퍼의 저장하지 않은 변경만 버린다. 이미 `:w`로 디스크에 저장한 내용은 복구하지 않는다.

### Step 4. 이동

```vim
h j k l                 " 왼쪽·아래·위·오른쪽
w e b                   " 다음 단어·단어 끝·이전 단어
W E B                   " 공백 기준 WORD 이동
0 ^ $                   " 줄 처음·첫 비공백·줄 끝
gg G 25G                " 파일 처음·끝·25번째 줄
Ctrl+f Ctrl+b           " 한 화면 아래·위
Ctrl+d Ctrl+u           " 반 화면 아래·위
```

### Step 5. 편집

```vim
x X                     " 현재 문자·왼쪽 문자 삭제
dw de db                " 단어 범위 삭제
dd 5dd                  " 현재 줄·현재 줄 포함 5줄 삭제
d$ D                    " 줄 끝까지 삭제
dG dgg                  " 파일 끝·처음 방향 삭제

yy 5yy                  " 현재 줄·5줄 복사
yw ye y$                " 단어·단어 끝·줄 끝까지 복사
p P                     " 뒤·앞에 붙여넣기

rX R                    " 문자 하나 교체·Replace 모드
cw ce cc C              " 범위 변경
u Ctrl+r .              " 실행 취소·재실행·반복
```

### Step 6. Ex 범위 조작

```vim
:3                      " 3번째 줄 이동
:$                      " 마지막 줄 이동

:3,10d                  " 3~10번째 줄 삭제
:.,+3d                  " 현재 줄 포함 총 4줄 삭제
:-3,.d                  " 위 3줄부터 현재 줄까지 삭제
:.,$d                   " 현재 줄부터 마지막까지 삭제
:%d                     " 전체 삭제

:3,10y                  " 3~10번째 줄 복사
:.,$y                   " 현재 줄부터 마지막까지 복사
:%y                     " 전체 복사

:3,10move 20            " 3~10번 줄을 20번 줄 뒤로 이동
:3,10copy 20            " 3~10번 줄을 20번 줄 뒤에 복사
```

### Step 7. 치환

```vim
:s/old/new/             " 현재 줄 첫 번째 일치
:s/old/new/g            " 현재 줄 모든 일치
:6s/old/new/g            " 6번째 줄 모든 일치
:2,7s/old/new/g          " 2~7번째 줄 모든 일치
:%s/old/new/g            " 파일 전체 모든 일치
:%s/old/new/gc           " 각 항목 확인
:%s/old/new/gi           " 대소문자 무시
:%s/old//gn              " 일치 개수만 확인
```

정확한 단어:

```vim
:%s/\<linux\>/WindowS/gc
```

IP 주소:

```vim
:%s/192\.168\.1/172.16.100/gc
```

줄 끝 공백 제거:

```vim
:%s/\s\+$//
```

공백만 있는 줄 포함 빈 줄 삭제:

```vim
:g/^\s*$/d
```

### Step 8. 검색

```vim
/pattern                " 아래 방향 검색
?pattern                " 위 방향 검색
n                       " 같은 방향 다음 결과
N                       " 반대 방향 결과
*                       " 커서 아래 단어를 아래로 검색
#                       " 커서 아래 단어를 위로 검색
:nohlsearch             " 검색 강조 제거
```

### Step 9. 셸 연동

```vim
:!date                  " 명령 실행
:!ls -l /root           " 목록 확인

:read !date             " 현재 줄 다음에 결과 삽입
:10read !date           " 10번째 줄 다음에 삽입
:$read !date            " 파일 끝에 삽입

:.!tr 'a-z' 'A-Z'       " 현재 줄 필터
:1,10!sort              " 1~10번째 줄 정렬
:%!sort                 " 전체 정렬

:w !diff -u /tmp/f.bak -    " 현재 버퍼와 백업 비교
```

> **참고:** `:10!명령`은 10번째 줄을 명령 출력으로 교체한다. 10번째 줄 다음에 삽입하려면 `:10read !명령`을 사용한다.

### Step 10. 권한이 없는 파일 저장

처음부터 권장:

```bash
sudoedit /etc/example.conf
```

이미 편집한 경우:

```vim
:execute 'write !sudo tee ' . shellescape(expand('%:p')) . ' >/dev/null'
```

성공 확인 후 수정 표시 해제:

```vim
:set nomodified
```

단순 경로에서 사용하는 축약형:

```vim
:w !sudo tee % > /dev/null
```

### Step 11. Swap 복구

외부 셸:

```bash
vim -r
vim -r /path/to/file
```

복구 결과를 별도 파일에 저장:

```vim
:w /tmp/file.recovered
```

원본과 비교:

```vim
:!diff -u /path/to/file /tmp/file.recovered
```

다른 편집 세션 확인:

```bash
lsof /path/to/file
fuser -v /path/to/file
pgrep -af '(^|/)(vi|vim|nvim)( |$)'
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 상태 확인

```vim
:file                   " 파일명·줄 수·수정 상태
:pwd                    " 작업 디렉터리
:buffers                " 버퍼 목록
:registers              " 레지스터
:changes                " 변경 위치
:undolist               " Undo 목록
:set modified?          " 수정 상태
:set readonly?          " 읽기 전용 여부
:set swapfile?          " Swap 사용 여부
:set directory?         " Swap 경로 설정
```

외부 셸:

```bash
command -v vi
command -v vim
type -a vi vim
lsof <파일>
fuser -v <파일>
vim -r
```

### 3-2. 안전 편집 표준 절차

```bash
sudoedit /etc/example.conf
```

Vim 내부:

```vim
:set number
:w /tmp/example.conf.bak
```

치환 개수 확인:

```vim
:%s/old//gn
```

확인하며 치환:

```vim
:%s/old/new/gc
```

현재 버퍼와 백업 비교:

```vim
:w !diff -u /tmp/example.conf.bak -
```

저장:

```vim
:w
```

외부 문법 검사:

```vim
:!<서비스별 문법 검사 명령>
```

문제가 없을 때 reload:

```vim
:!sudo systemctl reload <서비스>
```

### 3-3. 대표 함정

| 함정 | 결과 | 올바른 접근 |
|---|---|---|
| 실행 직후 문자 입력 | 명령이 실행됨 | `i`로 입력 모드 |
| `:q` 거부 | 미저장 변경 존재 | `:wq` 또는 `:q!` |
| `:w!` 사용 | OS 권한 우회 실패 | `sudoedit` 또는 `sudo tee` |
| `5dd` 범위 오해 | 예상보다 많거나 적게 삭제 | 현재 줄 포함 총 5줄 |
| `:.,-3d` | 역방향 범위 문제 | `:-3,.d` |
| 치환 부분 일치 | `selinux`까지 변경 | `\<linux\>` |
| IP에서 `.` 미이스케이프 | 임의 문자까지 일치 | `\.` |
| `:10!명령` | 10번째 줄이 교체됨 | `:10read !명령` |
| `:!diff backup %` | 미저장 변경 누락 | `:w !diff -u backup -` |
| Swap 즉시 삭제 | 복구 데이터·충돌 경고 손실 | 세션 확인 후 복구 |
| 저장 후 `:q!` | 저장 내용이 복구되지 않음 | Undo 후 다시 저장 |
| `.swp`를 백업으로 의존 | 정상 종료 시 삭제될 수 있음 | 별도 백업·버전 관리 |

> 📌 **핵심 요약**
> - 모드 복귀: `Esc`
> - 편집: 카운트+오퍼레이터+모션
> - 치환: `:%s/old/new/gc`
> - 백업 비교: `:w !diff -u backup -`
> - 명령 결과 삽입: `:read !명령`
> - 관리자 파일: `sudoedit`
> - 복구: `vim -r`
> - 관련: 4-1. 🎬 VI 3-Mode 아키텍처 & 파일 열기 · 4-2. ⌨️ VI 편집 명령어 (커서·삭제·복사·Ex Mode 범위 조작) · 4-3. 🔁 VI 치환 · 4-4. 🔗 VI Shell 연동 & Swap 파일 복구
