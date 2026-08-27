# ⚡ 파일 압축·아카이브 명령어 퀵 레퍼런스

> **Tag:** #Linux #Compression #Archive #tar #gzip #bzip2 #xz #QuickReference #CheatSheet  
> **핵심 요약:** gzip·bzip2·xz 단일 파일 압축과 tar 아카이브 생성·목록·추출·검증 명령을 빠르게 조회하는 문서.

---

## 1. 🛠️ 단일 파일 압축·해제

### 1-1. gzip

```bash
gzip file                        # file을 file.gz로 압축하고 원본 삭제
gzip -k file                     # 원본을 유지하면서 file.gz 생성
gzip -c file > file.gz           # 압축 결과를 표준 출력으로 보내 file.gz에 저장

gzip -d file.gz                  # file.gz 압축 해제
gunzip file.gz                   # gzip -d와 동일하게 압축 해제
gzip -dc file.gz > file          # 압축 파일을 유지하면서 해제 결과 저장

zcat file.gz                     # 압축을 풀지 않고 텍스트 내용 출력
gzip -l file.gz                  # 압축 전후 크기와 압축률 확인
gzip -t file.gz                  # gzip 파일 무결성 검사
```

### 1-2. bzip2

```bash
bzip2 file                       # file을 file.bz2로 압축하고 원본 삭제
bzip2 -k file                    # 원본을 유지하면서 file.bz2 생성
bzip2 -c file > file.bz2         # 압축 결과를 표준 출력으로 저장

bzip2 -d file.bz2                # file.bz2 압축 해제
bunzip2 file.bz2                 # bzip2 -d와 동일하게 압축 해제
bzip2 -dc file.bz2 > file        # 압축 파일을 유지하면서 해제 결과 저장

bzcat file.bz2                   # 압축을 풀지 않고 텍스트 내용 출력
bzip2 -t file.bz2                # bzip2 파일 무결성 검사
```

### 1-3. xz

```bash
xz file                          # file을 file.xz로 압축하고 원본 삭제
xz -k file                       # 원본을 유지하면서 file.xz 생성
xz -c file > file.xz             # 압축 결과를 표준 출력으로 저장

xz -d file.xz                    # file.xz 압축 해제
unxz file.xz                     # xz -d와 동일하게 압축 해제
xz -dc file.xz > file            # 압축 파일을 유지하면서 해제 결과 저장

xzcat file.xz                    # 압축을 풀지 않고 텍스트 내용 출력
xz -l file.xz                    # 압축 크기와 관련 정보 확인
xz -t file.xz                    # xz 파일 무결성 검사
```

---

## 2. 🛠️ tar 아카이브

### 2-1. 순수 tar

```bash
tar -cvf archive.tar file1 file2 directory/   # 여러 파일과 디렉터리를 하나의 tar로 묶기
tar -tvf archive.tar                          # tar 내부 목록 확인
tar -xvf archive.tar                          # 현재 디렉터리에 tar 추출
tar -xvf archive.tar -C /restore              # /restore 디렉터리에 tar 추출
```

> 순수 `.tar`는 여러 파일을 하나로 묶은 아카이브이며, 별도의 압축 알고리즘을 적용하지 않은 상태이다.

### 2-2. gzip 결합

```bash
tar -czvf archive.tar.gz source/              # source 디렉터리를 tar로 묶고 gzip 압축
tar -tzvf archive.tar.gz                      # gzip 압축 tar의 내부 목록 확인
tar -xzvf archive.tar.gz                      # 현재 디렉터리에 압축 해제 및 추출
tar -xzvf archive.tar.gz -C /restore          # /restore에 압축 해제 및 추출
```

### 2-3. bzip2 결합

```bash
tar -cjvf archive.tar.bz2 source/             # source 디렉터리를 tar로 묶고 bzip2 압축
tar -tjvf archive.tar.bz2                     # bzip2 압축 tar의 내부 목록 확인
tar -xjvf archive.tar.bz2                     # 현재 디렉터리에 압축 해제 및 추출
tar -xjvf archive.tar.bz2 -C /restore         # /restore에 압축 해제 및 추출
```

### 2-4. xz 결합

```bash
tar -cJvf archive.tar.xz source/              # source 디렉터리를 tar로 묶고 xz 압축
tar -tJvf archive.tar.xz                      # xz 압축 tar의 내부 목록 확인
tar -xJvf archive.tar.xz                      # 현재 디렉터리에 압축 해제 및 추출
tar -xJvf archive.tar.xz -C /restore          # /restore에 압축 해제 및 추출
```

> xz 옵션은 대문자 `-J`이다. 소문자 `-j`는 bzip2를 의미한다.

### 2-5. 확장자 기반 생성

```bash
tar -cavf archive.tar.gz source/              # .tar.gz 확장자를 보고 gzip 자동 선택
tar -cavf archive.tar.bz2 source/             # .tar.bz2 확장자를 보고 bzip2 자동 선택
tar -cavf archive.tar.xz source/              # .tar.xz 확장자를 보고 xz 자동 선택
```

> `-a`는 생성할 아카이브 파일의 확장자를 기준으로 압축 방식을 선택한다.

### 2-6. 상대경로 백업

```bash
tar -czf backup.tar.gz -C /home guest         # /home으로 이동한 뒤 guest 디렉터리 백업
tar -czf content.tar.gz -C /home/guest .      # guest 이름 없이 내부 내용만 백업
```

첫 번째 명령의 아카이브 구조:

```text
guest/
├─ file1
└─ directory/
```

두 번째 명령의 아카이브 구조:

```text
./
├─ file1
└─ directory/
```

> `-C`를 사용하면 불필요한 절대경로를 피하고 원하는 상대경로 구조로 아카이브를 만들 수 있다.

---

## 3. 🔢 tar 옵션 조회표

| 옵션 | 의미 |
|---|---|
| `-c` | 새 아카이브 생성 |
| `-x` | 아카이브 추출 |
| `-t` | 아카이브 내부 목록 출력 |
| `-v` | 처리 중인 파일명 출력 |
| `-f` | 사용할 아카이브 파일 지정 |
| `-C` | 지정한 디렉터리로 이동한 후 작업 |
| `-z` | gzip 압축 또는 해제 |
| `-j` | bzip2 압축 또는 해제 |
| `-J` | xz 압축 또는 해제 |
| `-a` | 생성 시 확장자를 기준으로 압축 방식 선택 |
| `--exclude` | 일치하는 파일 또는 경로 제외 |
| `--exclude-from` | 파일에 작성된 제외 패턴 사용 |
| `--keep-old-files` | 기존 파일을 덮어쓰지 않고 오류 처리 |

> `-c`, `-x`, `-t`는 작업 종류를 지정하는 옵션이므로 일반적으로 한 명령에서 하나만 사용한다.

---

## 4. 🔢 압축 형식 조회표

| 형식 | 확장자 | 압축 | 해제 | 내용 출력·목록 |
|---|---|---|---|---|
| gzip | `.gz` | `gzip` | `gunzip` | `zcat` |
| bzip2 | `.bz2` | `bzip2` | `bunzip2` | `bzcat` |
| xz | `.xz` | `xz` | `unxz` | `xzcat` |
| tar | `.tar` | `tar -cf` | `tar -xf` | `tar -tf` |
| tar+gzip | `.tar.gz` | `tar -czf` | `tar -xzf` | `tar -tzf` |
| tar+bzip2 | `.tar.bz2` | `tar -cjf` | `tar -xjf` | `tar -tjf` |
| tar+xz | `.tar.xz` | `tar -cJf` | `tar -xJf` | `tar -tJf` |

---

## 5. 🔍 형식·무결성 검증

```bash
file <파일>                                 # 확장자가 아닌 실제 파일 형식 확인

gzip -t file.gz                            # gzip 압축 데이터 무결성 검사
bzip2 -t file.bz2                          # bzip2 압축 데이터 무결성 검사
xz -t file.xz                              # xz 압축 데이터 무결성 검사

tar -tvf archive.tar                       # 일반 tar 내부 목록 확인
tar -tzvf archive.tar.gz                   # gzip 압축 tar 내부 목록 확인

sha256sum archive.tar.gz                   # 파일의 SHA-256 체크섬 출력
sha256sum archive.tar.gz > archive.tar.gz.sha256  # 체크섬을 검증 파일로 저장
sha256sum -c archive.tar.gz.sha256         # 저장된 체크섬과 현재 파일 비교
```

검증 성공 예시:

```text
archive.tar.gz: OK
```

> `gzip -t`, `bzip2 -t`, `xz -t`는 압축 데이터의 구조적 무결성을 검사한다. 원본 파일과 정확히 같은 파일인지 확인하려면 별도로 생성한 체크섬과 비교해야 한다.

---

## 6. 🛠️ 제외 패턴

명령줄에서 제외 패턴 지정:

```bash
tar -czf backup.tar.gz \
  --exclude='*.tmp' \
  --exclude='cache/' \
  source/
# source에서 .tmp 파일과 cache 디렉터리를 제외하고 백업
```

제외 패턴 파일 사용:

```bash
tar -czf backup.tar.gz \
  --exclude-from=exclude.txt \
  source/
# exclude.txt에 작성된 패턴을 제외하고 백업
```

`exclude.txt` 예시:

```text
*.tmp
*.log
cache/
node_modules/
```

> 제외 패턴은 아카이브에 저장되는 상대경로와 일치해야 한다. 실제 적용 여부는 생성 후 `tar -tf`로 확인한다.

---

## 7. 🛠️ 안전한 추출

```bash
file archive.tar.gz                         # 실제 파일 형식 확인
tar -tzvf archive.tar.gz                    # 추출 전에 내부 경로와 파일 목록 확인

mkdir -p /tmp/archive-review                # 검토용 임시 디렉터리 생성
tar -xzvf archive.tar.gz \
  -C /tmp/archive-review                    # 임시 디렉터리에 gzip 압축 tar 추출

find /tmp/archive-review -maxdepth 3 -ls    # 추출된 파일을 3단계 깊이까지 확인
```

기존 파일 덮어쓰기 방지:

```bash
tar --keep-old-files \
  -xvf archive.tar \
  -C /restore
# /restore의 기존 파일은 덮어쓰지 않고 새 파일만 추출
```

> 신뢰할 수 없는 아카이브는 운영 디렉터리에 바로 추출하지 말고 별도의 임시 디렉터리에서 경로·권한·심볼릭 링크 등을 먼저 확인한다.

---

## 8. 🔍 공간·권한 확인

```bash
df -h <대상경로>                 # 대상 파일시스템의 남은 디스크 공간 확인
df -i <대상경로>                 # 대상 파일시스템의 남은 inode 확인

id                               # 현재 사용자의 UID·GID와 그룹 확인
namei -l <대상경로>              # 경로 구성 요소별 권한 확인
ls -ld <대상경로>                # 대상 디렉터리 자체의 권한 확인
findmnt -T <대상경로>            # 대상 경로가 속한 파일시스템과 옵션 확인
```

추출 전에는 다음 항목을 확인한다.

```text
디스크 여유 공간
inode 여유 공간
대상 디렉터리 쓰기 권한
읽기 전용 마운트 여부
기존 파일과 이름 충돌 여부
```

---

## 9. 📌 핵심 규칙

```text
단일 파일 압축        → gzip / bzip2 / xz
여러 파일 하나로 관리 → tar
빠른 압축             → gzip
높은 압축률 우선      → xz
원본 유지             → -k 또는 -c
gzip 결합             → tar -czf
bzip2 결합            → tar -cjf
xz 결합               → tar -cJf
형식 확인             → file
목록 확인             → tar -tf
무결성 검사           → gzip/bzip2/xz -t
체크섬 검증           → sha256sum -c
안전한 추출           → 목록 확인 후 별도 디렉터리에 -C
```

> 📌 **핵심 요약**
> - gzip `-z`, bzip2 `-j`, xz `-J`
> - 압축 도구는 기본적으로 원본을 압축 파일로 대체
> - 원본을 유지하려면 `-k` 또는 `-c` 사용
> - `tar`는 기본적으로 원본 파일을 유지
> - 확장자와 실제 형식은 `file`로 검증
> - 복원 전 `tar -t`로 내부 경로 확인
> - 압축 무결성은 `-t`, 파일 동일성은 체크섬으로 검증
> - 관련: 7-1. 🗜️ 파일 압축 & 아카이브 — gzip · bzip2 · xz · tar · 7-2. 🧩 파일 압축·아카이브 통합 정리 · 7-3. 🚑 파일 압축·아카이브 트러블슈팅 치트시트
