# VI 트러블슈팅 치트시트

VI/Vim 문제는 현재 모드, 버퍼 수정 상태, 파일 권한, 파일시스템 상태, Swap 및 다른 편집 세션을 순서대로 확인한다. Swap을 삭제하기 전에 반드시 복구 필요 여부와 동시 편집 여부를 확인한다.

## 목차

1. [개요 (Overview)](#개요-overview)
2. [증상별 즉시 대응표 (Configuration)](#증상별-즉시-대응표-configuration)
3. [트러블슈팅 시나리오 (Verification & Troubleshooting)](#트러블슈팅-시나리오-verification--troubleshooting)

---

## 개요 (Overview)

문제가 발생했을 때 가장 먼저 확인할 것은 순서대로 현재 모드, 현재 파일명, 버퍼 변경 여부, 읽기 전용 여부, 파일 및 상위 디렉터리 권한, 파일시스템 공간과 마운트 상태, Swap 및 다른 편집 세션, Undo 또는 백업 사용 가능 여부이다.

Vim 내부:

```vim
:file
:set modified?
:set readonly?
:pwd
```

Swap 파일을 바로 삭제하면 안 되는 이유는, 저장하지 않은 편집 내용이 남아 있을 수 있고, 다른 세션이 현재 편집 중일 수 있으며, 공유 파일시스템에서 다른 호스트의 세션일 수 있고, 복구 가능성을 잃을 수 있기 때문이다. 처리 순서는 다음과 같다.

```text
Swap 경고 정보 확인
→ 사용자·호스트·PID 확인
→ 활성 편집 세션 확인
→ 복구 시도
→ 복구본 별도 저장
→ 원본과 비교
→ 필요할 때만 Swap 삭제
```

Swap, Undo, 백업은 서로 목적과 정상 종료 후 처리 방식이 다르다. Swap은 비정상 종료 복구·동시 편집 경고가 목적이며 정상 종료 후 일반적으로 삭제된다. Undo는 현재 세션 실행 취소가 목적이며 설정에 따라 종료 시 사라진다. Persistent Undo는 종료 후에도 Undo를 유지하는 것이 목적이며 별도 Undo 파일로 유지된다. 백업은 원본 복사본 보관이 목적이며 직접 삭제할 때까지 유지된다. 버전 관리는 변경 이력과 비교·복원이 목적이며 저장소에 유지된다.

| 기능 | 목적 | 정상 종료 후 |
|---|---|---|
| Swap | 비정상 종료 복구·동시 편집 경고 | 일반적으로 삭제 |
| Undo | 현재 세션 실행 취소 | 설정에 따라 종료 시 사라짐 |
| Persistent Undo | 종료 후에도 Undo 유지 | 별도 Undo 파일 유지 |
| 백업 | 원본 복사본 보관 | 직접 삭제할 때까지 유지 |
| 버전 관리 | 변경 이력과 비교·복원 | 저장소에 유지 |

---

## 증상별 즉시 대응표 (Configuration)

### 2-1. 모드·입력·종료

| 증상 | 원인 | 조치 |
|---|---|---|
| 문자가 입력되지 않음 | 명령 모드 | `i`, `a`, `o` |
| 키를 누를 때 줄이 삭제됨 | 명령 모드에서 편집 키 입력 | `u` 후 `i` |
| 이상한 키만 동작 | 모드 혼동 | `Esc` 한두 번 |
| `:q` 거부 | 미저장 변경 | `:wq` 또는 `:q!` |
| `:q!` 후 변경이 남아 있음 | 이미 저장함 | Undo 후 다시 저장 또는 백업 복원 |

### 2-2. 저장

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| `E212` | 권한·경로·파일시스템 문제 | `namei`, `findmnt`, `df` 확인 |
| `E45 readonly option` | `'readonly'` 설정 | 권한 확인 후 `:set noreadonly` |
| `E32 No file name` | 파일명 없는 버퍼 | `:w 파일명` |
| `E514 write error` | 디스크·파일시스템 문제 | 공간·inode·로그 확인 |
| Root 파일 저장 실패 | 일반 사용자 권한 | `sudoedit` 또는 `sudo tee` |
| 외부 변경 경고 | 다른 프로세스가 파일 변경 | 비교 후 `:edit!` 여부 결정 |

### 2-3. 편집·치환

| 증상 | 원인 | 조치 |
|---|---|---|
| 들여쓰기 계단식 | 자동 들여쓰기 | `:set paste` 후 `nopaste` |
| 부분 문자열까지 치환 | 단어 경계 없음 | `\<word\>` |
| IP 치환 과다 일치 | `.` 미이스케이프 | `\.` |
| 전체 파일 정렬 | `:%!sort` 사용 | `u`, 좁은 범위 테스트 |
| 범위 줄 수가 예상과 다름 | 끝점 포함 범위 | `:.,+3`은 총 4줄 |
| 현재 줄이 명령 출력으로 바뀜 | `:10!명령` 사용 | 삽입은 `:10read !명령` |

### 2-4. 복구·세션

| 증상 | 원인 | 조치 |
|---|---|---|
| E325 Swap 경고 | 충돌 또는 이전 크래시 | 세션 확인 후 `vim -r` |
| SSH 종료 후 편집 내용 누락 | 세션 종료·프로세스 잔존 | 프로세스와 Swap 확인 |
| `:%d` 저장 후 빈 파일 | 전체 삭제 저장 | 열린 세션 Undo·백업 확인 |
| Swap이 원본 폴더에 없음 | `'directory'` 설정 | `:set directory?`, `vim -r` |
| Swap 복구 내용이 오래됨 | Swap 갱신·복구 한계 | 원본과 복구본 비교 |

---

## 트러블슈팅 시나리오 (Verification & Troubleshooting)

### 시나리오 1. 키를 눌러도 입력되지 않거나 줄이 삭제된다

명령 모드로 복귀:

```text
Esc
```

실수로 삭제했다면:

```vim
u
```

입력 모드 진입:

```vim
i
```

화면 하단 확인:

```text
-- INSERT --
```

### 시나리오 2. `:q`가 거부된다

현재 상태:

```vim
:file
:set modified?
```

저장 후 종료:

```vim
:wq
```

저장하지 않은 변경을 버리고 종료:

```vim
:q!
```

> 이미 저장한 변경은 `:q!`로 복원되지 않는다.

### 시나리오 3. `E212: Can't open file for writing`

파일과 상위 경로 권한:

```bash
namei -l /path/to/file
```

마운트 상태:

```bash
findmnt -T /path/to/file
```

공간과 inode:

```bash
df -h /path/to/file
df -i /path/to/file
```

SELinux와 ACL:

```bash
ls -lZ /path/to/file
getfacl /path/to/file
```

처음부터 다시 편집할 수 있다면:

```bash
sudoedit /path/to/file
```

현재 버퍼를 유지해야 한다면:

```vim
:execute 'write !sudo tee ' . shellescape(expand('%:p')) . ' >/dev/null'
```

저장 성공 확인 후:

```vim
:set nomodified
```

### 시나리오 4. E325 Swap 경고가 발생한다

Swap 경고에 표시된 정보를 확인한다.

```text
소유자
호스트
PID
파일명
수정 시간
프로세스 실행 여부
```

외부 셸:

```bash
lsof /path/to/file
fuser -v /path/to/file
pgrep -af '(^|/)(vi|vim|nvim)( |$)'
```

활성 세션이 없으면:

```bash
vim -r /path/to/file
```

복구본을 별도로 저장:

```vim
:w /tmp/file.recovered
```

비교:

```vim
:!diff -u /path/to/file /tmp/file.recovered
```

확실한 경우에만 Swap 삭제:

```bash
rm -- /path/to/.file.swp
```

### 시나리오 5. 웹에서 복사한 설정의 들여쓰기가 밀린다

```vim
:set paste
```

입력 모드에서 붙여넣기:

```vim
i
```

완료:

```text
Esc
```

옵션 복구:

```vim
:set nopaste
```

### 시나리오 6. `linux`를 치환했더니 `selinux`도 변경되었다

실행 취소:

```vim
u
```

독립된 단어만 치환:

```vim
:%s/\<linux\>/WIN/gc
```

대소문자 무시:

```vim
:%s/\<linux\>/WIN/gic
```

### 시나리오 7. IP 치환이 예상보다 넓게 적용되었다

잘못된 예:

```vim
:%s/192.168.1.1/10.0.0.1/g
```

수정:

```vim
:%s/192\.168\.1\.1/10.0.0.1/gc
```

### 시나리오 8. `:%!sort`로 파일 전체가 정렬되었다

즉시 Undo:

```vim
u
```

백업:

```vim
:w /tmp/file.bak
```

좁은 범위 테스트:

```vim
:1,10!sort
```

전체 적용 전 결과 확인:

```vim
:w !diff -u /tmp/file.bak -
```

### 시나리오 9. `:%d` 후 저장했다

같은 세션이 열려 있다면:

```vim
u
:w
```

Undo 목록:

```vim
:undolist
```

과거 상태 이동:

```vim
:earlier 1f
```

필요한 상태에 도달한 후 저장:

```vim
:w
```

세션을 종료했다면:

```bash
vim -r /path/to/file
find /path/to -maxdepth 1 \
  \( -name '.*.sw?' -o -name '*.bak' -o -name '*.rpmnew' -o -name '*.rpmsave' \)
```

> 정상 저장·종료 후에는 Swap이 삭제될 수 있으므로 복구를 보장할 수 없다.

### 시나리오 10. SSH가 종료되어 편집 내용이 사라진 것 같다

프로세스 확인:

```bash
pgrep -af '(^|/)(vi|vim|nvim)( |$)'
lsof /path/to/file
```

`tmux` 또는 `screen` 세션 확인:

```bash
tmux ls
screen -ls
```

Swap 복구 목록:

```bash
vim -r
```

특정 파일 복구:

```bash
vim -r /path/to/file
```

### 시나리오 11. `dd`를 잘못 실행한 후 여러 작업을 더 했다

Undo:

```vim
u
```

Undo 목록 확인:

```vim
:undolist
```

시간 기준 복구:

```vim
:earlier 5m
```

다시 앞으로 이동:

```vim
:later 5m
```

Persistent Undo 설정:

```vim
:set undofile
:set undodir?
```

`~/.vimrc` 예시:

```vim
set undofile
set undodir=~/.vim/undo//
```

외부 셸에서 디렉터리 준비:

```bash
mkdir -p ~/.vim/undo
chmod 700 ~/.vim/undo
```

### 시나리오 12. Swap 파일을 검색했지만 나오지 않는다

현재 Vim 설정 확인:

```vim
:set directory?
```

전체 복구 대상 확인:

```bash
vim -r
```

일반적인 Swap 확장자 검색:

```bash
find /home /tmp /var/tmp \
  -type f -name '.*.sw?' 2>/dev/null
```

> 설정에 따라 다른 경로 또는 인코딩된 파일명으로 저장될 수 있으므로 `vim -r` 결과도 확인한다.

## 요약

- 먼저 `Esc`로 명령 모드 확보
- 저장 문제: 권한·마운트·공간·inode 확인
- Root 파일: `sudoedit`
- 치환: `\<word\>`와 `gc`
- 필터 실수: `u`
- Swap 경고: 세션 확인 → 복구 → 비교 → 삭제
- 정상 저장·종료 후 Swap 복구는 보장되지 않음
- 관련: **VI 3-Mode 아키텍처 & 파일 열기** · **VI 편집 명령어 (커서·삭제·복사·Ex Mode 범위 조작)** · **VI 치환** · **VI Shell 연동 & Swap 파일 복구**
