# 🎬 VI 3-Mode 아키텍처 & 파일 열기

> **Tag:** #Linux #Vi #Vim #Editor #ModalEditor  
> **핵심 요약:** VI/Vim은 명령·입력·명령행(Ex) 모드로 동작하는 모달 편집기이다. 실행 직후에는 명령 모드이며, 파일은 저장 명령을 실행해야 실제 디스크에 반영된다.

---

## 1. 📖 개요 (Overview)

서버 환경에서 VI/Vim을 사용하는 이유는 터미널에서 실행할 수 있고 키보드만으로 파일 조회와 편집이 가능하기 때문이다. GUI가 없는 서버에서도 사용 가능하고, SSH 원격 접속 환경에서 사용 가능하며, 커서 이동·삭제·복사·치환·외부 명령 실행을 지원하고, 다양한 Unix/Linux 환경에서 널리 사용된다. Vim은 전통적인 vi와 호환성을 유지하면서 기능을 확장한 편집기다.

> **참고:** 모든 Unix/Linux 또는 최소 컨테이너 이미지에 vi가 반드시 설치되는 것은 아니다. 최소 이미지에서는 편집기가 없거나 `vi`, `vim-minimal`, BusyBox vi 등 서로 다른 구현이 제공될 수 있다.

설치 상태 확인:

```bash
command -v vi
command -v vim
type -a vi vim
```

Rocky Linux에서 패키지 확인:

```bash
rpm -qa | grep -i '^vim'
```

필요한 경우 Vim 설치:

```bash
dnf -y install vim-enhanced
```

`visudo`, `crontab -e`, `git commit`이 항상 VI를 실행하는 것은 아니다. 다음 환경변수와 프로그램별 설정에 따라 편집기가 달라질 수 있다.

```bash
echo "$VISUAL"
echo "$EDITOR"
```

일시적으로 Vim 지정:

```bash
EDITOR=vim crontab -e
```

Git 편집기 지정:

```bash
git config --global core.editor vim
```

`visudo`는 환경과 보안 정책에 따라 편집기 선택이 제한될 수 있다.

VI의 모드 구조는 기초 학습에서는 다음 세 가지 모드로 구분한다.

```text
┌──────────────────┐     i, a, I, A, o, O     ┌──────────────────┐
│ 명령 모드        │ ───────────────────────▶ │ 입력 모드        │
│ Normal Mode      │                           │ Insert Mode      │
└────────┬─────────┘ ◀──────── Esc ─────────── └──────────────────┘
         │
         │ :
         ▼
┌──────────────────┐
│ 명령행 모드      │
│ Command-line/Ex  │
└────────┬─────────┘
         │ Esc 또는 명령 완료
         ▼
    명령 모드
```

> **참고:** Vim에는 Visual 모드 등 추가 모드도 있지만, 기초 과정에서는 명령·입력·명령행 모드를 중심으로 학습한다.

각 모드의 역할은 다음과 같다. 명령 모드는 실행 직후 또는 `Esc`로 진입하며 이동, 삭제, 복사, 붙여넣기, 실행 취소를 담당한다. 입력 모드는 `i`, `a`, `I`, `A`, `o`, `O`로 진입하며 실제 텍스트 입력을 담당한다. 명령행 모드는 명령 모드에서 `:`로 진입하며 저장, 종료, 치환, 범위 작업, 셸 연동을 담당한다.

| 모드 | 진입 | 역할 |
|---|---|---|
| 명령 모드 | 실행 직후 또는 `Esc` | 이동, 삭제, 복사, 붙여넣기, 실행 취소 |
| 입력 모드 | `i`, `a`, `I`, `A`, `o`, `O` | 실제 텍스트 입력 |
| 명령행 모드 | 명령 모드에서 `:` | 저장, 종료, 치환, 범위 작업, 셸 연동 |

모드가 혼동되면 다음과 같이 처리한다.

```text
Esc를 한두 번 눌러 명령 모드로 복귀
```

---

## 2. 🛠️ 표준 사용 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 파일 열기

```bash
vi <파일>
vim <파일>
```

파일이 존재하면 해당 파일을 열고, 존재하지 않으면 새 버퍼를 연다.

```bash
vi newfile
```

> **참고:** 존재하지 않는 파일을 열었다고 즉시 디스크에 파일이 생성되는 것은 아니다. `:w` 등으로 저장해야 파일이 생성된다.

### Step 2. 특정 위치에서 열기

10번째 줄에서 열기:

```bash
vi +10 /backup/passwd
```

마지막 줄에서 열기:

```bash
vi + /backup/passwd
```

패턴이 처음 나타나는 위치에서 열기:

```bash
vi +/guest /etc/passwd
```

읽기 전용으로 열기:

```bash
vi -R /etc/passwd
view /etc/passwd
```

여러 파일 열기:

```bash
vi file1 file2 file3
```

다음·이전 파일:

```vim
:n
:N
```

변경 사항을 버리고 다음 파일:

```vim
:n!
```

### Step 3. 입력 모드 진입

| 키 | 동작 |
|---|---|
| `i` | 현재 커서 앞에서 입력 |
| `a` | 현재 커서 뒤에서 입력 |
| `I` | 현재 줄의 첫 번째 비공백 문자 앞에서 입력 |
| `A` | 현재 줄 끝에서 입력 |
| `o` | 현재 줄 아래에 새 줄 생성 후 입력 |
| `O` | 현재 줄 위에 새 줄 생성 후 입력 |

입력 모드 종료:

```text
Esc
```

### Step 4. 저장과 종료

```vim
:w                              " 저장
:w /tmp/file.bak                " 현재 버퍼를 다른 파일에도 저장
:q                              " 종료
:wq                             " 저장 후 종료
:x                              " 변경된 경우 저장 후 종료
:q!                             " 저장하지 않은 변경을 버리고 종료
```

명령 모드 단축키:

```vim
ZZ                              " :x와 유사
ZQ                              " :q!와 유사
```

주의사항:

- `:w!`는 Vim 내부의 일부 읽기 전용 경고를 강제로 처리할 수 있다.
- 운영체제의 파일 권한, 읽기 전용 파일시스템, 디스크 오류까지 우회하지는 못한다.
- 이미 저장한 변경은 이후 `:q!`를 실행해도 디스크에서 되돌아가지 않는다.

### Step 5. 다른 이름으로 저장

현재 버퍼 내용을 다른 파일에도 저장하되 편집 대상은 유지:

```vim
:w /tmp/passwd.bak
```

다른 이름으로 저장하고 편집 대상도 새 파일로 전환:

```vim
:saveas /tmp/passwd.new
```

현재 파일명 확인:

```vim
:file
```

### Step 6. 다른 파일 열기

```vim
:e /etc/hosts
```

저장하지 않은 변경이 있으면 파일 전환이 거부될 수 있다.

변경 사항을 버리고 다시 열기:

```vim
:e!
```

> **참고:** `:e!`는 현재 버퍼의 저장하지 않은 변경을 버린다. 실행 전에 정말 버려도 되는지 확인한다.

분할 창:

```vim
:split /etc/hosts
:vsplit /etc/passwd
```

창 이동:

```text
Ctrl+w w
```

창 닫기:

```vim
:q
```

### Step 7. Swap 파일의 기본 개념

Vim은 편집 중 복구 정보를 저장하기 위해 Swap 파일을 사용할 수 있다.

일반적인 예:

```text
원본: /backup3/passwd
Swap: /backup3/.passwd.swp
```

하지만 Swap 위치는 Vim의 `'directory'` 옵션에 따라 달라질 수 있다.

```vim
:set directory?
:set swapfile?
```

중요한 동작:

- 편집 내용은 Vim 버퍼에 존재한다.
- `:w`를 실행해야 원본 파일에 저장된다.
- Swap은 비정상 종료 시 복구를 돕는 임시 파일이다.
- Swap은 완전한 백업이나 버전 관리 시스템이 아니다.
- 정상 종료 시 일반적으로 Swap이 삭제된다.
- 다른 세션에서 편집 중일 때도 Swap 경고가 발생할 수 있다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 확인 명령

외부 셸:

```bash
command -v vi
command -v vim
type -a vi vim
rpm -qa | grep -i '^vim'
```

Vim 내부:

```vim
:version
:file
:pwd
:set modified?
:set readonly?
:set swapfile?
:set directory?
```

### 3-2. 대표 트러블슈팅

#### 🚨 시나리오 1. 키를 눌렀는데 문자가 입력되지 않는다

실행 직후에는 명령 모드이다.

```text
Esc
i
```

화면 아래의 다음 표시를 확인한다.

```text
-- INSERT --
```

편집 완료:

```text
Esc
```

저장 후 종료:

```vim
:wq
```

#### 🚨 시나리오 2. `:q`가 거부된다

오류 예:

```text
E37: No write since last change
```

저장 후 종료:

```vim
:wq
```

변경을 버리고 종료:

```vim
:q!
```

#### 🚨 시나리오 3. `E212: Can't open file for writing`

가능한 원인:

- 파일 또는 상위 디렉터리 권한 부족
- 읽기 전용 파일시스템
- 존재하지 않는 상위 경로
- 디스크 공간 또는 inode 부족
- 잘못된 파일명
- SELinux 또는 ACL 정책

외부 셸에서 확인:

```bash
namei -l /path/to/file
findmnt -T /path/to/file
df -h /path/to/file
df -i /path/to/file
ls -lZ /path/to/file
getfacl /path/to/file
```

처음부터 관리자 권한으로 편집:

```bash
sudoedit /etc/example.conf
```

또는:

```bash
sudo vi /etc/example.conf
```

> **참고:** `sudoedit`는 사용자용 임시 사본을 편집한 뒤 권한 있는 방식으로 원본에 반영하므로 일반적으로 권장할 수 있다.

#### 🚨 시나리오 4. 읽기 전용으로 열었는데 변경이 입력되었다

`vi -R` 또는 `view`는 의도치 않은 저장을 줄이지만 절대적인 보안 기능은 아니다.

변경을 버리고 종료:

```vim
:q!
```

현재 상태 확인:

```vim
:set readonly?
:set modified?
```

> 📌 **핵심 요약**
> - 실행 직후: 명령 모드
> - 입력: `i`, `a`, `I`, `A`, `o`, `O`
> - 모드 초기화: `Esc`
> - 저장: `:w`
> - 저장 후 종료: `:wq`, `:x`
> - 변경 버리기: `:q!`
> - 파일은 저장해야 디스크에 생성·반영됨
> - 관련: VI 편집 명령어 (커서·삭제·복사·Ex Mode 범위 조작) · VI 치환 · VI Shell 연동 & Swap 파일 복구

---
