# 파일시스템 기본 명령어 트러블슈팅 치트시트

> **Tag:** #Linux #Troubleshooting #cd #ls #rm #cp #mv #Wildcard #CheatSheet  
> **핵심 요약:** 경로·조회·생성·복사·이동·삭제 작업에서 발생하는 문제를 증상, 원인, 조치 순서로 빠르게 확인한다.

---

## 1. 개요 (Overview)

가장 위험한 파일시스템 명령은 잘못된 경로에 실행한 `rm -rf`이다. 이를 막기 위해서는 다음 항목을 함께 사용해야 한다: `pwd`로 현재 위치 확인, `ls` 또는 `find`로 대상 확인, `${VAR:?}`로 빈 변수 방지, `realpath`로 경로 정규화, 허용된 경로 범위 검사, 백업 및 스냅샷 확인.

> `set -u`와 `${VAR:?}`만으로 모든 삭제 사고를 막을 수는 없다. 값이 존재하지만 잘못된 경로인 경우에는 별도의 범위 검증이 필요하다.

와일드카드 사고가 발생하는 이유는 `*`, `?`를 셸이 명령 실행 전에 확장하기 때문이다.

```bash
printf '%s\n' /home/guest/work/*/*.orig
```

확장 결과를 확인한 후 실제 명령을 실행한다.

---

## 2. 증상별 즉시 대응표 (Configuration)

### 1. 경로 및 조회

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| 대화형 셸에서는 되지만 Cron에서 실패 | 작업 디렉터리 차이 | 절대경로 또는 명시적 `cd` |
| 상대경로 이동 결과가 다름 | 현재 위치 오인 | `pwd`, `readlink -f` |
| 숨김파일이 보이지 않음 | `ls`에 `-a` 없음 | `ls -alh` |
| 디스크 공간을 차지하는 파일이 안 보임 | 삭제됐지만 프로세스가 점유 | `lsof +L1` |
| 링크의 실제 대상이 혼동됨 | 심볼릭 링크 | `readlink -f`, `namei -l` |

### 2. 생성 및 삭제

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| `mkdir: No such file or directory` | 상위 경로 없음 | `mkdir -p` |
| `rm: Is a directory` | `-r` 없음 | `rmdir` 또는 `rm -r` |
| `rmdir` 실패 | 내용물이 있음 | 내용을 확인한 뒤 별도 삭제 |
| `rm dir/*` 후 숨김파일이 남음 | `*`가 숨김파일 제외 | `find` 사용 |
| 변수 기반 삭제 사고 | 변수·범위 검증 부족 | `${VAR:?}` + `realpath` + 범위 검사 |
| `rm -f`인데 실패 | 권한·파일시스템 문제 | 권한, 마운트 상태, 로그 확인 |

### 3. 복사·이동·와일드카드

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| `omitting directory` | `-R` 없음 | `cp -R` 또는 `cp -a` |
| `.bashrc` 누락 | `*`가 숨김파일 제외 | `cp -a source/. destination/` |
| `mv a old/` 후 위치 혼동 | `old`가 없어서 이름 변경 | `mkdir -p`, `mv -t` |
| `Permission denied` | 상위 디렉터리 권한 부족 | `namei -l` |
| 덮어쓰기 질문 반복 | alias 또는 `-i` | `type cp`, `type mv` |
| 와일드카드 과다 매칭 | 확장 결과 미확인 | `printf '%s\n' 패턴` |
| 와일드카드가 문자 그대로 전달 | 일치 항목 없음 | `shopt`와 패턴 확인 |
| `cp -p`인데 소유자가 바뀜 | 일반 사용자 권한 제한 | Root 실행 또는 사후 권한 설정 |

### 4. 핵심 진단 명령어

```bash
pwd
printf '%s\n' <패턴>
ls -alh <경로>
ls -ld <경로>
readlink -f <경로>
namei -l <경로>
find / -xtype l 2>/dev/null
lsof +L1
stat <파일>
type cp
type mv
type rm
```

---

## 3. 트러블슈팅 시나리오 (Verification & Troubleshooting)

### 시나리오 1. Cron에서만 파일을 찾지 못한다

```bash
crontab -l
journalctl -u crond --since today
```

스크립트에서 절대경로를 사용한다.

```bash
/usr/bin/cp /opt/app/source.txt /opt/app/backup/
```

또는 작업 디렉터리를 명시한다.

```bash
cd /opt/app || exit 1
```

### 시나리오 2. 삭제 변수에 잘못된 값이 들어갔다

```bash
set -u
: "${DIR:?DIR must be set}"

TARGET=$(realpath -e -- "$DIR") || exit 1

case "$TARGET" in
  /home/guest/*) ;;
  *)
    echo "삭제 허용 범위를 벗어났습니다: $TARGET" >&2
    exit 1
    ;;
esac

find "$TARGET" -maxdepth 2 -print
```

확인 후 삭제한다.

```bash
rm -rf -- "$TARGET"
```

### 시나리오 3. `rm`이 디렉터리를 삭제하지 못한다

```bash
ls -ld /path/dir
```

비어 있는 경우:

```bash
rmdir /path/dir
```

내용까지 삭제하는 경우:

```bash
rm -r /path/dir
```

### 시나리오 4. `mv a old/` 후 `a`가 보이지 않는다

`old`가 없었다면 `a`가 `old`라는 이름으로 변경되었을 수 있다.

```bash
ls -ld /home/guest/work/old
```

예방:

```bash
mkdir -p /home/guest/work/old
mv -t /home/guest/work/old/ /home/guest/work/a
```

### 시나리오 5. `cp: -r not specified; omitting directory`

디렉터리 포함 복사:

```bash
cp -R /etc/a* /backup/
```

아카이브 방식:

```bash
cp -a /etc/a* /backup/
```

일반 파일만 복사:

```bash
find /etc -maxdepth 1 -type f -name 'a*' \
  -exec cp -t /backup/ -- {} +
```

### 시나리오 6. `/etc/skel/*` 복사 후 숨김파일이 없다

```bash
cp -a /etc/skel/. /home/newuser/
```

복사 결과:

```bash
ls -la /home/newuser/
```

### 시나리오 7. `mv` 후 심볼릭 링크가 깨졌다

```bash
find / -xtype l 2>/dev/null
readlink -f <링크>
```

링크 재생성:

```bash
ln -sfn <새로운-대상> <링크-경로>
```

### 시나리오 8. `cp` 또는 `mv` 질문이 계속 나온다

```bash
type cp
type mv
alias cp
alias mv
```

현재 alias를 우회한다.

```bash
\cp source destination
\mv source destination
```

자동화에서는 alias가 적용되지 않는 경우가 많으므로 명령 옵션을 명시한다.

### 시나리오 9. `rm dir/*` 후 디렉터리가 비어 있지 않다

숨김파일이나 하위 디렉터리가 남았을 수 있다.

```bash
ls -la /path/dir
find /path/dir -mindepth 1 -maxdepth 1 -print
```

일반 파일만 삭제:

```bash
find /path/dir -mindepth 1 -maxdepth 1 -type f -delete
```

디렉터리 자체를 포함하여 모두 삭제:

```bash
rm -rf -- /path/dir
```

### 시나리오 10. `cp -p`를 사용했는데 Root 소유권이 보존되지 않는다

일반 사용자는 다른 사용자의 소유권을 임의로 설정할 수 없다.

```bash
id
stat /원본 /복사본
```

Root 권한으로 복사해야 하는 관리 작업이라면:

```bash
sudo cp -p /원본 /목적지
```

>  **핵심 요약**
> - 삭제 전 `pwd`, `ls`, `find`
> - 변수 삭제는 빈 값과 허용 범위를 모두 검증
> - 디렉터리 복사는 `cp -R` 또는 `cp -a`
> - 숨김파일 복사는 `source/.`
> - 이동 목적지는 미리 생성하거나 `mv -t` 사용
> - 와일드카드는 `printf`로 확장 결과 확인
> - 관련: 경로 이동 & 목록 조회 (cd & ls & pwd) · 디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm) · 복사·이동·와일드카드 (cp · mv · glob)
