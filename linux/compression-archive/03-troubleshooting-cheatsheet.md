# 파일 압축·아카이브 트러블슈팅 치트시트

> **Tag:** #Linux #Compression #Archive #tar #gzip #bzip2 #xz #Troubleshooting #CheatSheet  
> **핵심 요약:** 압축·아카이브 장애는 파일의 실제 형식, 원본 처리 상태, 압축 스트림 무결성, tar 내부 경로, 대상 디렉터리 권한과 디스크 공간 순서로 확인한다. 확장자만 믿지 말고 먼저 `file`과 `tar -t`로 검사한다.

---

## 1. 개요 (Overview)

**가장 먼저 확인할 것은?**

1. 파일 존재 여부와 크기
2. 파일의 실제 형식
3. 압축 스트림 무결성
4. tar 내부 목록
5. 추출 대상 디렉터리
6. 디스크 공간과 inode
7. 권한과 소유권
8. 체크섬

```bash
ls -lh <파일>
file <파일>
tar -tvf <파일>
df -h <대상경로>
df -i <대상경로>
```

**확장자와 실제 형식이 다른 경우는 어떻게 확인하는가?**

```bash
file AWS_SOL.tar
```

예상 가능한 결과:

```text
POSIX tar archive
gzip compressed data
bzip2 compressed data
XZ compressed data
HTML document
ASCII text
```

## 2. 증상별 즉시 대응표 (Configuration)

### 1. gzip·bzip2·xz

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| 원본이 사라짐 | 기본 대체 동작 | 해제하거나 다음부터 `-k` |
| 압축본이 더 큼 | 작은 파일·이미 압축된 데이터 | 원본 유지, 압축 생략 검토 |
| `not in gzip format` | 실제 형식 불일치 | `file` 확인 |
| `unexpected end of file` | 전송 중 손상·잘림 | 체크섬 및 재전송 |
| 압축이 너무 느림 | 높은 레벨·CPU 부족 | 낮은 레벨 또는 gzip |
| 해제 후 압축본도 필요 | 기본적으로 압축본 제거 | `-k` 또는 `-dc` |

### 2. tar

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| `not a tar archive` | 압축 미해제·형식 불일치 | `file`, 올바른 압축 옵션 |
| 원하는 위치에 안 풀림 | `-C` 누락 | 대상 생성 후 `-C` |
| 기존 파일이 덮어써짐 | 현재 경로에 직접 추출 | 격리 디렉터리 사용 |
| 특정 파일이 없음 | 제외 패턴·경로 오해 | `tar -tvf` 확인 |
| 권한이 다르게 복원됨 | 실행 사용자·umask·옵션 | 소유권·권한 검증 |
| 추출 중 공간 부족 | 압축 크기만 보고 계산 | `df -h`, 원본 크기 확인 |
| 백업 중 변경 경고 | 생성 중 원본 수정 | 스냅샷·서비스별 백업 |

---

## 3. 트러블슈팅 시나리오 (Verification & Troubleshooting)

### 시나리오 1. 압축 후 원본 파일이 사라졌다

실행:

```bash
gzip services
```

결과:

```text
services 삭제
services.gz 생성
```

복원:

```bash
gunzip services.gz
```

앞으로 원본 유지:

```bash
gzip -k services
```

bzip2와 xz:

```bash
bzip2 -k file
xz -k file
```

표준 출력 방식:

```bash
gzip -c services > services.gz
```

---

### 시나리오 2. `.tar` 파일이 풀리지 않는다

오류:

```text
tar: This does not look like a tar archive
```

실제 형식 확인:

```bash
file AWS_SOL.tar
```

bzip2 형식이면:

```bash
tar -xjvf AWS_SOL.tar
```

gzip 형식이면:

```bash
tar -xzvf AWS_SOL.tar
```

xz 형식이면:

```bash
tar -xJvf AWS_SOL.tar
```

파일명 정리:

```bash
mv AWS_SOL.tar AWS_SOL.tar.bz2
```

---

### 시나리오 3. `gzip: not in gzip format`

확인:

```bash
file download.gz
```

파일이 HTML이라면 다운로드 과정에서 오류 페이지를 받은 것일 수 있다.

```bash
head -n 20 download.gz
```

체크섬이 제공된 경우:

```bash
sha256sum download.gz
```

정상 파일을 다시 내려받는다.

---

### 시나리오 4. 다중 파일 압축 결과가 여러 개 생겼다

```bash
gzip file1 file2 file3
```

결과:

```text
file1.gz
file2.gz
file3.gz
```

하나의 아카이브로 생성:

```bash
tar -czvf files.tar.gz file1 file2 file3
```

---

### 시나리오 5. 지정 위치에 추출되지 않았다

잘못된 예:

```bash
tar -xvf archive.tar
```

현재 작업 디렉터리에 추출된다.

대상 디렉터리 생성:

```bash
mkdir -p /restore
```

지정 추출:

```bash
tar -xvf archive.tar -C /restore
```

---

### 시나리오 6. 아카이브 내부 구조를 모른 채 추출했다

목록 확인:

```bash
tar -tvf archive.tar.gz
```

격리 디렉터리 생성:

```bash
mkdir -p /tmp/archive-review
```

추출:

```bash
tar -xvf archive.tar.gz -C /tmp/archive-review
```

확인:

```bash
find /tmp/archive-review -maxdepth 3 -ls
```

> 출처를 신뢰할 수 없는 아카이브는 root 권한으로 운영 경로에 직접 추출하지 않는다.

---

### 시나리오 7. 추출 중 `No space left on device`

공간 확인:

```bash
df -h <대상경로>
df -i <대상경로>
```

압축 파일 크기는 해제 후 필요한 공간과 다르다.

gzip 정보:

```bash
gzip -l file.gz
```

xz 정보:

```bash
xz -l file.xz
```

tar 내부 크기 확인:

```bash
tar -tvf archive.tar.gz
```

필요하면 더 넓은 파일시스템으로 대상을 변경한다.

---

### 시나리오 8. `Permission denied`가 발생한다

대상 디렉터리 확인:

```bash
id
namei -l /restore/path
ls -ld /restore/path
```

아카이브 파일 읽기 권한:

```bash
ls -l archive.tar.gz
```

SELinux와 마운트 상태:

```bash
ls -ldZ /restore/path
findmnt -T /restore/path
```

---

### 시나리오 9. 압축 파일이 손상되었다

검사:

```bash
gzip -t file.gz
bzip2 -t file.bz2
xz -t file.xz
```

체크섬 비교:

```bash
sha256sum -c file.sha256
```

손상된 파일은 가능하면 원본 또는 다른 백업에서 다시 복사한다.

---

### 시나리오 10. 추출 시 기존 파일을 덮어쓰고 싶지 않다

먼저 목록 확인:

```bash
tar -tvf archive.tar
```

격리된 경로 사용:

```bash
mkdir -p /tmp/restore-test
tar -xvf archive.tar -C /tmp/restore-test
```

GNU tar에서 기존 파일 보존:

```bash
tar --keep-old-files -xvf archive.tar -C /restore
```

기존 파일이 있으면 오류를 발생시키고 덮어쓰지 않는다.

---

### 시나리오 11. 백업 중 `file changed as we read it`

원본 파일이 tar 처리 중 변경되었다.

대응:

- 애플리케이션 쓰기 중지
- 로그 회전 후 백업
- 데이터베이스 전용 덤프 도구 사용
- LVM·ZFS·Btrfs 스냅샷 사용
- 백업 결과 폐기 후 다시 생성
- 파일 목록과 복원 결과 검증

---

### 시나리오 12. 공유 폴더의 아카이브를 다른 사용자가 삭제했다

공개 임시 공간:

```bash
chown root:root /shared
chmod 1777 /shared
```

팀 전용:

```bash
chown root:backup /shared
chmod 2770 /shared
```

팀 전용이며 타인 파일 삭제 방지:

```bash
chmod 3770 /shared
```

> `1770`도 Sticky-bit는 적용하지만 그룹 상속이 필요하면 `3770`을 검토한다.

---

## 4. 긴급 점검 명령 모음

```bash
ls -lh <파일>
file <파일>

gzip -t <파일>.gz
bzip2 -t <파일>.bz2
xz -t <파일>.xz

tar -tvf <아카이브>
sha256sum <아카이브>

df -h <대상경로>
df -i <대상경로>

id
namei -l <대상경로>
findmnt -T <대상경로>
```

>  **핵심 요약**
> - 확장자보다 `file` 결과를 신뢰
> - 압축 후 원본이 없으면 기본 대체 동작 확인
> - 압축 손상은 `-t`와 체크섬으로 검증
> - 추출 전 `tar -t`
> - 추출은 격리 디렉터리와 `-C`
> - 공간 부족은 `df -h`, inode 부족은 `df -i`
> - 관련: 7-1.  파일 압축 & 아카이브 — gzip · bzip2 · xz · tar · 7-2.  파일 압축·아카이브 통합 정리 · 7-4.  파일 압축·아카이브 명령어 퀵 레퍼런스
