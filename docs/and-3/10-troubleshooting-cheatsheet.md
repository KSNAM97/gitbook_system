# 허가권·소유권 트러블슈팅 치트시트

`Permission denied`가 발생하면 대상 파일만 보지 말고 현재 사용자의 UID·GID, 상위 경로의 `x`, 대상 권한, ACL, SELinux, 파일 속성, 마운트 상태 순서로 확인한다. `chmod 777`은 근본적인 해결 방법이 아니며 기존 권한 정보를 손실시킬 수 있다.

## 목차

1. [개요 (Overview)](#개요-overview)
2. [증상별 즉시 대응표](#증상별-즉시-대응표)
3. [트러블슈팅 시나리오](#트러블슈팅-시나리오)
4. [긴급 진단 명령 모음](#긴급-진단-명령-모음)

---

## 개요 (Overview)

**가장 먼저 확인할 것은?**

1. 현재 사용자와 그룹
2. 전체 상위 경로 권한
3. 대상 파일·디렉터리 권한
4. 소유자·소유 그룹
5. 특수 권한
6. ACL
7. SELinux
8. 파일 속성
9. 마운트 상태
10. 디스크 공간과 inode

```bash
id
namei -l /path/to/object
stat /path/to/object
getfacl /path/to/object
ls -lZ /path/to/object
lsattr /path/to/object
findmnt -T /path/to/object
df -h /path/to/object
df -i /path/to/object
```

## 증상별 즉시 대응표

| 증상 | 주요 원인 | 확인 |
|---|---|---|
| `cd` 실패 | 디렉터리 `x` 없음 | `namei -l` |
| `ls` 실패 | 디렉터리 `r` 없음 | `ls -ld` |
| 파일 생성 실패 | 디렉터리 `w+x` 없음 | `stat` |
| 파일 수정 실패 | 파일 `w` 없음 | `ls -l`, `getfacl` |
| 파일 삭제 실패 | 부모 권한·Sticky | 부모 `ls -ld` |
| 그룹 권한 미적용 | 세션 미갱신 | `id` |
| Set-GID인데 수정 실패 | Group `w` 없음 | `umask`, ACL |
| `chmod 777`인데 실패 | 상위 경로·SELinux·RO | 전체 진단 |
| `chown` 실패 | 권한 부족 | 소유자·capability |
| SUID 미작동 | `nosuid`·SELinux | `findmnt -T` |
| 공간이 있는데 생성 실패 | inode 부족 | `df -i` |

---

## 트러블슈팅 시나리오

### 시나리오 1. 파일이 `777`인데 접근할 수 없다

```bash
namei -l /path/to/file
```

추가 확인:

```bash
getfacl /path/to/file
ls -lZ /path/to/file
findmnt -T /path/to/file
```

가능한 원인:

- 상위 디렉터리 `x` 없음
- ACL 제한
- SELinux
- 읽기 전용 마운트
- 컨테이너 제한

---

### 시나리오 2. 읽기 전용 파일이 삭제된다

```bash
ls -l /path/to/file
ls -ld /path/to
```

파일 삭제는 상위 디렉터리 엔트리 조작이다.

타인 삭제 제한:

```bash
chmod +t /path/to
```

공개 임시 공간:

```bash
chmod 1777 /path/to
```

---

### 시나리오 3. 그룹 추가 후 접근할 수 없다

계정 데이터베이스:

```bash
id <사용자>
```

현재 세션:

```bash
id
```

현재 세션에 그룹이 없다면 재로그인한다.

```bash
newgrp <그룹>
```

---

### 시나리오 4. root도 수정할 수 없다

```bash
findmnt -T /path/to/file
lsattr /path/to/file
ls -lZ /path/to/file
getfacl /path/to/file
```

가능한 원인:

- 읽기 전용 파일시스템
- immutable 속성
- SELinux
- NFS `root_squash`
- 컨테이너 권한 제한
- 파일시스템·스토리지 오류

---

### 시나리오 5. Set-GID인데 새 파일이 `644`이다

Set-GID는 그룹을 상속하지만 Group 쓰기 권한까지 보장하지 않는다.

```bash
umask
ls -l <파일>
```

사용자 셸 정책:

```bash
umask 0002
```

기본 ACL:

```bash
chown root:project /project
chmod 2770 /project

setfacl -m g:project:rwx,m::rwx /project
setfacl -d -m g:project:rwx,m::rwx /project
```

검증:

```bash
getfacl /project
```

> `g::rwx`는 이름 있는 `project` 그룹이 아니라 객체의 소유 그룹 항목이다. 이름 있는 그룹을 지정하려면 `g:project:rwx`를 사용한다.

---

### 시나리오 6. `chmod -R 644` 후 디렉터리에 들어갈 수 없다

디렉터리 `x`가 제거된 상태다.

표준 정책이 디렉터리 `750`, 일반 파일 `640`으로 명확하다면:

```bash
find /project -type d -exec chmod 750 {} +
find /project -type f -exec chmod 640 {} +
```

이 명령은 기존 실행 파일의 실행 권한까지 복구하지는 못한다.

패키지 파일:

```bash
pkg=$(rpm -qf --qf '%{NAME}\n' <파일>)
rpm -V "$pkg"
```

형상관리와 백업도 확인한다.

---

### 시나리오 7. `/usr/bin/passwd`가 동작하지 않는다

현재 권한:

```bash
ls -l /usr/bin/passwd
```

소유 패키지 확인:

```bash
pkg=$(rpm -qf --qf '%{NAME}\n' /usr/bin/passwd)
printf 'package=%s\n' "$pkg"
```

검증:

```bash
rpm -V "$pkg"
```

복구:

```bash
dnf reinstall "$pkg"
```

재확인:

```bash
ls -l /usr/bin/passwd
rpm -V "$pkg"
```

> `/usr/bin/passwd`의 패키지명을 `shadow-utils`로 고정하지 말고 `rpm -qf` 결과를 사용한다.

---

### 시나리오 8. `chmod -R 777` 후 원래 권한을 모르겠다

`chmod`는 이전 권한을 저장하지 않는다. 자동 Undo 명령도 없다.

복원 자료 확인:

- ACL 백업
- 시스템 백업
- Git·형상관리
- RPM 패키지 메타데이터
- 애플리케이션 공식 권한 정책
- 다른 동일 서버와 비교

ACL 백업 복원:

```bash
setfacl --restore=/root/project.acl
```

패키지 검증:

```bash
rpm -qf <파일>
rpm -V <패키지>
```

그룹만 변경해야 한다면:

```bash
chgrp -R project /project
```

> 모든 사용자 소유자를 root로 바꾸는 `chown -R root:project`를 일반적인 권한 복구 명령으로 사용하지 않는다.

---

### 시나리오 9. 소유자가 숫자로 표시된다

```bash
ls -ln <경로>
getent passwd <UID>
getent group <GID>
```

삭제 계정 잔재:

```bash
find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
```

`-xdev`는 다른 파일시스템을 검색하지 않는다. 별도 마운트포인트도 확인한다.

```bash
find /home -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
find /data -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
```

---

### 시나리오 10. 디스크 공간은 있는데 파일이 생성되지 않는다

용량:

```bash
df -h <경로>
```

inode:

```bash
df -i <경로>
```

권한:

```bash
namei -l <경로>
```

파일시스템 상태:

```bash
findmnt -T <경로>
dmesg --ctime | tail -n 50
```

---

## 긴급 진단 명령 모음

```bash
id
namei -l /path/to/object
stat /path/to/object

getfacl /path/to/object
ls -lZ /path/to/object
lsattr /path/to/object

findmnt -T /path/to/object
df -h /path/to/object
df -i /path/to/object
```

## 요약

- 진단 순서: 사용자 → 상위 경로 → 권한 → ACL → SELinux → 속성 → 마운트
- 삭제·이름 변경은 부모 디렉터리 확인
- 그룹 변경 후 재로그인
- Set-GID와 Group 쓰기 권한은 별개
- `chmod 777`은 근본 해결책이 아님
- 관련: **허가권 (Permission) — chmod & rwx·UGO 모델** · **소유권 (Ownership) — chown & UID·GID 소유 모델** · **특수권한 Set-UID — 실행 중 소유자 권한 위임 (4XXX)** · **허가권·소유권 통합 정리** · **허가권·소유권 명령어 퀵 레퍼런스**
