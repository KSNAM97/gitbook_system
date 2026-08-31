# 디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm)

`mkdir`로 디렉터리를 생성하고, 빈 디렉터리는 `rmdir`, 내용이 있는 디렉터리는 `rm -r`로 삭제한다. `rm -rf`는 복구가 어려운 명령이므로 실행 전 대상과 경로를 반드시 검증한다.

## 목차

1. [개요 (Overview)](#개요-overview)
2. [표준 사용 템플릿 (Configuration)](#표준-사용-템플릿-configuration)
3. [검증 및 트러블슈팅 (Verification & Troubleshooting)](#검증-및-트러블슈팅-verification--troubleshooting)
4. [요약](#요약)

---

## 개요 (Overview)

`mkdir`, `rmdir`, `rm`의 차이는 다음과 같다.

| 명령어 | 용도 |
|---|---|
| `mkdir` | 디렉터리 생성 |
| `rmdir` | 빈 디렉터리 삭제 |
| `rm` | 파일 삭제 |
| `rm -r` | 디렉터리와 하위 내용 재귀 삭제 |

`mkdir -p`는 중간 경로가 존재하지 않더라도 필요한 상위 디렉터리를 함께 생성할 때 사용한다.

```bash
mkdir -p /sk/sk-networks/sk-net-a1/sk-net-b1
```

`-p` 없이 실행하면 상위 디렉터리가 먼저 존재해야 한다.

```bash
mkdir /sk/sk-networks/sk-net-a1/sk-net-b1
```

상위 경로가 없을 경우:

```text
mkdir: cannot create directory: No such file or directory
```

> **참고:** `mkdir -p`는 중첩 경로를 편리하게 생성하지만 파일시스템 전체 작업의 원자성을 보장하는 명령은 아니다.

`rmdir`과 `rm -r`은 서로 다른 용도로 구분해서 사용한다. `rmdir`은 빈 디렉터리만 삭제하며 내용이 있으면 실패하는, 의도하지 않은 내용 삭제를 방지하는 안전장치 역할을 한다. `rm -r`은 디렉터리와 하위 내용을 모두 재귀 삭제하므로 삭제 범위를 반드시 확인해야 한다.

`rm -rf`를 안전하게 사용하는 방법은 다음과 같은 순서를 따르는 것이다: 1) `pwd`로 현재 위치 확인, 2) `ls` 또는 `find`로 삭제 대상 확인, 3) 절대경로와 변수 값 확인, 4) `--`로 옵션 해석 종료, 5) 삭제 후 결과 확인.

```bash
pwd
ls -ld -- /home/guest/app
find /home/guest/app -maxdepth 2 -print
rm -rf -- /home/guest/app
```

> **주의:** `rm`으로 삭제한 항목은 일반적으로 휴지통으로 이동하지 않는다. 백업이 없다면 복구가 매우 어렵거나 불가능할 수 있다.

`rm -f`가 모든 오류를 무시하는지에 대해서는, 그렇지 않다. `-f`의 주요 동작은 존재하지 않는 파일을 오류로 처리하지 않는 것, 일부 확인 질문을 생략하는 것, 쓰기 권한이 없는 파일에 대한 확인을 생략하고 삭제를 시도하는 것이다. 다만 상위 디렉터리 권한 부족, 읽기 전용 파일시스템, 파일시스템 I/O 오류, SELinux 또는 ACL에 의한 접근 거부와 같은 문제까지 모두 무시하는 것은 아니다.

---

## 표준 사용 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. `mkdir` — 디렉터리 생성

단일 디렉터리 생성:

```bash
mkdir /sk
```

여러 디렉터리 생성:

```bash
mkdir /sk/sk-energy
mkdir /sk/sk-telecom
mkdir /sk/sk-networks
```

한 명령으로 여러 디렉터리 생성:

```bash
mkdir /sk/sk-energy /sk/sk-telecom /sk/sk-networks
```

현재 위치를 기준으로 생성:

```bash
cd /sk
mkdir sk-energy sk-telecom sk-networks
```

중첩 디렉터리 생성:

```bash
mkdir -p /sk/sk-energy/sk-e-a1
mkdir -p /sk/sk-networks/sk-net-a1/sk-net-b1
```

여러 트리를 동시에 생성:

```bash
mkdir -p \
  /sol/A-class/a1/a2/a3/a4 \
  /sol/B-class/b1/b2/b3/b4
```

생성 결과 확인:

```bash
ls -lR /sk
ls -lR /sol
```

### Step 2. `rmdir` — 빈 디렉터리 삭제

```bash
mkdir /tmp/empty_dir
rmdir /tmp/empty_dir
```

내용이 있는 디렉터리를 삭제하면 실패한다.

```bash
rmdir /sk
```

```text
rmdir: failed to remove '/sk': Directory not empty
```

여러 단계의 빈 디렉터리를 상위 방향으로 삭제:

```bash
rmdir -p /a/b/c
```

경로 중 하나라도 비어 있지 않으면 해당 위치에서 중단된다.

### Step 3. `rm` — 파일 삭제

단일 파일 삭제:

```bash
rm /sol/group
```

여러 파일 삭제:

```bash
rm /sol/grub2.cfg /sol/gshadow
```

삭제 전 확인:

```bash
rm -i /sol/group
```

대량 삭제 전 한 번 확인:

```bash
rm -I /sol/*
```

존재하지 않는 파일을 오류 없이 무시:

```bash
rm -f /sol/ABCD
```

> **참고:** RHEL 계열의 대화형 Root 환경에서는 `rm`, `cp`, `mv`에 `-i` alias가 설정되어 있을 수 있다. 확인 메시지는 `rm` 자체의 기본 동작이 아닐 수 있다.

```bash
type rm
alias rm
```

### Step 4. 디렉터리 재귀 삭제

특정 하위 디렉터리 삭제:

```bash
rm -r /sol/B-class/b1/b2/b3/b4
```

하위 트리 전체 삭제:

```bash
rm -r /sol/B-class/b1
```

확인 없이 재귀 삭제:

```bash
rm -rf -- /sol/B-class/b1
```

### Step 5. 디렉터리는 유지하고 일반 파일만 삭제

다음 명령은 숨김파일을 제외하고, 하위 디렉터리가 매칭되면 삭제하지 못한다.

```bash
rm -f /sol/A-class/a1/a2/a3/*
```

현재 디렉터리 바로 아래의 일반 파일만 확인:

```bash
find /sol/A-class/a1/a2/a3 \
  -mindepth 1 -maxdepth 1 -type f -print
```

확인 후 일반 파일만 삭제:

```bash
find /sol/A-class/a1/a2/a3 \
  -mindepth 1 -maxdepth 1 -type f -delete
```

이 방식은 숨김 일반 파일도 포함한다.

### Step 6. 삭제 전 검증 패턴

```bash
# 1단계: 현재 위치 확인
pwd

# 2단계: 삭제 대상 확인
printf '%s\n' /backup/g*
ls -ld -- /backup/g*

# 3단계: 확인 후 삭제
rm -- /backup/g*

# 4단계: 삭제 결과 확인
find /backup -maxdepth 1 -name 'g*' -print
```

### Step 7. 변수를 사용하는 삭제 스크립트의 방어

```bash
set -u

: "${DIR:?DIR must be set}"

TARGET=$(realpath -e -- "$DIR") || exit 1

case "$TARGET" in
  /home/guest/*)
    ;;
  *)
    echo "허용되지 않은 삭제 경로: $TARGET" >&2
    exit 1
    ;;
esac

printf '삭제 대상: %s\n' "$TARGET"
find "$TARGET" -maxdepth 2 -print
```

확인 후 삭제:

```bash
rm -rf -- "$TARGET"
```

> **참고:** `${DIR:?}`와 `set -u`는 빈 변수와 미정의 변수를 방어한다. 하지만 `/home/guest/data` 대신 `/home/guest`처럼 잘못된 값이 들어간 경우까지 자동으로 막지는 못하므로 허용 경로 검사도 필요하다.

---

## 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
ls -ld <경로>                    # 대상 자체
ls -la <경로>                    # 숨김 포함 내용
ls -lR <경로>                    # 재귀 조회
find <경로> -type d              # 디렉터리 목록
find <경로> -type f              # 파일 목록
du -sh <경로>                    # 사용량 확인
```

### 3-2. 실습 문제

#### 문제 6. 중첩 디렉터리를 한 번에 생성

현재 위치:

```text
/home/guest
```

정답:

```bash
cd /home/guest
mkdir -p app/logs/2024/jan
```

#### 문제 7. 빈 `tmp` 디렉터리 생성 후 삭제

```bash
mkdir /home/guest/app/logs/2024/jan/tmp
rmdir /home/guest/app/logs/2024/jan/tmp
```

#### 문제 8. 상대경로로 `archive/2024/feb` 생성

현재 위치:

```text
/home/guest/work/a
```

정답:

```bash
cd /home/guest/work/a
mkdir -p ../archive/2024/feb
```

생성 결과:

```text
/home/guest/work/archive/2024/feb
```

#### 문제 9. `/home/guest/app` 전체 삭제

삭제 전 확인:

```bash
ls -lR /home/guest/app
```

디렉터리 자체와 모든 내용을 삭제:

```bash
rm -rf -- /home/guest/app
```

검증:

```bash
test ! -e /home/guest/app && echo "삭제 완료"
```

> **주의:** `rm -r /home/guest/app/*`는 `app` 디렉터리 자체를 삭제하지 않으며 숨김파일을 누락할 수 있다.

### 3-3. `/sol` 실습

디렉터리 구조 생성:

```bash
mkdir -p \
  /sol/A-class/a1/a2/a3/a4 \
  /sol/B-class/b1/b2/b3/b4
```

확인:

```bash
ls -lR /sol
```

특정 파일 삭제:

```bash
rm /sol/group
rm /sol/grub2.cfg /sol/gshadow
rm -f /sol/group-
```

특정 디렉터리 삭제:

```bash
rm -r /sol/B-class/b1/b2/b3/b4
rm -rf -- /sol/B-class/b1
```

`a3` 디렉터리 안의 일반 파일만 삭제:

```bash
find /sol/A-class/a1/a2/a3 \
  -mindepth 1 -maxdepth 1 -type f -print
```

확인 후:

```bash
find /sol/A-class/a1/a2/a3 \
  -mindepth 1 -maxdepth 1 -type f -delete
```

### 3-4. 대표 트러블슈팅

#### 시나리오 1. `rm: cannot remove: Is a directory`

`rm`은 기본적으로 일반 파일을 삭제하며 디렉터리에는 `-r`이 필요하다.

```bash
rm -r /path/dir
```

빈 디렉터리라면:

```bash
rmdir /path/dir
```

#### 시나리오 2. `mkdir: No such file or directory`

상위 디렉터리가 존재하지 않는다.

```bash
mkdir /sk/sk-networks/sk-net-a1/sk-net-b1
```

수정:

```bash
mkdir -p /sk/sk-networks/sk-net-a1/sk-net-b1
```

#### 시나리오 3. `rm -f`를 사용했지만 삭제되지 않는다

경로의 각 디렉터리 권한을 확인한다.

```bash
namei -l /path/to/file
```

파일시스템이 읽기 전용인지 확인한다.

```bash
findmnt -T /path/to/file
```

SELinux 로그를 확인한다.

```bash
ausearch -m AVC -ts recent
```

#### 시나리오 4. `rm dir/*` 후 디렉터리가 비어 있지 않다

숨김파일이나 하위 디렉터리가 남았는지 확인한다.

```bash
ls -la /path/dir
find /path/dir -mindepth 1 -maxdepth 1 -print
```

일반 파일만 삭제:

```bash
find /path/dir -mindepth 1 -maxdepth 1 -type f -delete
```

디렉터리 자체까지 모두 삭제:

```bash
rm -rf -- /path/dir
```

#### 시나리오 5. 변수 기반 삭제 명령이 위험한 경로로 확장된다

```bash
: "${DIR:?DIR must be set}"
TARGET=$(realpath -e -- "$DIR") || exit 1
printf 'TARGET=%q\n' "$TARGET"
```

Root 디렉터리와 허용 범위 밖의 경로를 거부한다.

```bash
case "$TARGET" in
  /|/home|/home/guest)
    echo "삭제 금지 경로입니다." >&2
    exit 1
    ;;
  /home/guest/*)
    ;;
  *)
    echo "허용되지 않은 경로입니다." >&2
    exit 1
    ;;
esac
```

---

## 요약

- 중첩 생성: `mkdir -p`
- 빈 디렉터리 삭제: `rmdir`
- 재귀 삭제: `rm -r`
- 강제 재귀 삭제: `rm -rf`
- 삭제 전: `pwd` → `ls/find` → `rm`
- 변수 사용 시 빈 값과 허용 범위를 모두 검증
- 관련: **경로 이동 & 목록 조회 (cd & ls & pwd)** · **복사·이동·와일드카드 (cp · mv · glob)**
