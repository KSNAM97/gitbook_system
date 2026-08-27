# 🗜️ 파일 압축 & 아카이브 — gzip · bzip2 · xz · tar

> **Tag:** #Linux #Compression #gzip #bzip2 #xz #tar #Archive #Backup  
> **핵심 요약:** Linux에서 `gzip`, `bzip2`, `xz`는 데이터를 압축하는 도구이고, `tar`는 여러 파일과 디렉터리를 하나의 아카이브로 묶는 도구이다. 단일 파일은 압축 도구로 처리하고, 여러 파일과 디렉터리는 `tar`로 묶은 뒤 gzip·bzip2·xz를 결합한다. 압축 도구는 기본적으로 입력 파일을 압축본으로 대체하지만 `-k` 또는 `-c`로 원본을 유지할 수 있으며, `tar`는 원본을 삭제하지 않는다.

---

## 1. 📖 개요 (Overview)

**압축과 아카이브의 차이는?**

압축과 아카이브는 목적이 다르다.

```text
압축(Compression)
→ 데이터 표현을 줄여 파일 크기를 감소
→ gzip, bzip2, xz

아카이브(Archive)
→ 여러 파일과 디렉터리를 하나의 파일로 묶음
→ tar
```

예를 들어 다음 세 파일을 `gzip`으로 처리하면:

```bash
gzip file1 file2 file3
```

하나의 압축 파일이 아니라 각각 별도의 압축 파일이 생성된다.

```text
file1.gz
file2.gz
file3.gz
```

세 파일을 하나로 관리하려면 먼저 `tar` 아카이브를 만든다.

```bash
tar -cvf files.tar file1 file2 file3
```

그 후 압축한다.

```bash
gzip files.tar
```

결과:

```text
files.tar.gz
```

두 작업을 한 명령으로 처리할 수도 있다.

```bash
tar -czvf files.tar.gz file1 file2 file3
```

**gzip·bzip2·xz는 어떤 기준으로 선택하는가?**

일반적인 경향은 다음과 같다.

| 도구 | 일반적인 속도 | 일반적인 압축률 | 확장자 | 대표 용도 |
|---|---|---|---|---|
| gzip | 빠름 | 보통 | `.gz` | 로그, 빠른 압축·전송 |
| bzip2 | gzip보다 느림 | gzip보다 높은 경우가 많음 | `.bz2` | 기존 백업·소스 배포 |
| xz | 압축 시 CPU·메모리 사용량이 큼 | 높은 경우가 많음 | `.xz` | 장기 보관, 배포 이미지 |

압축률은 원본 데이터의 종류와 압축 레벨에 따라 달라진다.

- 텍스트 파일은 비교적 잘 압축됨
- 이미지·동영상·ZIP처럼 이미 압축된 파일은 효과가 작음
- 작은 파일은 압축 헤더 때문에 결과가 더 커질 수 있음
- 모든 파일에서 `xz > bzip2 > gzip` 순서가 절대적으로 보장되지는 않음

실무 선택 기준:

```text
빠른 처리와 높은 호환성   → gzip
기존 bzip2 형식 호환 필요 → bzip2
압축률과 장기 보관 우선   → xz
```

> 최신 환경에서는 `zstd`도 널리 사용되지만, 이 문서는 gzip·bzip2·xz·tar를 중심으로 다룬다.

**압축하면 원본 파일이 사라지는 이유는?**

다음 명령은 기본적으로 입력 파일을 압축본으로 대체한다.

```bash
gzip file
bzip2 file
xz file
```

결과:

```text
file      → 삭제
file.gz   → 생성
```

원본을 유지하려면 `-k` 옵션을 사용한다.

```bash
gzip -k file
bzip2 -k file
xz -k file
```

결과:

```text
file
file.gz
```

압축 데이터를 표준 출력으로 보내고 리다이렉션하는 방법도 있다.

```bash
gzip -c file > file.gz
bzip2 -c file > file.bz2
xz -c file > file.xz
```

`tar`는 원본 파일을 아카이브에 복사하여 묶기 때문에 원본을 삭제하지 않는다.

```bash
tar -cvf backup.tar file1 file2 directory/
```

결과:

```text
backup.tar
file1
file2
directory/
```

**확장자는 실제 파일 형식을 결정하는가?**

확장자는 파일 형식을 사람이 쉽게 식별하기 위한 관례이다. 실제 압축 형식은 파일 내부의 형식 정보로 결정된다.

권장 확장자:

| 실제 형식 | 권장 확장자 |
|---|---|
| gzip 단일 파일 | `.gz` |
| bzip2 단일 파일 | `.bz2` |
| xz 단일 파일 | `.xz` |
| 순수 tar | `.tar` |
| tar + gzip | `.tar.gz`, `.tgz` |
| tar + bzip2 | `.tar.bz2`, `.tbz2` |
| tar + xz | `.tar.xz`, `.txz` |

파일명이 잘못되어도 실제 형식은 `file`로 확인할 수 있다.

```bash
file archive
```

예시:

```text
archive: XZ compressed data
archive: bzip2 compressed data
archive: POSIX tar archive
```

**압축 파일과 백업 파일은 같은가?**

압축은 저장 공간을 줄이는 기능일 뿐, 그 자체로 완전한 백업을 의미하지 않는다.

안전한 백업에는 다음 요소도 필요하다.

- 원본과 다른 디스크 또는 호스트에 보관
- 정기적인 무결성 검사
- 복원 테스트
- 보관 기간과 세대 관리
- 접근 권한과 암호화
- 체크섬 또는 서명
- 랜섬웨어와 운영자 실수로부터 격리

```text
같은 디스크에 backup.tar.gz 하나 생성
→ 디스크 장애에는 안전하지 않음
```

## 2. 🛠️ 표준 사용 템플릿 (Configuration)

### 2-1. gzip — 빠르고 호환성이 높은 압축

압축:

```bash
gzip file
```

결과:

```text
file.gz
```

압축 해제:

```bash
gzip -d file.gz
```

또는:

```bash
gunzip file.gz
```

원본 유지:

```bash
gzip -k file
```

표준 출력 사용:

```bash
gzip -c file > file.gz
```

압축본을 유지하며 해제 결과만 다른 파일로 저장:

```bash
gzip -dc file.gz > file
```

또는:

```bash
zcat file.gz > file
```

압축 레벨:

```bash
gzip -1 file     # 빠른 압축
gzip -9 file     # 높은 압축률 우선
```

기본 레벨은 일반적으로 속도와 압축률의 균형값을 사용한다.

압축 정보:

```bash
gzip -l file.gz
```

무결성 검사:

```bash
gzip -t file.gz
```

정상이면 일반적으로 출력 없이 종료 코드 `0`을 반환한다.

```bash
gzip -t file.gz && echo "gzip OK"
```

---

### 2-2. bzip2 — 비교적 높은 압축률

압축:

```bash
bzip2 file
```

결과:

```text
file.bz2
```

압축 해제:

```bash
bzip2 -d file.bz2
```

또는:

```bash
bunzip2 file.bz2
```

원본 유지:

```bash
bzip2 -k file
```

표준 출력 사용:

```bash
bzip2 -c file > file.bz2
```

압축본을 유지하며 해제:

```bash
bzip2 -dc file.bz2 > file
```

내용 출력:

```bash
bzcat file.bz2
```

압축 레벨:

```bash
bzip2 -1 file
bzip2 -9 file
```

무결성 검사:

```bash
bzip2 -t file.bz2
```

검사 결과 확인:

```bash
bzip2 -t file.bz2 && echo "bzip2 OK"
```

---

### 2-3. xz — 높은 압축률을 목표로 하는 압축

압축:

```bash
xz file
```

결과:

```text
file.xz
```

압축 해제:

```bash
xz -d file.xz
```

또는:

```bash
unxz file.xz
```

원본 유지:

```bash
xz -k file
```

표준 출력 사용:

```bash
xz -c file > file.xz
```

압축본을 유지하며 해제:

```bash
xz -dc file.xz > file
```

내용 출력:

```bash
xzcat file.xz
```

압축 레벨:

```bash
xz -1 file
xz -9 file
```

높은 압축 레벨은 CPU 시간과 메모리 사용량을 크게 증가시킬 수 있다.

압축 정보:

```bash
xz -l file.xz
```

무결성 검사:

```bash
xz -t file.xz
```

검사 결과 확인:

```bash
xz -t file.xz && echo "xz OK"
```

---

### 2-4. 단일 파일 압축 실습

실습 디렉터리 준비:

```bash
mkdir -p /home/guest/compress-lab
cp /etc/services /home/guest/compress-lab/
cp /etc/mime.types /home/guest/compress-lab/
cp /etc/brltty.conf /home/guest/compress-lab/
```

확인:

```bash
ls -lSh /home/guest/compress-lab
```

gzip:

```bash
gzip -k /home/guest/compress-lab/services
```

bzip2:

```bash
bzip2 -k /home/guest/compress-lab/mime.types
```

xz:

```bash
xz -k /home/guest/compress-lab/brltty.conf
```

결과 확인:

```bash
ls -lSh /home/guest/compress-lab
file /home/guest/compress-lab/*
```

> 원본 크기와 데이터 특성이 다르므로 세 결과만 보고 압축 도구의 절대적인 우열을 판단하지 않는다. 정확한 비교는 동일한 원본 복사본에 같은 조건으로 각각 압축해야 한다.

---

### 2-5. 동일 파일 압축률 비교

동일한 원본으로 복사본 생성:

```bash
cd /home/guest/compress-lab

cp services services.gzip-test
cp services services.bzip2-test
cp services services.xz-test
```

압축:

```bash
gzip  services.gzip-test
bzip2 services.bzip2-test
xz    services.xz-test
```

결과 비교:

```bash
ls -lSh services*
```

압축률과 시간을 함께 비교하려면:

```bash
cp /etc/services gzip-source
cp /etc/services bzip2-source
cp /etc/services xz-source

time gzip gzip-source
time bzip2 bzip2-source
time xz xz-source
```

> 동일한 원본과 같은 시스템 조건에서 비교해야 의미가 있다.

---

### 2-6. tar의 기본 역할

`tar`는 Tape Archive의 약자로 여러 파일과 디렉터리를 하나의 아카이브로 묶는다.

기본 옵션:

| 옵션 | 의미 |
|---|---|
| `-c` | 새 아카이브 생성 |
| `-x` | 아카이브 추출 |
| `-t` | 내용 목록 출력 |
| `-v` | 처리 대상 상세 출력 |
| `-f` | 다음 인수를 아카이브 파일명으로 사용 |
| `-C` | 작업 또는 추출 기준 디렉터리 변경 |

아카이브 생성:

```bash
tar -cvf archive.tar file1 file2 directory/
```

내용 확인:

```bash
tar -tvf archive.tar
```

추출:

```bash
tar -xvf archive.tar
```

지정 디렉터리에 추출:

```bash
mkdir -p /restore
tar -xvf archive.tar -C /restore
```

> `-C`로 지정하는 디렉터리는 먼저 생성되어 있어야 한다.

---

### 2-7. `-f` 옵션의 정확한 의미

다음 명령에서:

```bash
tar -cvf backup.tar file1 file2
```

`-f`는 다음 인수인 `backup.tar`를 아카이브 파일명으로 사용한다.

다음 형식도 명확하다.

```bash
tar --create --verbose --file=backup.tar file1 file2
```

짧은 옵션을 묶을 때 `f`를 마지막에 두는 것은 널리 사용하는 관례이지만, 핵심은 `-f`가 아카이브 파일명 인수를 요구한다는 점이다.

권장:

```bash
tar -cvf backup.tar source/
tar -xvf backup.tar
tar -tvf backup.tar
```

---

### 2-8. 순수 tar 아카이브 생성

실습 파일:

```bash
cd /home/guest

tar -cvf "$(date +%Y%m%d).tar" \
  services \
  dnsmasq.conf \
  ld.so.cache
```

확인:

```bash
ls -lh "$(date +%Y%m%d).tar"
file "$(date +%Y%m%d).tar"
```

목록 확인:

```bash
tar -tvf "$(date +%Y%m%d).tar"
```

원본이 유지되는지 확인:

```bash
ls -l services dnsmasq.conf ld.so.cache
```

---

### 2-9. tar + gzip

생성:

```bash
tar -czvf backup.tar.gz file1 file2 directory/
```

옵션:

```text
c → 생성
z → gzip 압축
v → 진행 출력
f → 아카이브 파일명
```

목록 확인:

```bash
tar -tzvf backup.tar.gz
```

추출:

```bash
tar -xzvf backup.tar.gz
```

지정 위치에 추출:

```bash
mkdir -p /restore/gzip
tar -xzvf backup.tar.gz -C /restore/gzip
```

---

### 2-10. tar + bzip2

생성:

```bash
tar -cjvf backup.tar.bz2 file1 file2 directory/
```

목록 확인:

```bash
tar -tjvf backup.tar.bz2
```

추출:

```bash
tar -xjvf backup.tar.bz2
```

지정 위치에 추출:

```bash
mkdir -p /restore/bzip2
tar -xjvf backup.tar.bz2 -C /restore/bzip2
```

---

### 2-11. tar + xz

생성:

```bash
tar -cJvf backup.tar.xz file1 file2 directory/
```

옵션:

```text
-J → xz 압축
```

목록 확인:

```bash
tar -tJvf backup.tar.xz
```

추출:

```bash
tar -xJvf backup.tar.xz
```

지정 위치에 추출:

```bash
mkdir -p /restore/xz
tar -xJvf backup.tar.xz -C /restore/xz
```

---

### 2-12. 확장자 기반 자동 압축 선택

GNU tar의 `-a` 또는 `--auto-compress`는 아카이브를 생성할 때 파일 확장자를 보고 압축 방식을 선택할 수 있다.

```bash
tar -cavf backup.tar.gz source/
tar -cavf backup.tar.bz2 source/
tar -cavf backup.tar.xz source/
```

확장자에 따라 gzip, bzip2, xz가 선택된다.

> `--auto-compress`는 특히 생성 시 확장자에 맞는 압축기를 선택하는 옵션이다. GNU tar는 일반 파일에서 압축 형식을 자동으로 인식해 `tar -xf`만으로 추출할 수 있는 경우가 많지만, 이 동작은 tar 구현과 입력 방식에 따라 다를 수 있다.

명시적인 방식:

```bash
tar -xzf backup.tar.gz
tar -xjf backup.tar.bz2
tar -xJf backup.tar.xz
```

GNU tar에서 자동 인식:

```bash
tar -xf backup.tar.gz
tar -xf backup.tar.bz2
tar -xf backup.tar.xz
```

---

### 2-13. 절대경로 대신 상대경로로 아카이브하기

다음 명령은 절대경로를 전달한다.

```bash
tar -czf home-guest.tar.gz /home/guest
```

GNU tar는 일반적으로 선행 `/`를 제거하며 경고를 표시할 수 있다.

더 명확한 방법:

```bash
tar -czf home-guest.tar.gz -C /home guest
```

아카이브 내부 경로:

```text
guest/
guest/file1
guest/file2
```

특정 디렉터리의 내용만 저장:

```bash
tar -czf guest-content.tar.gz -C /home/guest .
```

---

### 2-14. 제외 패턴 사용

캐시와 임시 파일 제외:

```bash
tar -czf guest-backup.tar.gz \
  --exclude='*.tmp' \
  --exclude='cache/' \
  -C /home guest
```

여러 제외 패턴 파일 사용:

```bash
cat > exclude.txt <<'EOF'
*.tmp
*.log
cache/
EOF

tar -czf backup.tar.gz \
  --exclude-from=exclude.txt \
  -C /home guest
```

---

### 2-15. 아카이브를 두 단계로 해제

`.tar.bz2`를 먼저 bzip2 해제:

```bash
bunzip2 backup.tar.bz2
```

결과:

```text
backup.tar
```

tar 추출:

```bash
tar -xvf backup.tar
```

한 번에 처리:

```bash
tar -xjvf backup.tar.bz2
```

`.tar.gz`:

```bash
gunzip backup.tar.gz
tar -xvf backup.tar
```

한 번에:

```bash
tar -xzvf backup.tar.gz
```

`.tar.xz`:

```bash
unxz backup.tar.xz
tar -xvf backup.tar
```

한 번에:

```bash
tar -xJvf backup.tar.xz
```

---

### 2-16. 공유 추출 디렉터리 구성

모든 사용자가 파일을 생성할 수 있지만 다른 사용자의 파일 삭제를 제한하는 공개 임시 공간:

```bash
mkdir -p /home/guest/temp
chown root:root /home/guest/temp
chmod 1777 /home/guest/temp
```

확인:

```bash
ls -ld /home/guest/temp
```

예상:

```text
drwxrwxrwt root root /home/guest/temp
```

팀 전용 공유 공간:

```bash
chown root:archive /srv/archive
chmod 2770 /srv/archive
```

그룹 상속과 타인 파일 삭제 방지를 함께 적용:

```bash
chmod 3770 /srv/archive
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 압축 파일 실제 형식 확인

```bash
file <파일>
```

예시:

```bash
file backup.tar
file backup.tar.gz
file backup.tar.bz2
file backup.tar.xz
```

확장자가 잘못되어도 실제 형식을 확인할 수 있다.

---

### 3-2. 압축 파일 무결성 검사

```bash
gzip -t file.gz
bzip2 -t file.bz2
xz -t file.xz
```

성공 여부 출력:

```bash
gzip -t file.gz && echo "OK" || echo "FAIL"
```

> 무결성 검사는 압축 스트림이 손상되지 않았는지 확인한다. 아카이브 내부 내용의 안전성이나 신뢰성까지 보장하지는 않는다.

---

### 3-3. 압축을 풀지 않고 내용 확인

단일 텍스트 파일:

```bash
zcat file.gz
bzcat file.bz2
xzcat file.xz
```

페이지 단위:

```bash
zcat file.gz | less
bzcat file.bz2 | less
xzcat file.xz | less
```

tar 목록:

```bash
tar -tvf archive.tar
tar -tzvf archive.tar.gz
tar -tjvf archive.tar.bz2
tar -tJvf archive.tar.xz
```

GNU tar 자동 인식:

```bash
tar -tvf archive.tar.gz
```

---

### 3-4. 안전한 추출 절차

실제 형식 확인:

```bash
file archive.tar.gz
```

목록 먼저 확인:

```bash
tar -tvf archive.tar.gz
```

격리된 대상 디렉터리 생성:

```bash
mkdir -p /tmp/archive-review
```

추출:

```bash
tar -xvf archive.tar.gz -C /tmp/archive-review
```

결과 확인:

```bash
find /tmp/archive-review -maxdepth 3 -ls
```

> 출처를 신뢰할 수 없는 아카이브는 운영 경로나 `/`에서 직접 추출하지 않는다. 악의적인 경로, 심볼릭 링크, 기존 파일 덮어쓰기 위험이 있을 수 있으므로 격리된 환경에서 검토한다.

---

### 3-5. 체크섬 생성과 검증

생성:

```bash
sha256sum backup.tar.gz > backup.tar.gz.sha256
```

검증:

```bash
sha256sum -c backup.tar.gz.sha256
```

예상:

```text
backup.tar.gz: OK
```

---

### 3-6. 대표 트러블슈팅

#### 🚨 시나리오 1. 압축 후 원본이 사라졌다

원인:

```text
gzip, bzip2, xz는 기본적으로 입력 파일을 압축본으로 대체
```

복원:

```bash
gunzip file.gz
bunzip2 file.bz2
unxz file.xz
```

앞으로 원본 유지:

```bash
gzip -k file
bzip2 -k file
xz -k file
```

---

#### 🚨 시나리오 2. `.tar`인데 `tar -xvf`가 실패한다

확인:

```bash
file AWS_SOL.tar
```

실제 결과가 bzip2라면:

```text
bzip2 compressed data
```

명시적으로 추출:

```bash
tar -xjvf AWS_SOL.tar
```

권장 파일명으로 변경:

```bash
mv AWS_SOL.tar AWS_SOL.tar.bz2
```

---

#### 🚨 시나리오 3. `not in gzip format` 오류가 발생한다

원인:

- 확장자는 `.gz`지만 실제 형식이 다름
- 파일이 손상됨
- HTML 오류 페이지 등을 압축 파일로 잘못 저장함

확인:

```bash
file file.gz
gzip -t file.gz
head file.gz
```

실제 형식에 맞는 도구를 사용한다.

---

#### 🚨 시나리오 4. 다중 파일을 압축했더니 압축 파일이 여러 개 생겼다

실행:

```bash
gzip file1 file2 file3
```

결과:

```text
file1.gz
file2.gz
file3.gz
```

하나로 만들려면:

```bash
tar -czvf files.tar.gz file1 file2 file3
```

---

#### 🚨 시나리오 5. 원하는 위치가 아닌 현재 디렉터리에 풀렸다

추출 전에 대상 디렉터리를 생성한다.

```bash
mkdir -p /restore
tar -xvf archive.tar -C /restore
```

---

#### 🚨 시나리오 6. 추출 중 기존 파일이 덮어써질까 걱정된다

먼저 목록을 확인한다.

```bash
tar -tvf archive.tar
```

격리 디렉터리에 추출한다.

```bash
mkdir -p /tmp/archive-test
tar -xvf archive.tar -C /tmp/archive-test
```

기존 파일을 덮어쓰지 않도록 GNU tar 옵션을 사용할 수 있다.

```bash
tar --keep-old-files -xvf archive.tar -C /restore
```

---

#### 🚨 시나리오 7. 압축본이 원본보다 크다

작은 파일이거나 이미 압축된 파일일 수 있다.

```bash
file original
ls -lh original original.gz
```

이미 압축된 데이터 예:

```text
JPEG
PNG
MP4
ZIP
GZIP
XZ
```

이 경우 추가 압축 효과가 작거나 결과가 더 커질 수 있다.

---

#### 🚨 시나리오 8. 백업 중 파일이 변경됐다는 경고가 발생한다

예시:

```text
tar: file changed as we read it
```

아카이브 생성 중 원본이 변경된 상태다.

대응:

- 서비스를 일시 중지할 수 있는지 검토
- 애플리케이션 전용 백업 기능 사용
- LVM/ZFS/Btrfs 스냅샷 사용
- 데이터베이스는 논리 백업 도구 사용
- 생성된 아카이브를 검증하고 필요하면 다시 백업

---

> 📌 **핵심 요약**
> - gzip·bzip2·xz는 압축, tar는 아카이브
> - gzip·bzip2·xz는 기본적으로 원본을 압축본으로 대체
> - 원본 유지: `-k` 또는 `-c`
> - tar는 원본을 유지
> - gzip 결합: `-z`, bzip2 결합: `-j`, xz 결합: `-J`
> - 실제 형식 확인: `file`
> - 무결성 검사: `gzip -t`, `bzip2 -t`, `xz -t`
> - 추출 전 `tar -tvf`, 격리 디렉터리에 `-C`로 추출
> - 관련: 7-2. 🧩 파일 압축·아카이브 통합 정리 · 7-3. 🚑 파일 압축·아카이브 트러블슈팅 치트시트 · 7-4. ⚡ 파일 압축·아카이브 명령어 퀵 레퍼런스
