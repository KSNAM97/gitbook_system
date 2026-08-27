# 허가권·소유권 명령어 퀵 레퍼런스

`chmod`, `chown`, `chgrp`, `umask`, Set-UID, Set-GID, Sticky-bit 및 권한 장애 진단 명령을 빠르게 조회하는 문서.

## 목차

1. [권한 조회](#권한-조회)
2. [chmod](#chmod)
3. [chown·chgrp](#chownchgrp)
4. [umask](#umask)
5. [특수 권한](#특수-권한)
6. [특수 권한 조회표](#특수-권한-조회표)
7. [목적별 디렉터리](#목적별-디렉터리)
8. [ACL](#acl)
9. [SELinux·속성·마운트](#selinux속성마운트)
10. [감사](#감사)
11. [RPM 파일 권한 검증](#rpm-파일-권한-검증)
12. [핵심 판정표](#핵심-판정표)
13. [권한 장애 진단 순서](#권한-장애-진단-순서)

---

## 권한 조회

```bash
ls -l <파일>                               # 파일 권한·소유자·그룹 확인
ls -ld <디렉터리>                          # 디렉터리 내부가 아닌 디렉터리 자체 확인
ls -ln <경로>                              # 소유자·그룹을 UID·GID 숫자로 출력

stat <경로>                                # 권한·소유권·시간·inode 등 상세 확인
stat -c '%A %a %U:%G %u:%g %n' <경로>     # 권한과 소유권을 지정 형식으로 출력

namei -l /전체/경로                        # 경로 구성 요소별 권한·소유권 확인
```

---

## chmod

숫자 방식:

```bash
chmod 600 private.key                      # 소유자만 읽기·쓰기
chmod 640 config.conf                      # 소유자 rw, 그룹 r
chmod 644 document.txt                    # 소유자 rw, 그룹·기타 r

chmod 700 private-dir                     # 소유자만 디렉터리 접근 가능
chmod 750 admin-dir                       # 소유자 rwx, 그룹 r-x
chmod 755 public-dir                      # 모두 접근 가능, 쓰기는 소유자만
chmod 770 team-dir                        # 소유자와 그룹만 전체 권한
```

심볼릭 방식:

```bash
chmod u+x script                          # 소유자에게 실행 권한 추가
chmod g+w shared                          # 그룹에 쓰기 권한 추가
chmod o-r private                         # 기타 사용자의 읽기 권한 제거
chmod o-rwx private-dir                   # 기타 사용자의 모든 권한 제거
chmod u=rw,g=r,o= file                    # 소유자 rw, 그룹 r, 기타 권한 없음
chmod -R g+rX project                     # 그룹 읽기와 조건부 실행 권한 재귀 추가
```

> 대문자 `X`는 디렉터리 또는 기존에 실행 권한이 있는 파일에만 실행 권한을 적용한다.

파일과 디렉터리 구분:

```bash
find /project -type d -exec chmod 750 {} +  # 디렉터리만 750으로 변경
find /project -type f -exec chmod 640 {} +  # 일반 파일만 640으로 변경
```

> 위 권한은 프로젝트의 표준 정책이 명확할 때만 사용한다. 기존 실행 파일은 별도 복구가 필요할 수 있다.

---

## chown·chgrp

소유자:

```bash
chown user file                           # 파일 소유자를 user로 변경
```

소유자·그룹:

```bash
chown user:group file                     # 소유자와 소유 그룹을 함께 변경
```

그룹만:

```bash
chown :group file                         # 소유자는 유지하고 그룹만 변경
chgrp group file                          # chgrp로 소유 그룹 변경
```

사용자의 기본 그룹:

```bash
chown user: file                          # 소유자와 user의 기본 그룹 적용
```

재귀:

```bash
chown -R user:group directory             # 하위 항목까지 소유자·그룹 변경
chgrp -R group directory                  # 하위 항목까지 그룹만 변경
```

기준 파일:

```bash
chown --reference=reference target        # reference와 같은 소유자·그룹 적용
chmod --reference=reference target        # reference와 같은 권한 적용
```

> 그룹만 통일하려는 경우 `chown -R root:group`이 아니라 `chgrp -R group`을 검토한다.

---

## umask

```bash
umask                                     # 현재 umask를 숫자로 확인
umask -S                                  # 현재 결과를 심볼릭 권한으로 확인

umask 0002                                # 그룹 협업형 기본 권한
umask 0022                                # 일반적인 공개 읽기형 기본 권한
umask 0027                                # 기타 사용자 권한 차단
umask 0077                                # 소유자 이외의 권한 차단
```

| umask | 파일 | 디렉터리 |
|---:|---:|---:|
| `0002` | `664` | `775` |
| `0022` | `644` | `755` |
| `0027` | `640` | `750` |
| `0077` | `600` | `700` |

계산:

```text
최종 권한 = 요청 모드 & ~umask

일반 파일의 일반적인 요청 모드 → 666
디렉터리의 일반적인 요청 모드  → 777
```

> 일반 파일은 보안상 기본 요청 모드에 실행 권한이 포함되지 않는다.

---

## 특수 권한

Set-UID:

```bash
chmod 4755 executable                     # 숫자 방식으로 Set-UID 설정
chmod u+s executable                      # 기존 권한을 유지하며 Set-UID 추가
```

Set-GID:

```bash
chmod 2770 shared-dir                     # 그룹 상속용 Set-GID 디렉터리 설정
chmod g+s shared-dir                      # 기존 권한을 유지하며 Set-GID 추가
```

Sticky-bit:

```bash
chmod 1777 public-tmp                     # 공개 쓰기 공간에 Sticky-bit 설정
chmod +t public-tmp                       # 기존 권한을 유지하며 Sticky-bit 추가
```

Set-GID + Sticky-bit:

```bash
chmod 3770 protected-team-dir             # 그룹 상속과 삭제 제한을 함께 설정
```

해제:

```bash
chmod u-s executable                      # Set-UID 제거
chmod g-s shared-dir                      # Set-GID 제거
chmod -t public-tmp                       # Sticky-bit 제거
```

> root 소유 Set-UID는 검증된 시스템 바이너리에만 제한적으로 사용한다. 임의 프로그램에 `4755`를 설정하지 않는다.

---

## 특수 권한 조회표

| 권한 | 숫자 | 표시 위치 |
|---|---:|---|
| Set-UID | 4 | Owner `x` 자리에 `s/S` |
| Set-GID | 2 | Group `x` 자리에 `s/S` |
| Sticky-bit | 1 | Other `x` 자리에 `t/T` |
| Set-GID + Sticky | 3 | Group `s`, Other `t/T` |

```text
소문자 s·t → 해당 위치의 x도 있음
대문자 S·T → 특수 권한은 있으나 x 없음
```

---

## 목적별 디렉터리

개인 전용:

```bash
chmod 700 private-dir                     # 소유자만 접근할 수 있도록 설정
```

공개 조회:

```bash
chmod 755 public-read-dir                 # 모두 조회·접근 가능, 쓰기는 소유자만
```

팀 공유:

```bash
chown root:team team-dir                  # 소유 그룹을 team으로 변경
chmod 2770 team-dir                       # 팀 전체 권한과 그룹 자동 상속 설정
```

팀 공유 + 삭제 보호:

```bash
chmod 3770 protected-team-dir             # 그룹 상속과 Sticky-bit 설정
```

공개 임시 공간:

```bash
chown root:root public-temp-dir           # 소유자·그룹을 root로 설정
chmod 1777 public-temp-dir                # 모두 생성 가능, 임의 삭제 제한
```

---

## ACL

현재 ACL:

```bash
getfacl <경로>                            # 기본 권한과 확장 ACL 확인
```

사용자 ACL:

```bash
setfacl -m u:user1:rw <파일>              # user1에게 읽기·쓰기 권한 부여
```

그룹 ACL:

```bash
setfacl -m g:team:rwx <디렉터리>          # team 그룹에 rwx 권한 부여
```

기본 ACL:

```bash
setfacl -d -m g:team:rwx,m::rwx <디렉터리> # 새 하위 항목에 상속할 ACL 설정
```

ACL 삭제:

```bash
setfacl -x u:user1 <파일>                  # user1 ACL 항목만 삭제
setfacl -b <파일>                          # 모든 확장 ACL 삭제
```

ACL 백업:

```bash
getfacl -R /project > /root/project.acl    # /project ACL을 재귀적으로 백업
```

복원:

```bash
setfacl --restore=/root/project.acl        # 백업 파일로 ACL 복원
```

> ACL의 실제 유효 권한은 `mask` 항목의 제한을 받을 수 있으므로 `getfacl`의 `effective` 표시를 확인한다.

---

## SELinux·속성·마운트

SELinux:

```bash
getenforce                                 # SELinux 현재 모드 확인
ls -lZ <경로>                              # SELinux 컨텍스트 확인
restorecon -Rv <경로>                      # 정책 기준으로 컨텍스트 재적용
ausearch -m AVC -ts recent                 # 최근 SELinux 거부 기록 확인
```

파일 속성:

```bash
lsattr <경로>                              # immutable 등 파일 속성 확인
chattr +i <파일>                           # immutable 설정
chattr -i <파일>                           # immutable 해제
```

마운트:

```bash
findmnt -T <경로>                          # 경로가 속한 파일시스템 확인
findmnt -no SOURCE,FSTYPE,OPTIONS <경로>    # 소스·종류·마운트 옵션만 출력
```

---

## 감사

Set-UID·Set-GID 파일:

```bash
find / -xdev -type f -perm /6000 -ls 2>/dev/null  # SUID 또는 SGID 파일 검색
```

Set-GID·Sticky 디렉터리:

```bash
find / -xdev -type d -perm /3000 -ls 2>/dev/null  # SGID 또는 Sticky 디렉터리 검색
```

소유자·그룹 없는 파일:

```bash
find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null  # 소유자·그룹 없는 항목 검색
```

> `-xdev`는 별도 마운트된 파일시스템을 검색하지 않는다. `/home`, `/data` 등이 별도 볼륨이면 각 마운트포인트에서 반복한다.

---

## RPM 파일 권한 검증

파일 소유 패키지:

```bash
rpm -qf <파일>                             # 파일을 설치한 RPM 패키지 확인
```

패키지명 변수:

```bash
pkg=$(rpm -qf --qf '%{NAME}\n' <파일>)     # 패키지 이름을 변수에 저장
```

검증:

```bash
rpm -V "$pkg"                              # 패키지 파일의 속성·체크섬 검증
```

재설치:

```bash
dnf reinstall "$pkg"                       # 해당 패키지 재설치
```

`/usr/bin/passwd`:

```bash
pkg=$(rpm -qf --qf '%{NAME}\n' /usr/bin/passwd) # passwd 소유 패키지 확인
rpm -V "$pkg"                              # 패키지 변경 여부 검증
dnf reinstall "$pkg"                       # 패키지 재설치
```

---

## 핵심 판정표

| 작업 | 필요한 대표 권한 |
|---|---|
| 파일 읽기 | 파일 `r` |
| 파일 수정 | 파일 `w` |
| 파일 실행 | 파일 `x` |
| 디렉터리 목록 | 디렉터리 `r` |
| 디렉터리 접근 | 디렉터리 `x` |
| 파일 생성 | 부모 디렉터리 `w+x` |
| 파일 삭제 | 부모 디렉터리 `w+x` + Sticky 정책 |
| 파일명 변경 | 부모 디렉터리 `w+x` + Sticky 정책 |

---

## 권한 장애 진단 순서

```bash
id                                         # 현재 UID·GID·보조 그룹 확인
namei -l /전체/경로                        # 상위 경로를 포함한 권한 확인
stat <경로>                                # 대상 권한·소유권·inode 확인

getfacl <경로>                             # 확장 ACL과 유효 권한 확인
ls -lZ <경로>                              # SELinux 컨텍스트 확인
lsattr <경로>                              # immutable 등 파일 속성 확인

findmnt -T <경로>                          # 파일시스템과 마운트 옵션 확인
df -h <경로>                               # 남은 저장공간 확인
df -i <경로>                               # 남은 inode 확인
```

## 요약

- 권한 확인: `stat`, `namei -l`
- 권한 변경: `chmod`
- 소유권 변경: `chown`, `chgrp`
- 기본 생성 권한: `umask`
- 그룹 상속: `2770`
- 그룹 상속 + 삭제 보호: `3770`
- 공개 임시 공간: `1777`
- 관련: **허가권 (Permission) — chmod & rwx·UGO 모델** · **허가권 상세 (chmod & 8진수 · 심볼릭 표기)** · **소유권 (Ownership) — chown & UID·GID 소유 모델** · **소유권 & 특수 권한 (chown & chgrp & SUID · SGID · Sticky)** · **Umask — 기본 권한 마스크 (User Mask)** · **특수권한 Set-GID — 소유 그룹 자동 상속 (2XXX)** · **특수권한 Sticky-bit — 공유 디렉터리 삭제 방지 (1XXX)** · **특수권한 Set-UID — 실행 중 소유자 권한 위임 (4XXX)** · **허가권·소유권 통합 정리** · **허가권·소유권 트러블슈팅 치트시트**
