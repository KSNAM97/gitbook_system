# 🧩 파일시스템 기본 명령어 통합 정리 — 경로·조회·생성·복사 한눈에

> **Tag:** #Linux #Filesystem #cd #ls #mkdir #cp #mv #rm #Wildcard #Summary  
> **핵심 요약:** 파일시스템 기본 작업은 위치 확인, 조회, 생성, 복사, 이동, 삭제 순서로 이해한다. 실행 전 현재 위치와 대상 범위를 확인하는 습관이 가장 중요하다.

---

## 1. 📖 개요 (Overview)

파일시스템 작업의 기본 흐름은 다음과 같은 순서로 이해할 수 있다.

```text
pwd로 현재 위치 확인
        ↓
ls/find로 대상 확인
        ↓
mkdir로 목적지 준비
        ↓
cp 또는 mv 실행
        ↓
diff/stat/ls로 검증
        ↓
필요한 경우 rm으로 정리
```

가장 중요한 안전 원칙은 실행 전에 셸이 실제로 처리할 경로와 와일드카드 확장 결과를 확인하는 것이다.

```bash
pwd
printf '%s\n' /etc/a*
ls -ld /home/guest/work/old
```

`cp -p`와 `cp -a`는 각각 다른 상황에 사용한다. 단일 파일의 기본 속성을 보존하려면 `cp -p`를, 디렉터리·심볼릭 링크·확장 속성 등을 최대한 보존하려면 `cp -a`를 사용한다. 일반 사용자는 Root 소유권과 같은 허용되지 않은 메타데이터를 보존할 수 없다.

---

## 2. 🛠️ 표준 개념 정리 (Configuration)

### 2-1. 경로 이동

```bash
pwd                                # 현재 위치
cd /etc/ssh                        # 절대경로
cd ../..                           # 상대경로
cd ~                                # 현재 사용자 홈
cd ~guest                          # guest 홈
cd -                                # 직전 디렉터리
```

### 2-2. 목록 조회

```bash
ls                                 # 이름 목록
ls -l                              # 상세 정보
ls -a                              # 숨김 포함
ls -alh                            # 숨김+상세+크기 단위
ls -ld /etc                        # 디렉터리 자체
ls -lR /lab                       # 재귀 조회
ls -lt | head                     # 최근 수정 항목
ls -lS | head                     # 크기가 큰 항목
```

### 2-3. 디렉터리 생성과 삭제

```bash
mkdir /backup
mkdir -p /lab/linux/projectA/src

rmdir /empty-dir                  # 빈 디렉터리
rm file                           # 파일
rm -r directory                   # 디렉터리 재귀 삭제
rm -rf -- directory               # 확인 없는 재귀 삭제
```

### 2-4. 파일 복사

```bash
cp source destination
cp -p source destination
cp -R directory destination/
cp -a directory/. destination/
cp -u source destination
cp -n source destination
```

### 2-5. 파일 이동과 이름 변경

```bash
mv old-name new-name
mv source /destination/
mv -n source destination
mv -t /destination/ source1 source2
```

### 2-6. 와일드카드

```text
*       0개 이상의 문자
?       정확히 한 문자
```

```bash
printf '%s\n' /etc/a*
printf '%s\n' /etc/*.conf
printf '%s\n' /etc/s???
printf '%s\n' /etc/??????.conf
```

기본 `*`는 숨김파일을 포함하지 않는다.

```bash
cp -a /etc/skel/. /home/newuser/
```

### 2-7. 덮어쓰기 옵션

| 옵션 | `cp` | `mv` | 설명 |
|---|---:|---:|---|
| `-i` | O | O | 덮어쓰기 전 확인 |
| `-f` | O | O | 강제 처리 시도 |
| `-n` | O | O | 기존 파일 유지 |
| `-u` | O | O | 원본이 최신일 때 처리 |

### 2-8. 표준 실습 환경

```bash
mkdir -p /lab/linux/projectA/src
mkdir -p /lab/linux/projectA/conf
mkdir -p /lab/linux/projectB/logs
mkdir -p /lab/backup
mkdir -p /lab/users/guest1

cp -p /etc/passwd      /lab/backup/passwd.orig
cp -p /etc/group       /lab/backup/group.orig
cp -p /etc/login.defs  /lab/backup/login.defs.orig
cp -p /etc/resolv.conf /lab/backup/resolv.conf.orig

mkdir -p /home/guest/work/a
mkdir -p /home/guest/work/b
mkdir -p /home/guest/work/c/sub1
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 공통 검증

```bash
pwd
ls -alh <경로>
ls -ld <목적지>
stat <원본> <사본>
diff -- <원본> <사본> && echo "OK"
sha256sum <원본> <사본>
find <경로> -maxdepth 2 -print
```

### 3-2. 대표 함정

| 함정 | 원인 | 올바른 접근 |
|---|---|---|
| 자동화에서 파일 없음 | 실행 위치 차이 | 절대경로 또는 명시적 `cd` |
| `mkdir` 중첩 경로 실패 | 상위 경로 없음 | `mkdir -p` |
| `rm`으로 디렉터리 삭제 실패 | `-r` 없음 | `rmdir` 또는 `rm -r` |
| `rm -rf "$DIR"` 사고 | 변수·경로 검증 부족 | `${DIR:?}` + 허용 경로 검증 |
| 디렉터리 복사 생략 | `-R` 없음 | `cp -R` 또는 `cp -a` |
| 숨김파일 복사 누락 | `*`의 기본 동작 | `cp -a source/. destination/` |
| `mv a old/` 결과 혼동 | `old`가 없음 | `mkdir -p` 후 `mv -t` |
| 덮어쓰기 질문 반복 | alias 또는 `-i` | `type cp`, `type mv` 확인 |
| 와일드카드 과다 매칭 | 실행 전 미검증 | `printf '%s\n' 패턴` |
| 패턴이 그대로 출력됨 | 일치 파일 없음 | Glob 결과와 `nullglob` 확인 |

### 3-3. 안전 실행 패턴

```bash
# 현재 위치
pwd

# 원본 확인
ls -l -- /lab/backup/*.orig

# 목적지 준비
mkdir -p /home/guest/work/c/sub1

# 확장 결과 확인
printf '%s\n' /lab/backup/*.orig

# 복사
cp -- /lab/backup/*.orig /home/guest/work/c/sub1/

# 결과 검증
ls -l /home/guest/work/c/sub1/
```

> 📌 **핵심 요약**
> - 위치: `pwd`, `cd`
> - 조회: `ls -alh`, `find`
> - 생성: `mkdir -p`
> - 복사: `cp -p`, `cp -a`
> - 이동: `mv`, `mv -t`
> - 삭제: `rmdir`, `rm -r`
> - 와일드카드: 실행 전 `printf`로 확인
> - 관련: 경로 이동 & 목록 조회 (cd & ls & pwd) · 디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm) · 복사·이동·와일드카드 (cp · mv · glob)
