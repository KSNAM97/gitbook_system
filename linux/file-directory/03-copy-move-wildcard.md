# 📋 복사·이동·와일드카드 (cp · mv · glob)

> **Tag:** #Linux #cp #mv #Wildcard #Glob #Backup  
> **핵심 요약:** `cp`는 원본을 유지한 채 복사하고, `mv`는 파일이나 디렉터리를 이동하거나 이름을 변경한다. 와일드카드 `*`, `?`는 셸이 명령 실행 전에 확장하므로 실행 전 결과를 확인해야 한다.

---

## 1. 📖 개요 (Overview)

`cp`와 `mv`의 차이는 다음과 같다.

| 명령어 | 동작 |
|---|---|
| `cp` | 원본을 유지하고 복사본 생성 |
| `mv` | 원본을 다른 위치로 이동하거나 이름 변경 |

```bash
cp source destination
mv source destination
```

`cp -p`와 `cp -a`의 차이를 보면, `cp -p`는 권한, 소유자·그룹, 접근 시간과 수정 시간의 보존을 시도하는 반면, `cp -a`는 디렉터리 재귀 복사, 심볼릭 링크 자체 보존, 권한·소유권·시간·확장 속성 등 최대한 보존을 수행하며 시스템 트리나 디렉터리 백업에 적합하다.

```bash
cp -p /etc/passwd /backup/passwd.orig
cp -a /etc/skel/. /home/newuser/
```

> 일반 사용자는 자신에게 허용되지 않은 소유권을 설정할 수 없다. 따라서 `cp -p`를 사용하더라도 Root 소유권 보존이 실패하거나 복사한 사용자 소유로 생성될 수 있다.

와일드카드가 어디에서 처리되는지에 대해서는, Bash와 같은 셸이 먼저 패턴을 파일 목록으로 확장한 뒤 명령에 전달한다.

```bash
cp /etc/hosts* /backup/
```

개념적인 처리:

```text
cp /etc/hosts /etc/hosts.allow ... /backup/
```

기본 Bash에서 패턴과 일치하는 항목이 없으면 패턴이 제거되거나 즉시 오류가 발생하는 것이 아니라, 일반적으로 패턴 문자열이 그대로 명령에 전달된다.

```bash
echo /no-such-path/*.orig
```

`nullglob`, `failglob` 설정에 따라 동작이 달라질 수 있다.

```bash
shopt -p nullglob failglob
```

---

## 2. 🛠️ 표준 사용 템플릿 (Configuration)

### 2-1. `cp` 기본 복사

원본 이름 그대로 복사:

```bash
cp /backup/login.defs /home/guest/
```

목적지 파일명을 생략:

```bash
cd /home/guest
cp /backup/inittab ./
```

이름을 변경하여 복사:

```bash
cp /backup/passwd /home/guest/guestPasswd
```

### 2-2. 속성 보존 복사

```bash
cp -p /home/guest/guestPasswd /backup/
```

디렉터리와 메타데이터를 최대한 보존한다.

```bash
cp -a /etc/skel/. /home/newuser/
```

### 2-3. 최신 파일만 복사

```bash
cp -u /etc/passwd /backup/passwd
```

속성 보존과 함께 사용:

```bash
cp -pu /backup/login.defs /home/guest/logout.defs
```

`-u`는 원본의 수정 시간이 목적지보다 최신이거나 목적지가 없을 때 복사한다. 파일 내용 자체를 비교하는 옵션은 아니다.

### 2-4. 덮어쓰기 옵션

| 옵션 | 의미 |
|---|---|
| `-i` | 덮어쓰기 전 확인 |
| `-f` | 필요 시 기존 목적지를 제거하고 복사·이동 시도 |
| `-n` | 기존 목적지를 덮어쓰지 않음 |
| `-u` | 원본이 더 최신일 때 복사·이동 |

```bash
cp -i source destination
cp -f source destination
cp -n source destination
cp -u source destination

mv -i source destination
mv -f source destination
mv -n source destination
mv -u source destination
```

> `-i`는 명령의 기본값이 아니다. 프롬프트가 자동으로 표시된다면 alias를 확인한다.

```bash
type cp
type mv
alias cp
alias mv
```

### 2-5. 디렉터리 복사

```bash
cp -R /etc/ssh /backup/
```

메타데이터와 심볼릭 링크를 포함해 최대한 원형으로 복사:

```bash
cp -a /etc/ssh /backup/
```

숨김파일을 포함하여 디렉터리 내용만 복사:

```bash
cp -a /etc/skel/. /home/newuser/
```

다음 명령은 숨김파일을 제외한다.

```bash
cp -R /etc/skel/* /home/newuser/
```

### 2-6. 다중 파일 복사

여러 원본을 하나의 목적지 디렉터리로 복사한다.

```bash
cp /home/guest/passwd \
   /home/guest/inittab \
   /home/guest/logout.defs \
   /soldesk/linux/rocky/version9/
```

- 마지막 인자는 반드시 목적지 디렉터리여야 한다.
- 여러 원본을 복사하면서 각각 다른 이름을 지정할 수는 없다.
- 디렉터리를 포함하려면 `-R` 또는 `-a`를 사용한다.

하나의 파일을 두 목적지에 배포하려면 `cp`를 두 번 실행한다.

```bash
cp /lab/backup/resolv.conf.orig /home/guest/work/a/ &&
cp /lab/backup/resolv.conf.orig /home/guest/work/b/
```

### 2-7. `mv` 이동과 이름 변경

이름 변경:

```bash
mv passwd.lab passwd.final
```

다른 디렉터리로 이동:

```bash
mv /backup/aliases /soldesk/linux/rocky/version9/
```

이동하면서 이름 변경:

```bash
mv /backup/passwd /backup/passwd.old
```

디렉터리 전체 이동:

```bash
mkdir -p /home/guest/work/old
mv /home/guest/work/a /home/guest/work/old/
```

목적지를 반드시 디렉터리로 처리:

```bash
mv -t /home/guest/work/old/ /home/guest/work/a
```

> 파일 이동에는 원본 파일 자체의 쓰기 권한보다 원본이 들어 있는 디렉터리와 목적지 디렉터리에 대한 적절한 권한이 중요하다.

### 2-8. 와일드카드

#### `*` — 0개 이상의 문자

```bash
ls /etc/a*
ls /etc/*.conf
ls /etc/*log*
cp /backup/*.orig /home/guest/work/c/
```

#### `?` — 정확히 한 문자

```bash
ls /etc/s???          # s로 시작하고 뒤에 3문자
ls /etc/??????.conf   # 확장자 제외 이름 6문자
ls /etc/s??l          # s로 시작하고 l로 끝나는 4문자
ls /etc/su????        # su 뒤에 4문자가 오는 6문자
ls /etc/?r???         # 두 번째 문자가 r인 5문자
```

`?`는 현재 로케일 기준 한 문자를 의미한다.

#### 숨김파일

기본적으로 `*`는 `.`으로 시작하는 숨김파일을 매칭하지 않는다.

```bash
ls /etc/skel/*
ls -la /etc/skel
```

숨김파일까지 복사:

```bash
cp -a /etc/skel/. /home/newuser/
```

### 2-9. 와일드카드 사전 검증

```bash
printf '%s\n' /etc/a*
printf '%s\n' /etc/s???
printf '%s\n' /lab/backup/*.orig
```

`echo`도 사용할 수 있지만 한 줄로 출력되므로 `printf`가 항목 구분에 더 명확하다.

```bash
echo /etc/a*
```

### 2-10. 이름 기준 검색과 `grep` 구분

다음 명령은 `ls -l` 출력 전체에서 문자 `a`를 검색하므로 파일명만 검사하지 않는다.

```bash
ls -l /backup | grep a
```

현재 디렉터리에서 이름이 `a`로 시작하는 항목을 조회하려면:

```bash
printf '%s\n' /backup/a*
```

디렉터리만 조회하려면:

```bash
find /backup -mindepth 1 -maxdepth 1 -type d -print
```

파일만 조회하려면:

```bash
find /backup -mindepth 1 -maxdepth 1 -type f -name 'a*' -print
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
ls -l <목적지>
stat <원본> <목적지>
diff -- <원본> <목적지> && echo "OK"
sha256sum <원본> <목적지>
printf '%s\n' <와일드카드>
```

`md5sum`도 단순 복사 오류 확인에 사용할 수 있지만, 보안 무결성 검증에는 SHA-256 이상을 사용한다.

### 3-2. 대표 트러블슈팅

#### 🚨 시나리오 1. `cp: -r not specified; omitting directory`

와일드카드에 디렉터리가 포함되었지만 재귀 옵션이 없기 때문이다.

```bash
cp -R /etc/a* /backup/
```

메타데이터까지 보존하려면:

```bash
cp -a /etc/a* /backup/
```

일반 파일만 복사하려면:

```bash
find /etc -maxdepth 1 -type f -name 'a*' \
  -exec cp -t /backup/ -- {} +
```

#### 🚨 시나리오 2. 복사할 때 덮어쓰기 질문이 반복된다

```bash
type cp
alias cp
```

현재 셸의 alias만 우회:

```bash
\cp -R /etc/a* /backup/
```

비대화형 복사가 정말 필요한지 확인한 뒤:

```bash
cp -Rf /etc/a* /backup/
```

#### 🚨 시나리오 3. `cp /etc/skel/*`에서 `.bashrc`가 누락된다

```bash
cp -a /etc/skel/. /home/newuser/
```

#### 🚨 시나리오 4. `mv a old/` 실행 후 `a`가 사라졌다

`old`가 존재하지 않으면 `a`의 이름이 `old`로 변경될 수 있다.

```bash
mkdir -p /home/guest/work/old
mv -t /home/guest/work/old/ /home/guest/work/a
```

#### 🚨 시나리오 5. 일반 사용자로 이동했더니 `Permission denied`

원본과 목적지의 상위 디렉터리 권한을 확인한다.

```bash
namei -l /backup/adjtime
namei -l /soldesk/linux/rocky/
```

#### 🚨 시나리오 6. `mv` 후 심볼릭 링크가 깨졌다

```bash
find / -xtype l 2>/dev/null
readlink <링크>
```

필요한 경우 링크를 다시 생성한다.

```bash
ln -sfn <새로운-대상> <링크-경로>
```

> 📌 **핵심 요약**
> - 기본 복사: `cp`
> - 속성 보존: `cp -p`
> - 디렉터리 백업: `cp -a`
> - 이동·이름 변경: `mv`
> - 목적지 디렉터리 강제: `mv -t`
> - 와일드카드 검증: `printf '%s\n' 패턴`
> - 관련: 경로 이동 & 목록 조회 (cd & ls & pwd) · 디렉터리·파일 생성 및 삭제 (mkdir · rmdir · rm)
