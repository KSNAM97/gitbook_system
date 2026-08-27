# 파일시스템 기본 명령어 퀵 레퍼런스

파일시스템 기본 명령어의 문법과 자주 사용하는 옵션을 빠르게 조회하는 문서.

## 목차

1. [명령어 문법 (Configuration)](#명령어-문법-configuration)
2. [빠른 조회표 (Configuration)](#빠른-조회표-configuration)
3. [검증 명령어 모음 (Verification)](#검증-명령어-모음-verification)
4. [요약](#요약)

---

## 명령어 문법 (Configuration)

### 1-1. 경로 이동 및 확인

```bash
pwd                              # 현재 논리적 경로 확인
pwd -P                           # 심볼릭 링크를 해석한 물리적 경로 확인
cd /etc/ssh                      # 절대경로로 이동
cd ..                            # 상위 디렉터리로 이동
cd ../..                         # 두 단계 위로 이동
cd                               # 현재 사용자의 홈으로 이동
cd ~                             # 현재 사용자의 홈으로 이동
cd ~guest                        # guest 사용자의 홈으로 이동
cd -                             # 직전 디렉터리로 이동
readlink -f <경로>               # 정규화된 절대경로 확인
```

### 1-2. `ls`

```bash
ls                               # 이름 목록 출력
ls -l                            # 상세 정보 출력
ls -a                            # 숨김 파일 포함
ls -h                            # 크기를 읽기 쉬운 단위로 표시
ls -R                            # 하위 디렉터리를 재귀 조회
ls -t                            # 수정 시간 최신순 정렬
ls -S                            # 크기 내림차순 정렬
ls -r                            # 정렬 결과 역순
ls -s                            # 할당된 블록 수 표시
ls -alh                          # 숨김 포함 상세·가독성 단위 출력
ls -lt | head                    # 최근 수정 항목 일부 확인
ls -lS | head                    # 큰 항목 일부 확인
ls -ld /etc                      # /etc 내부가 아닌 디렉터리 자체 확인
ls -ld . ..                      # 현재·상위 디렉터리 자체 확인
```

### 1-3. `mkdir`, `rmdir`, `rm`

```bash
mkdir /a                         # 단일 디렉터리 생성
mkdir /a /b /c                   # 여러 디렉터리 생성
mkdir -p /a/b/c                  # 중간 경로를 포함해 생성

rmdir /empty                     # 빈 디렉터리 삭제
rmdir -p /a/b/c                  # 비어 있는 상위 경로까지 삭제

rm file                          # 파일 삭제
rm -i file                       # 파일마다 삭제 전 확인
rm -I file...                    # 대량 삭제 전 한 번 확인
rm -f file                       # 존재하지 않는 파일 오류 생략
rm -r directory                  # 디렉터리 재귀 삭제
rm -rf -- directory              # 확인 없이 강제 재귀 삭제
```

> `rm -rf` 실행 전에는 대상 경로를 `ls -ld`, `readlink -f` 등으로 확인한다.

### 1-4. `cp`

```bash
cp source destination            # 파일 기본 복사
cp -p source destination         # 권한·소유권·시간 보존 시도
cp -R directory destination/     # 디렉터리 재귀 복사
cp -a source/. destination/      # 숨김·링크·메타데이터를 포함해 내용 복사
cp -i source destination         # 덮어쓰기 전 확인
cp -f source destination         # 강제 처리 시도
cp -n source destination         # 기존 파일 덮어쓰기 금지
cp -u source destination         # 원본이 더 최신일 때만 복사
cp f1 f2 f3 /destination/        # 여러 파일을 지정 디렉터리에 복사
```

### 1-5. `mv`

```bash
mv old new                       # 파일 또는 디렉터리 이름 변경
mv source /destination/          # 지정 디렉터리로 이동
mv -t /destination/ source...    # 목적지를 디렉터리로 명확하게 지정
mv -i source destination         # 덮어쓰기 전 확인
mv -f source destination         # 강제 처리 시도
mv -n source destination         # 기존 파일 덮어쓰기 금지
mv -u source destination         # 원본이 더 최신일 때 처리
```

### 1-6. 와일드카드

```bash
printf '%s\n' /etc/a*            # a로 시작하는 경로 확인
printf '%s\n' /etc/*.conf        # .conf로 끝나는 경로 확인
printf '%s\n' /etc/*log*         # 이름에 log가 포함된 경로 확인
printf '%s\n' /etc/s???          # s 뒤에 정확히 3문자가 있는 이름
printf '%s\n' /etc/??????.conf   # 이름 6문자와 .conf로 구성된 항목
printf '%s\n' /etc/s??l          # s로 시작하고 l로 끝나는 4문자 이름
```

---

## 빠른 조회표 (Configuration)

### 2-1. `ls -l` 필드

```text
-rw-r--r--. 1 root root 2124  7월 2 19:36 passwd
①            ② ③    ④    ⑤          ⑥

① 파일 종류와 권한
② 하드링크 수
③ 소유자
④ 소유 그룹
⑤ 크기 및 최종 수정 시간
⑥ 파일명
```

### 2-2. 덮어쓰기 옵션

| 옵션 | 의미 |
|---|---|
| `-i` | 덮어쓰기 전 확인 |
| `-f` | 강제 처리 시도 |
| `-n` | 기존 파일을 덮어쓰지 않음 |
| `-u` | 원본이 목적지보다 최신일 때 처리 |

### 2-3. 복사 옵션

| 옵션 | 설명 |
|---|---|
| 없음 | 파일 내용 복사, 메타데이터 완전 보존 안 됨 |
| `-p` | 권한·소유권·시간 보존 시도 |
| `-R` | 디렉터리 재귀 복사 |
| `-a` | 아카이브 모드, `-dR --preserve=all`에 해당 |

### 2-4. 와일드카드

| 패턴 | 의미 |
|---|---|
| `*` | 0개 이상의 문자 |
| `?` | 정확히 한 문자 |
| `s???` | `s` 뒤에 3문자 |
| `*.conf` | `.conf`로 끝 |
| `*log*` | `log` 포함 |
| `.` 시작 파일 | 기본 `*`에 포함되지 않음 |

---

## 검증 명령어 모음 (Verification)

```bash
pwd                              # 현재 논리적 작업 경로 확인
pwd -P                           # 물리적 작업 경로 확인
ls -alh <경로>                   # 숨김 항목을 포함한 상세 목록 확인
ls -ld <경로>                    # 경로 자체 정보 확인
printf '%s\n' <패턴>             # 와일드카드 확장 결과 검증
readlink -f <경로>               # 정규화된 절대경로 확인
namei -l <경로>                  # 경로 구성 요소별 권한 확인
stat <파일>                      # 파일 권한·소유권·시간·inode 확인
diff -- <원본> <사본> && echo "OK" # 내용이 같으면 OK 출력
sha256sum <원본> <사본>          # 두 파일의 체크섬 비교
find / -xtype l 2>/dev/null      # 깨진 심볼릭 링크 검색
du -sh <디렉터리>                # 디렉터리 전체 사용량 요약
type cp                          # 실제 실행될 cp·alias 확인
type mv                          # 실제 실행될 mv·alias 확인
type rm                          # 실제 실행될 rm·alias 확인
```

---

## 요약

- 위치: `pwd`, `cd`
- 조회: `ls -alh`
- 생성: `mkdir -p`
- 빈 디렉터리 삭제: `rmdir`
- 재귀 삭제: `rm -r`
- 파일 속성 보존: `cp -p`
- 디렉터리 백업: `cp -a`
- 이동 목적지 강제: `mv -t`
- 와일드카드 검증: `printf '%s\n' 패턴`
- 관련: **경로 이동 & 목록 조회 (cd & ls & pwd)** · **디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm)** · **복사·이동·와일드카드 (cp · mv · glob)** · **종합실습 lab 프로젝트 디렉터리 표준화 시나리오**
