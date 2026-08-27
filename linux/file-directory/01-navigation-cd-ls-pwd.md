# 🧭 경로 이동 & 목록 조회 (cd & ls & pwd)

> **Tag:** #Linux #Filesystem #cd #ls #pwd #AbsolutePath #RelativePath  
> **핵심 요약:** 리눅스 파일시스템에서 현재 위치를 `pwd`로 확인하고, 절대경로와 상대경로를 구분하여 `cd`로 이동하며, `ls` 옵션을 조합해 파일·디렉터리의 속성과 구조를 확인한다.

---

## 1. 📖 개요 (Overview)

절대경로와 상대경로의 차이를 보면, **절대경로**는 최상위 디렉터리 `/`부터 목적지까지 전체 경로를 작성하는 방식이고, **상대경로**는 현재 작업 디렉터리를 기준으로 목적지 경로를 작성하는 방식이다.

```text
/       최상위 디렉터리
.       현재 디렉터리
..      상위 디렉터리
~       현재 사용자의 홈 디렉터리
~guest  guest 사용자의 홈 디렉터리
-       직전 작업 디렉터리(cd에서 사용)
```

예시:

```bash
cd /soldesk/linux/rocky/version9   # 절대경로
cd ../../                          # 상대경로
cd ~/work                          # 현재 사용자 홈 기준
cd ~guest                          # guest 사용자의 홈
cd -                               # 직전 디렉터리로 이동
```

> **참고:** `cd /../home`처럼 경로가 `/`로 시작하면 `..`이 포함되어 있어도 절대경로이다. 상대경로는 `/`로 시작하지 않는다.

자동화에서는 절대경로와 상대경로 중 무엇을 사용해야 하는지에 대해서는, 크론, systemd 서비스, 배치 스크립트처럼 실행 위치가 달라질 수 있는 작업에서는 절대경로 사용을 권장한다. Cron의 기본 작업 디렉터리는 실행 사용자와 구현 환경에 따라 달라질 수 있고, systemd는 `WorkingDirectory=`를 지정하지 않으면 서비스가 기대한 디렉터리에서 실행되지 않을 수 있다. 상대경로가 필요하면 스크립트 시작 시 작업 디렉터리를 명확히 지정한다.

```bash
SCRIPT_DIR=$(cd -- "$(dirname -- "$0")" && pwd)
cd "$SCRIPT_DIR" || exit 1
```

`ls -l`에서 확인해야 할 항목은 파일 종류, 권한, 링크 수, 소유자, 소유 그룹, 크기, 최종 수정 시간, 이름이다.

```text
-rw-r--r--. 1 root root 2124  7월 2 19:36 passwd
│└──┬───┘  │ │    │    │         └─ 파일명
│   │      │ │    │    └─ 최종 수정 시간
│   │      │ │    └─ 크기(Byte)
│   │      │ └─ 소유 그룹
│   │      └─ 소유자
│   └─ 권한
└─ 파일 종류
```

파일 종류:

```text
-  일반 파일
d  디렉터리
l  심볼릭 링크
b  블록 장치
c  문자 장치
p  파이프
s  소켓
```

권한 뒤의 추가 표시는 다음과 같이 해석할 수 있다.

```text
.  SELinux 보안 컨텍스트 등 추가 보안 정보 존재
+  ACL과 같은 추가 접근 제어 정보 존재
```

---

## 2. 🛠️ 표준 사용 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 현재 위치 확인

```bash
pwd
```

출력 예시:

```text
/home/guest
```

현재 디렉터리의 물리적 경로를 확인하려면 다음 명령을 사용할 수 있다.

```bash
pwd -P
```

### Step 2. 절대경로 이동

```bash
cd /home
cd /home/guest
cd /soldesk/linux/rocky/version9
cd /sk/sktel/sales/1team
cd /lg/uplus/display
```

예시:

```bash
cd /soldesk/linux/rocky/version9
pwd
```

```text
/soldesk/linux/rocky/version9
```

### Step 3. 상대경로 이동

현재 위치가 `/soldesk/linux/rocky`일 때:

```bash
cd ../../
pwd
```

```text
/soldesk
```

현재 위치가 `/lg/uplus/display`일 때 `/soldesk/linux`로 이동:

```bash
cd ../../../soldesk/linux
pwd
```

```text
/soldesk/linux
```

현재 위치가 `/soldesk/linux`일 때 `/sk/sktel/sales/1team`으로 이동:

```bash
cd ../../sk/sktel/sales/1team
pwd
```

```text
/sk/sktel/sales/1team
```

현재 위치가 `/lab/linux/projectA/src`일 때 `/lab/linux/projectB/logs`로 이동:

```bash
cd ../../projectB/logs
```

현재 위치가 `/lab/linux/projectB/logs`일 때 `/home/guest/work/c/sub1`으로 이동:

```bash
cd ../../../../home/guest/work/c/sub1
```

또는 홈 디렉터리를 이용한다.

```bash
cd ~guest/work/c/sub1
```

### Step 4. `ls` 기본 옵션

```bash
ls              # 현재 디렉터리의 이름 목록
ls -l           # 상세 정보
ls -a           # 숨김 파일 포함
ls -h           # 크기를 사람이 읽기 쉬운 단위로 표시
ls -S           # 파일 크기 내림차순 정렬
ls -r           # 정렬 결과 역순
ls -t           # 최종 수정 시간 기준 최신순
ls -R           # 하위 디렉터리까지 재귀 조회
ls -s           # 각 항목의 할당된 블록 수 표시
```

> **참고:** `ls -s`는 알파벳순 정렬 옵션이 아니다. 기본 이름 정렬은 별도 옵션이 없어도 적용된다.

### Step 5. 실무에서 자주 사용하는 조합

```bash
ls -alh                         # 숨김 포함 상세 정보
ls -lt | head                   # 최근 수정 항목
ls -lS | head                   # 크기가 큰 항목
ls -ld /etc                     # /etc 내용이 아닌 디렉터리 자체
ls -ld . ..                     # 현재·상위 디렉터리 자체 정보
ls -l . ..                      # 현재·상위 디렉터리의 내용
ls -lR /lab                     # /lab 전체 구조와 상세 정보
ls -ahR /sk/sktel/sales         # 숨김 포함 재귀 조회
```

`ll`은 일반적으로 다음과 같은 alias이지만 모든 시스템에서 보장되지는 않는다.

```bash
type ll
alias ll
```

스크립트와 문서에서는 `ll`보다 `ls -l`을 사용하는 것이 명확하다.

### Step 6. 디렉터리 트리 생성 및 확인 예제

```bash
mkdir -p /soldesk/linux/rocky/version9
mkdir -p /sk/sktel/sales/1team
mkdir -p /lg/uplus/display
```

재귀 조회:

```bash
ls -R /soldesk
ls -R /sk
ls -R /lg
```

`tree`가 설치되어 있다면 다음과 같이 확인할 수 있다.

```bash
dnf -y install tree
tree /soldesk
tree /sk
tree /lg
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
pwd                              # 현재 위치
pwd -P                           # 심볼릭 링크가 해석된 물리적 위치
readlink -f <경로>               # 정규화된 절대경로
ls -ld <경로>                    # 경로 자체 정보
ls -alh <경로>                   # 숨김 포함 내용 조회
stat <파일>                      # 상세 메타데이터
find /lab -maxdepth 2 -type d    # 디렉터리 구조
```

### 3-2. 대표 트러블슈팅

#### 🚨 시나리오 1. 상대경로로 이동했는데 예상과 다른 위치가 나온다

현재 위치부터 먼저 확인한다.

```bash
pwd
```

이동할 경로를 실행 전에 확인한다.

```bash
readlink -f ../../projectB/logs
```

경로가 `/`로 시작하면 절대경로라는 점에 주의한다.

```bash
cd /../home/guest   # 절대경로이며 결과적으로 /home/guest
cd ../home/guest    # 현재 위치 기준 상대경로
```

#### 🚨 시나리오 2. Cron에서는 파일을 찾지 못한다

```bash
crontab -l
journalctl -u crond --since today
```

스크립트에서 절대경로를 사용하거나 작업 디렉터리를 지정한다.

```bash
cd /opt/myapp || exit 1
/usr/bin/cp /opt/myapp/source.txt /opt/myapp/backup/
```

#### 🚨 시나리오 3. `ls`에 파일이 보이지 않는다

숨김파일을 포함하여 조회한다.

```bash
ls -alh
```

삭제되었지만 프로세스가 계속 열고 있는 파일을 확인한다.

```bash
lsof +L1
```

#### 🚨 시나리오 4. 디렉터리 자체가 아니라 내용이 출력된다

```bash
ls -l /etc     # /etc 내부 내용
ls -ld /etc    # /etc 디렉터리 자체
```

> 📌 **핵심 요약**
> - 현재 위치 확인: `pwd`
> - 절대경로: `/`부터 시작
> - 상대경로: 현재 위치부터 시작
> - 표준 조회: `ls -alh`
> - 디렉터리 자체 조회: `ls -ld`
> - 재귀 조회: `ls -lR`
> - 관련: 디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm) · 복사·이동·와일드카드 (cp · mv · glob)

---
