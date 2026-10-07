# 디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm)

> **Tag:** #Linux #mkdir #rmdir #rm #Recursive #DataLoss  
> **핵심 요약:** `mkdir`로 디렉터리를 생성하고, 빈 디렉터리는 `rmdir`, 내용이 있는 디렉터리는 `rm -r`로 삭제한다. `rm -rf`는 복구가 어려운 파괴적 명령이므로 실행 전 대상과 경로를 반드시 검증한다.

---

## 1. 개요 (Overview)

`mkdir`, `rmdir`, `rm`의 차이를 정리하면 다음과 같다.

| 명령어 | 용도 |
|---|---|
| `mkdir` | 디렉터리 생성 |
| `rmdir` | 빈 디렉터리 삭제 |
| `rm` | 파일 삭제 |
| `rm -r` | 디렉터리와 하위 내용 재귀 삭제 |

`mkdir -p`는 중간 경로가 존재하지 않아도 필요한 상위 디렉터리를 함께 생성할 때 사용한다.

```bash
mkdir -p /sk/sk-networks/sk-net-a1/sk-net-b1
```

`-p` 없이 마지막 디렉터리만 생성하려면 상위 경로가 모두 존재해야 한다. 여러 디렉터리를 편리하게 생성하지만, 파일시스템 전체에 대해 원자적 작업을 보장하는 것은 아니다.

`rm -rf`를 안전하게 사용하는 방법은 다음과 같은 순서를 따르는 것이다: 1) `pwd`로 현재 위치 확인, 2) `ls` 또는 `find`로 삭제 대상 확인, 3) 절대경로와 변수 값 검증, 4) 경로 앞에 `--`를 사용해 옵션 해석 종료, 5) 삭제 후 결과 확인.

```bash
pwd
ls -ld -- /home/guest/app
find /home/guest/app -maxdepth 2 -print
rm -rf -- /home/guest/app
```

> **주의:** `rm`은 일반적으로 휴지통을 사용하지 않는다. 파일시스템과 백업 상태에 따라 복구가 매우 어렵거나 불가능할 수 있다.

`rm -f`가 모든 오류를 무시하는지에 대해서는, 그렇지 않다. `-f`는 존재하지 않는 파일을 오류로 처리하지 않고, 일부 확인 질문을 생략할 뿐이며, 권한 부족, 읽기 전용 파일시스템, I/O 오류 같은 모든 오류를 무시하는 옵션은 아니다.

---

## 2. 표준 사용 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. `mkdir` — 디렉터리 생성

```bash
mkdir /sk

mkdir /sk/sk-energy
mkdir /sk/sk-telecom
mkdir /sk/sk-networks
```

여러 디렉터리를 한 번에 생성할 수 있다.

```bash
mkdir /sk/sk-energy /sk/sk-telecom /sk/sk-networks
```

현재 디렉터리 기준으로 생성할 수도 있다.

```bash
cd /sk
mkdir sk-energy sk-telecom sk-networks
```

중첩 디렉터리를 생성한다.

```bash
mkdir -p /sk/sk-networks/sk-net-a1/sk-net-b1
mkdir -p /sol/A-class/a1/a2/a3/a4
mkdir -p /sol/B-class/b1/b2/b3/b4
```

생성 결과를 확인한다.

```bash
ls -lR /sk
ls -lR /sol
```

### Step 2. `rmdir` — 빈 디렉터리 삭제

```bash
mkdir /tmp/empty_dir
rmdir /tmp/empty_dir
```

내용이 있으면 삭제되지 않는다.

```bash
rmdir /sk
```

```text
rmdir: failed to remove '/sk': Directory not empty
```

여러 단계의 빈 디렉터리를 상위 방향으로 삭제한다.

```bash
rmdir -p /a/b/c
```

경로 중 하나라도 비어 있지 않으면 해당 지점에서 실패한다.

### Step 3. `rm` — 파일 삭제

```bash
rm /sol/group
rm /sol/grub2.cfg /sol/gshadow
```

존재하지 않는 파일을 오류 없이 무시한다.

```bash
rm -f /sol/ABCD
```

삭제 전 확인한다.

```bash
rm -i /sol/group
```

대량 삭제에서는 한 번만 확인하는 `-I`도 사용할 수 있다.

```bash
rm -I /sol/*
```

> **참고:** RHEL 계열의 대화형 Root 환경에서는 `rm`, `cp`, `mv`에 `-i` alias가 설정되어 있을 수 있다. 명령 자체의 기본 동작이라고 단정해서는 안 된다.

```bash
type rm
alias rm
```

### Step 4. 디렉터리 재귀 삭제

```bash
rm -r /sol/B-class/b1/b2/b3/b4
```

하위 트리 전체를 삭제한다.

```bash
rm -r /sol/B-class/b1
```

확인 없이 재귀 삭제한다.

```bash
rm -rf -- /sol/B-class/b1
```

### Step 5. 디렉터리는 유지하고 일반 파일만 삭제

`*`는 숨김파일을 포함하지 않으며 하위 디렉터리도 매칭할 수 있다.

```bash
rm -f /sol/A-class/a1/a2/a3/*
```

현재 디렉터리 바로 아래의 일반 파일만 명확하게 삭제하려면 `find`를 사용할 수 있다.

```bash
find /sol/A-class/a1/a2/a3 \
  -mindepth 1 -maxdepth 1 -type f -print
```

출력을 검토한 뒤 삭제한다.

```bash
find /sol/A-class/a1/a2/a3 \
  -mindepth 1 -maxdepth 1 -type f -delete
```

숨김파일을 포함한 일반 파일에도 적용된다.

### Step 6. 삭제 전 3단계 검증

```bash
# 1단계: 현재 위치 확인
pwd

# 2단계: 대상 확인
ls -ld -- /backup/g*
printf '%s\n' /backup/g*

# 3단계: 확인 후 삭제
rm -- /backup/g*
```

변수를 사용하는 스크립트에서는 값과 허용 범위를 검증한다.

```bash
set -u

: "${DIR:?DIR must be set}"

TARGET=$(realpath -e -- "$DIR") || exit 1

case "$TARGET" in
  /home/guest/*) ;;
  *)
    echo "허용되지 않은 삭제 경로: $TARGET" >&2
    exit 1
    ;;
esac

printf '삭제 대상: %s\n' "$TARGET"
rm -rf -- "$TARGET"
```

> **참고:** `${DIR:?}`와 `set -u`는 미정의·빈 변수 문제를 줄이지만, 잘못된 비어 있지 않은 경로까지 막아주지는 않는다. 허용 경로 검증이 함께 필요하다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
ls -ld <경로>                    # 대상 자체 확인
ls -la <경로>                    # 숨김 포함 내용
ls -lR <경로>                    # 재귀 조회
find <경로> -type d              # 디렉터리 목록
find <경로> -type f              # 파일 목록
du -sh <경로>                    # 사용량 확인
```

### 3-2. 대표 실습

#### EX1. `/sk` 디렉터리 구조 생성

```bash
mkdir -p /sk/sk-energy/sk-e-a1
mkdir -p /sk/sk-telecom
mkdir -p /sk/sk-networks/sk-net-a1/sk-net-b1

ls -lR /sk
```

#### EX2. `/sol` 디렉터리 구조 생성

```bash
mkdir -p /sol/A-class/a1/a2/a3/a4
mkdir -p /sol/B-class/b1/b2/b3/b4

ls -lR /sol
```

#### EX3. 특정 파일 삭제

```bash
rm /sol/group
rm /sol/grub2.cfg /sol/gshadow
rm -f /sol/group-
```

#### EX4. 특정 디렉터리 삭제

```bash
rm -r /sol/B-class/b1/b2/b3/b4
rm -rf -- /sol/B-class/b1
```

### 3-3. 대표 트러블슈팅

#### 시나리오 1. `rm: cannot remove: Is a directory`

`rm`은 기본적으로 디렉터리를 삭제하지 않는다.

```bash
rm -r /path/dir
```

비어 있는 디렉터리라면 다음 명령이 더 안전하다.

```bash
rmdir /path/dir
```

#### 시나리오 2. `mkdir: No such file or directory`

상위 디렉터리가 없기 때문이다.

```bash
mkdir /sk/sk-networks/sk-net-a1/sk-net-b1
```

다음과 같이 수정한다.

```bash
mkdir -p /sk/sk-networks/sk-net-a1/sk-net-b1
```

#### 시나리오 3. `rm -rf "$DIR/"` 사용 후 예상하지 못한 경로가 삭제되었다

- 변수 값 출력
- 정규화 경로 확인
- 허용 경로 제한
- 실행 전 목록 출력

```bash
: "${DIR:?DIR must be set}"
TARGET=$(realpath -e -- "$DIR") || exit 1
printf 'TARGET=%q\n' "$TARGET"
find "$TARGET" -maxdepth 2 -print
```

#### 시나리오 4. `rm dir/*`를 실행했는데 디렉터리가 남아 있다

`*`는 디렉터리 자체도 매칭하지만 `-r`이 없으면 삭제되지 않는다. 또한 숨김파일은 기본적으로 매칭하지 않는다.

디렉터리 자체를 포함해 모두 삭제하려는 경우:

```bash
rm -rf -- /path/dir
```

디렉터리는 유지하고 일반 파일만 삭제하려는 경우:

```bash
find /path/dir -mindepth 1 -maxdepth 1 -type f -print
find /path/dir -mindepth 1 -maxdepth 1 -type f -delete
```

>  **핵심 요약**
> - 중첩 생성: `mkdir -p`
> - 빈 디렉터리 삭제: `rmdir`
> - 재귀 삭제: `rm -r`
> - 강제 재귀 삭제: `rm -rf`
> - 삭제 전: `pwd` → `ls/find` → `rm`
> - 관련: 경로 이동 & 목록 조회 (cd & ls & pwd) · 복사·이동·와일드카드 (cp · mv · glob)
