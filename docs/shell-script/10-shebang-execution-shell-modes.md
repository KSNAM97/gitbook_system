# Shell Script - Shebang · 실행 방법 · Login/Non-Login · 대화형/비대화형

> **Tag:** #Linux #ShellScript #Bash #Shebang #LoginShell #NonLoginShell #대화형쉘 #비대화형쉘
> **핵심 요약:** 스크립트 첫 줄의 `#!`(Shebang)은 어떤 인터프리터로 실행할지 지정하며, 실행 권한을 부여해 직접 실행하거나 `bash script.sh`로 호출할 수 있다. 쉘은 로그인 여부에 따라 **Login Shell / Non-Login Shell**, 사용자 입력 방식에 따라 **대화형(Interactive) / 비대화형(Non-Interactive)** 두 축으로 구분되며, 각각 읽어들이는 설정 파일과 활성화되는 기능(history, alias, job control 등)이 다르다.

---

## 1. 개요 (Overview)

**Shebang(`#!`)** 은 shabang·hashbang이라고도 불리며, 스크립트 파일의 맨 첫 줄에서 해당 스크립트를 실행할 인터프리터의 경로를 지정한다. 인터프리터 경로는 절대 경로 또는 현재 디렉터리 기준 상대 경로로 지정하며, 변수는 사용할 수 없고 옵션은 하나만 지정할 수 있다. `#!/bin/bash`, `#!/usr/bin/awk -f`, `#!/usr/bin/perl` 처럼 스크립트 언어별로 각각 다른 인터프리터를 지정한다. OS는 실행 권한을 가진 파일을 실행할 때 첫 줄의 `#!` 를 만나면 이어지는 경로를 인터프리터로 취급해 파일 전체를 그 인터프리터에 넘긴다. `bash script.sh` 처럼 인터프리터를 직접 호출해서 실행하면 파일 안의 Shebang은 bash에 의해 주석으로 처리되어 무시된다. Shebang 줄에 추가 주석을 붙이면 오류가 발생하며, `source` 로 읽어들이는 스크립트나 `~/.bashrc`, `~/.profile` 같은 환경설정 파일, 자동완성 함수처럼 스크립트 파일을 직접 실행하지 않는 경우에는 Shebang이 필요 없다. OS마다 인터프리터의 설치 위치가 다를 수 있으므로 `#!/usr/bin/env python` 처럼 `env`를 경유해 `$PATH`에서 실행 파일을 탐색하게 만들 수도 있고, `env -S` 옵션을 쓰면 `#!/usr/bin/env -S gawk -v AA=100 -f` 처럼 여러 옵션을 Shebang 한 줄에 담을 수 있다.

**스크립트 실행 방법**은 크게 두 가지다. 첫째, `chmod +x script.sh`(또는 `chmod 775`)로 실행 권한을 부여한 뒤 `/path/script.sh` 형태로 직접 실행하며, 이 경우 Shebang이 그대로 적용된다. 둘째, `bash script.sh` 또는 `sh script.sh` 처럼 인터프리터 명령으로 호출하며, 이때는 파일에 실행 권한이 없어도 실행할 수 있다. 반복 실행하거나 시스템을 변경하지 않는 스크립트는 실행 권한을 부여해 두고, 애플리케이션 설치처럼 시스템을 변경하는 일회성 스크립트는 `bash`로 호출하는 것이 안전하다.

**Login Shell**은 SSH·Telnet 등으로 원격 접속해 로그인 절차를 거쳐 진입하는 환경이다. `echo $0` 결과의 첫 문자가 `-` 이면 로그인 쉘이며, `shopt -q login_shell; echo $?` (0이면 로그인 쉘)로도 확인할 수 있다. Login Shell은 `/etc/profile → ~/.bash_profile → ~/.bash_login → ~/.profile` 순서로 파일을 읽어들인다(먼저 찾은 파일이 있으면 그 이후 파일은 읽지 않는다 — 예를 들어 `~/.bash_profile`이 있으면 `~/.profile`은 실행되지 않는다). `/etc/profile`은 내부에서 `source`로 `/etc/bash.bashrc`를, `~/.profile`은 `~/.bashrc`를 불러들인다. 로그아웃/`exit` 시에는 `~/.bash_logout`이 실행된다. **Non-Login Shell**은 데스크톱 터미널 프로그램을 실행했을 때의 환경으로, `/etc/bash.bashrc → ~/.bashrc` 만 읽어들인다. Non-Login Shell에서도 `bash -l` 또는 `sh -l`로 Login Shell 환경을 강제로 만들 수 있다. `.bash_profile`과 `.profile`이 별도로 존재하는 이유는 bash 전용 설정과 sh 호환 설정을 구분하기 위함이다. `.rc` 확장자의 유래는 1965년 MIT CTSS의 runcom(run commands) 기능에서 앞글자를 딴 것이다.

Real user id와 Effective user id도 구분해야 한다. **Real UID**는 로그인했을 때의 ID로 읽기 전용 환경변수 `$UID`에 저장되고, **Effective UID**는 SET UID가 설정된 프로그램을 실행했을 때 바뀌는 값으로 `$EUID`에 저장되며 실제 명령 실행 권한을 결정한다. 사용자가 직접 만든 alias·함수·자동완성 설정은 `~/.bashrc.d/{aliases,functions,completions}/` 처럼 디렉터리를 나눠 관리하고, `~/.bashrc`에서 `for file in ~/.bashrc.d/aliases/*.sh; do source "$file"; done` 형태로 순회하며 불러들이면 설정 파일이 커져도 관리하기 쉽다.

쉘 실행 환경은 로그인 여부와 별개로 **대화형(Interactive)** 과 **비대화형(Non-Interactive)** 으로도 나뉜다. 대화형 쉘은 프롬프트를 통해 사용자로부터 직접 명령을 입력받아 실행하는 환경이고, 비대화형 쉘은 스크립트 파일 등을 실행하는 환경이다. `history`, `alias`, job control(`bg`, `fg`, `suspend`) 같은 기능은 대화형 쉘 전용이라 비대화형 쉘에서는 비활성화된다. 현재 쉘이 대화형인지는 옵션 플래그가 저장된 `$-` 변수에 `i` 가 포함되는지로 판단한다: `case $- in *i*) echo interactive;; *) echo non-interactive;; esac`. 비대화형 쉘의 대표적 차이점은 다섯 가지다. ① alias 비활성화(사용자마다 alias 설정이 달라 스크립트가 오작동할 수 있으므로), ② history 확장 비활성화, ③ job control 비활성화(단, `&` 로 만든 백그라운드 작업 자체와 `jobs`·`wait`·`disown` 명령은 사용 가능), ④ `exec` 명령 실패 시 대화형 쉘은 에러 메시지만 표시하지만 비대화형 쉘은 즉시 종료, ⑤ 시작 시 읽는 파일이 다름 — 대화형 쉘은 `~/.bashrc`를, 비대화형 쉘은 `$BASH_ENV`에 지정된 파일을 읽는다(기본값은 비어 있음).

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** Bash 기반 Linux 셸 환경 (RHEL 계열 기본 `/bin/bash`).

### Step 1. Shebang 작성 표준 패턴

```bash
#!/bin/bash                 # bash 스크립트
#!/bin/sh                   # POSIX sh 스크립트
#!/usr/bin/awk -f           # awk 스크립트 (옵션 1개까지 허용)
#!/usr/bin/env python        # $PATH에서 python 탐색
#!/usr/bin/env -S gawk -v AA=100 -f   # env -S 로 옵션 여러 개 전달
```

### Step 2. 실행 방법 표준 패턴

```bash
# 방법 1: 실행 권한 부여 후 직접 실행 (Shebang이 적용됨)
chmod +x script.sh
./script.sh

# 방법 2: 인터프리터로 직접 호출 (실행 권한 불필요, Shebang은 주석 처리됨)
bash script.sh
sh script.sh
```

### Step 3. Login/Non-Login Shell 판별과 전환

```bash
echo $0                       # 맨 앞이 '-'면 Login Shell
shopt -q login_shell; echo $?  # 0이면 Login Shell

bash -l                       # Non-Login Shell에서 Login Shell처럼 시작
sh -l
```

### Step 4. 대화형/비대화형 판별

```bash
case $- in
    *i*) echo "interactive shell" ;;
    *)   echo "non-interactive shell" ;;
esac
```

### Step 5. `~/.bashrc.d` 로 설정 파일 분리 관리

```bash
# ~/.bashrc 에 추가
for file in ~/.bashrc.d/aliases/*.sh
do
    source "$file"
done

for file in ~/.bashrc.d/functions/*.sh
do
    source "$file"
done

for file in ~/.bashrc.d/completions/*.sh
do
    source "$file"
done

unset -v file
```

### Step 6. 기본 쉘 변경과 즉시 반영

```bash
chsh -s /bin/zsh <user>        # /etc/passwd 의 7번째 필드(기본 쉘) 변경
cat /etc/passwd | grep <user>   # 변경 결과 확인 (다음 로그인부터 적용)

exec /bin/zsh                   # 현재 세션에 즉시 반영 (재로그인 없이 쉘 교체)
```

### Step 7. 추가 쉘 설치와 `/etc/shells` 등록

```bash
dnf install -y csh zsh tcsh      # 설치 시 /etc/shells 에 자동 등록됨
cat /etc/shells                  # 로그인 쉘로 지정 가능한 쉘 목록 확인
chsh -s /bin/zsh                 # 목록에 있어야 chsh가 허용됨
```

### Step 8. 비대화형 쉘 초기화 파일 지정 (`BASH_ENV`)

```bash
# 비대화형 쉘(스크립트 실행)이 시작될 때 읽어들일 파일을 지정
export BASH_ENV=~/.bash_env

# ~/.bash_env 예시: 스크립트에서 공통으로 쓸 함수/변수 미리 로드
cat > ~/.bash_env << 'EOF'
log() { echo "[$(date +%T)] $*"; }
EOF

bash -c 'log "스크립트 시작"'    # 비대화형 쉘에서도 log 함수 사용 가능
```

### Step 9. Login/Non-Login × 대화형/비대화형 조합 정리표

| 실행 형태 | Login 여부 | 대화형 여부 | 읽는 파일 |
|---|---|---|---|
| SSH 접속 후 프롬프트 사용 | Login | 대화형 | `/etc/profile` → `~/.bash_profile` |
| 데스크톱 터미널 앱 실행 | Non-Login | 대화형 | `/etc/bash.bashrc` → `~/.bashrc` |
| `bash script.sh` 로 스크립트 실행 | Non-Login | 비대화형 | `$BASH_ENV` (지정 시) |
| `bash -l -c "명령"` | Login | 비대화형 | `/etc/profile` → `~/.bash_profile` |
| cron이 실행하는 스크립트 | Non-Login | 비대화형 | `$BASH_ENV` (지정 시) |

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
echo $0                    # Login Shell 여부(맨 앞 '-')
echo $-                    # 옵션 플래그 (i 포함 시 대화형)
echo $UID $EUID             # Real UID / Effective UID 비교
```

### 3-2. 트러블슈팅 시나리오

#### 시나리오 1. Shebang을 고쳤는데 적용되지 않음

- **원인:** `bash script.sh` 처럼 인터프리터를 직접 호출해서 실행하면 파일 내부의 Shebang은 무시(주석 처리)된다.
- **해결:** Shebang을 실제로 적용하려면 `chmod +x` 로 실행 권한을 준 뒤 `./script.sh` 형태로 직접 실행해야 한다.

#### 시나리오 2. `chsh` 로 기본 쉘을 바꿨는데 현재 세션에 반영되지 않음

- **원인:** `/etc/passwd` 의 로그인 쉘 필드는 다음 로그인부터 적용되며, 이미 열려 있는 세션의 `$SHELL` 값은 즉시 바뀌지 않는다.
- **해결:** 즉시 적용하려면 `exec /bin/bash` 로 현재 쉘을 교체한다.

#### 시나리오 3. 스크립트 안에서 정의한 alias가 동작하지 않음

- **원인:** 스크립트는 기본적으로 비대화형 쉘에서 실행되며, 비대화형 쉘은 alias 기능이 비활성화되어 있다.
- **해결:** alias 대신 함수로 정의하거나, `shopt -s expand_aliases` 로 명시적으로 활성화한다(스크립트 최상단에서 alias 정의보다 먼저 설정해야 한다).

#### 시나리오 4. cron으로 실행한 스크립트가 터미널에서는 되던 명령을 못 찾음(`command not found`)

- **원인:** cron이 실행하는 쉘은 Non-Login·비대화형이라 `~/.bashrc`, `~/.bash_profile`을 읽지 않으므로 `$PATH`가 대화형 쉘보다 훨씬 짧다.
- **해결:** 스크립트 안에서 필요한 명령을 절대 경로로 지정하거나, 스크립트 최상단에서 `export PATH=...`로 필요한 경로를 직접 추가한다.

#### 시나리오 5. `chsh -s /bin/zsh` 가 `zsh is an invalid shell` 로 거부됨

- **원인:** 지정하려는 쉘이 `/etc/shells`에 등록되어 있지 않다.
- **해결:** 쉘 패키지를 설치하면(`dnf install -y zsh`) 보통 `/etc/shells`에 자동 등록되며, 등록 여부는 `cat /etc/shells`로 확인한다.

---

>  **핵심 요약**
> - Shebang은 인터프리터 지정 전용, 변수 불가·옵션 1개 제한, 직접 실행할 때만 적용됨
> - Login Shell(`/etc/profile`→`~/.bash_profile`)과 Non-Login Shell(`/etc/bash.bashrc`→`~/.bashrc`)은 읽어들이는 설정 파일이 다름
> - 대화형/비대화형은 `$-` 변수의 `i` 유무로 판별하며, alias·history·job control은 비대화형에서 비활성화
> - 관련: Shell Script - 변수와 환경변수 (커널·쉘 개념 포함) · Shell Script - 조건문 (if · case) · Shell Script - 통합 정리
