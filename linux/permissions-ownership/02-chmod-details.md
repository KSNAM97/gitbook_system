# 🔧 허가권 상세 (chmod & 8진수 · 심볼릭 표기)

> **Tag:** #Linux #Permission #chmod #Octal #SymbolicMode  
> **핵심 요약:** `chmod`는 파일과 디렉터리의 권한 비트를 변경한다. 8진수 방식은 전체 권한을 한 번에 설정하고, 심볼릭 방식은 기존 권한을 기준으로 필요한 권한만 추가·제거·재설정한다.

---

## 1. 📖 개요 (Overview)

**`chmod`의 기본 형식은?**

```bash
chmod [옵션] <MODE> <파일 또는 디렉터리>
```

두 가지 표현 방식을 지원한다.

```text
8진수 방식   → chmod 640 file
심볼릭 방식  → chmod u=rw,g=r,o= file
```

**8진수 권한은 어떻게 계산하는가?**

```text
r = 4
w = 2
x = 1
```

| 권한 | 숫자 |
|---|---:|
| `---` | 0 |
| `--x` | 1 |
| `-w-` | 2 |
| `-wx` | 3 |
| `r--` | 4 |
| `r-x` | 5 |
| `rw-` | 6 |
| `rwx` | 7 |

예시:

```text
rwxr-xr-x = 755
rw-r----- = 640
rwxrwx--- = 770
```

**8진수와 심볼릭 방식의 차이는?**

8진수 방식:

```bash
chmod 640 file
```

Owner, Group, Other 권한 전체를 `640`으로 다시 설정한다.

심볼릭 방식:

```bash
chmod g+w file
```

기존 권한은 유지하고 Group 쓰기 권한만 추가한다.

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 8진수 방식

```bash
chmod 600 private.key
chmod 640 application.conf
chmod 644 document.txt
chmod 700 private-directory
chmod 750 admin-directory
chmod 755 public-directory
chmod 770 team-directory
```

### 2-2. 심볼릭 방식

```bash
chmod u+x script.sh
chmod g+w shared-file
chmod o-r private-file
chmod o-rwx private-directory
chmod a+r document.txt
```

정확한 권한 지정:

```bash
chmod u=rw,g=r,o= file
chmod u=rwx,g=rx,o= directory
```

### 2-3. 여러 대상 동시 지정

```bash
chmod ug+rw file
chmod go-rwx private
chmod a-x document
```

### 2-4. 대문자 `X`

```bash
chmod -R g+rX /project
```

`X`는 다음 대상에 실행 권한을 부여한다.

- 디렉터리
- 기존에 실행 권한이 하나라도 있는 파일

일반 문서 파일에 불필요한 실행 권한을 주지 않으면서 디렉터리 탐색 권한을 추가할 때 유용하다.

### 2-5. 재귀 변경

```bash
chmod -R 750 /project
```

파일과 디렉터리 권한을 구분하려면:

```bash
find /project -type d -exec chmod 750 {} +
find /project -type f -exec chmod 640 {} +
```

팀 협업 디렉터리:

```bash
find /project -type d -exec chmod 2770 {} +
find /project -type f -exec chmod 0660 {} +
```

### 2-6. 기준 파일과 동일한 권한 적용

```bash
chmod --reference=reference-file target-file
```

검증:

```bash
stat -c '%a %n' reference-file target-file
```

---

## 3. 🔍 검증 및 트러블슈팅

### 3-1. 권한 확인

```bash
ls -l <파일>
ls -ld <디렉터리>
stat -c '%A %a %n' <경로>
```

### 3-2. 대표 함정

| 명령 | 문제 |
|---|---|
| `chmod -R 777 /project` | 모든 사용자에게 과도한 권한 부여 |
| `chmod -R 644 /directory` | 디렉터리 `x`가 사라져 접근 불가 |
| `chmod -R 755 /data` | 모든 일반 파일에 실행 권한 부여 |
| `chmod 666 directory` | 디렉터리 탐색에 필요한 `x` 없음 |

### 3-3. `chmod`가 거부되는 경우

파일 소유자 확인:

```bash
ls -l <경로>
id
```

일반적으로 권한을 변경할 수 있는 주체:

- 파일 소유자
- root
- 관련 capability를 가진 프로세스

추가 확인:

```bash
getfacl <경로>
lsattr <경로>
findmnt -T <경로>
```

> 📌 **핵심 요약**
> - 숫자 방식은 전체 권한 재설정
> - 심볼릭 방식은 필요한 권한만 변경
> - 재귀 적용 시 파일과 디렉터리를 구분
> - 디렉터리는 내부 접근을 위해 `x` 필요
> - 관련: 🔐 허가권 (Permission) — chmod & rwx·UGO 모델 · 🧮Chmod 계산기 · ⚡ 허가권·소유권 명령어 퀵 레퍼런스
