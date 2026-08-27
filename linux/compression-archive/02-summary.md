# 🧩 파일 압축·아카이브 통합 정리

> **Tag:** #Linux #Compression #Archive #tar #gzip #bzip2 #xz #Summary  
> **핵심 요약:** Linux의 압축·아카이브 작업은 `tar`로 여러 파일을 하나로 묶고 gzip·bzip2·xz로 크기를 줄이는 구조이다. 작업 전에는 원본 유지 여부와 대상 경로를 확인하고, 생성 후에는 `file`, 압축 무결성 검사, `tar -t`, 체크섬으로 결과를 검증한다.

---

## 1. 📖 개요 (Overview)

**압축과 아카이브를 관통하는 핵심 구조는?**

```text
여러 파일·디렉터리
→ tar로 하나의 아카이브 생성
→ gzip·bzip2·xz로 압축
→ .tar.gz / .tar.bz2 / .tar.xz
```

역방향:

```text
압축 아카이브
→ 압축 해제
→ tar 아카이브 추출
→ 원본 파일·디렉터리 복원
```

GNU tar에서는 두 단계를 한 명령으로 처리할 수 있다.

```bash
tar -czf backup.tar.gz source/
tar -cjf backup.tar.bz2 source/
tar -cJf backup.tar.xz source/
```

**tar와 압축 도구의 원본 처리 차이는?**

| 작업 | 기본 원본 처리 |
|---|---|
| `gzip file` | 원본을 `file.gz`로 대체 |
| `bzip2 file` | 원본을 `file.bz2`로 대체 |
| `xz file` | 원본을 `file.xz`로 대체 |
| `tar -cf a.tar source` | 원본 유지 |
| `tar -czf a.tar.gz source` | 원본 유지 |

원본 유지 옵션:

```bash
gzip -k file
bzip2 -k file
xz -k file
```

**압축 도구는 어떻게 선택하는가?**

| 목적 | 일반적인 선택 |
|---|---|
| 빠른 압축과 높은 호환성 | gzip |
| 기존 `.bz2` 환경과 호환 | bzip2 |
| 높은 압축률과 장기 보관 | xz |

압축 결과는 데이터 종류와 옵션에 따라 달라지므로 실제 파일로 측정한다.

```bash
time gzip <복사본>
time bzip2 <복사본>
time xz <복사본>
```

## 2. 🛠️ 표준 개념 정리 (Configuration)

### 2-1. 단일 파일 압축

gzip:

```bash
gzip -k file
gunzip file.gz
zcat file.gz
gzip -t file.gz
```

bzip2:

```bash
bzip2 -k file
bunzip2 file.bz2
bzcat file.bz2
bzip2 -t file.bz2
```

xz:

```bash
xz -k file
unxz file.xz
xzcat file.xz
xz -t file.xz
```

---

### 2-2. tar 핵심 옵션

| 옵션 | 의미 |
|---|---|
| `-c` | 생성 |
| `-x` | 추출 |
| `-t` | 목록 확인 |
| `-v` | 상세 출력 |
| `-f` | 아카이브 파일 지정 |
| `-C` | 기준 디렉터리 변경 |
| `-z` | gzip 결합 |
| `-j` | bzip2 결합 |
| `-J` | xz 결합 |
| `-a` | 생성 시 확장자 기반 압축 선택 |

---

### 2-3. 순수 tar

생성:

```bash
tar -cvf archive.tar file1 file2 directory/
```

목록:

```bash
tar -tvf archive.tar
```

추출:

```bash
tar -xvf archive.tar
```

다른 위치:

```bash
mkdir -p /restore
tar -xvf archive.tar -C /restore
```

---

### 2-4. 압축 결합 tar

gzip:

```bash
tar -czvf archive.tar.gz source/
tar -tzvf archive.tar.gz
tar -xzvf archive.tar.gz
```

bzip2:

```bash
tar -cjvf archive.tar.bz2 source/
tar -tjvf archive.tar.bz2
tar -xjvf archive.tar.bz2
```

xz:

```bash
tar -cJvf archive.tar.xz source/
tar -tJvf archive.tar.xz
tar -xJvf archive.tar.xz
```

확장자 기반 생성:

```bash
tar -cavf archive.tar.gz source/
tar -cavf archive.tar.bz2 source/
tar -cavf archive.tar.xz source/
```

---

### 2-5. 안전한 백업 템플릿

백업 디렉터리 생성:

```bash
mkdir -p /backup
```

상대경로로 아카이브:

```bash
tar -czf "/backup/guest-$(date +%F).tar.gz" \
  -C /home guest
```

실제 형식 확인:

```bash
file "/backup/guest-$(date +%F).tar.gz"
```

목록 확인:

```bash
tar -tzf "/backup/guest-$(date +%F).tar.gz" | head
```

체크섬 생성:

```bash
sha256sum "/backup/guest-$(date +%F).tar.gz" \
  > "/backup/guest-$(date +%F).tar.gz.sha256"
```

검증:

```bash
sha256sum -c "/backup/guest-$(date +%F).tar.gz.sha256"
```

복원 테스트:

```bash
mkdir -p /tmp/restore-test
tar -xzf "/backup/guest-$(date +%F).tar.gz" \
  -C /tmp/restore-test
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 표준 검증 순서

```text
1. ls -lh로 파일 존재와 크기 확인
2. file로 실제 형식 확인
3. gzip/bzip2/xz -t로 압축 무결성 확인
4. tar -t로 내부 목록 확인
5. sha256sum으로 전송·보관 무결성 확인
6. 격리된 디렉터리에서 복원 테스트
```

명령:

```bash
ls -lh archive*
file archive.tar.gz
gzip -t archive.tar.gz
tar -tzf archive.tar.gz
sha256sum archive.tar.gz
```

### 3-2. 대표 함정

| 함정 | 결과 | 올바른 접근 |
|---|---|---|
| 압축 후 원본이 사라짐 | 압축본으로 대체 | `-k` 사용 |
| 다중 파일을 gzip으로 처리 | `.gz` 여러 개 생성 | tar 결합 |
| 확장자와 실제 형식 불일치 | 해제 도구 혼동 | `file` 확인 |
| `tar -xf` 전에 바로 추출 | 예상하지 않은 경로 생성 | `tar -tf` 먼저 |
| `/`에서 아카이브 추출 | 기존 파일 덮어쓰기 위험 | 격리 디렉터리 사용 |
| 같은 디스크에만 백업 | 디스크 장애에 취약 | 다른 저장소 복제 |
| 압축 성공만 확인 | 복원 불가능 가능성 | 실제 복원 테스트 |

---

> 📌 **핵심 요약**
> - 단일 파일 압축: gzip·bzip2·xz
> - 다중 파일 관리: tar
> - gzip `-z`, bzip2 `-j`, xz `-J`
> - 원본 유지: 압축 도구 `-k`
> - 형식 확인: `file`
> - 목록 확인: `tar -t`
> - 무결성: 압축 도구 `-t` + `sha256sum`
> - 관련: 7-1. 🗜️ 파일 압축 & 아카이브 — gzip · bzip2 · xz · tar · 7-3. 🚑 파일 압축·아카이브 트러블슈팅 치트시트 · 7-4. ⚡ 파일 압축·아카이브 명령어 퀵 레퍼런스
