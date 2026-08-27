# 👥 소유권 & 특수 권한 (chown & chgrp & SUID · SGID · Sticky)

> **Tag:** #Linux #Ownership #SpecialPermission #SUID #SGID #StickyBit  
> **핵심 요약:** Linux 특수 권한은 일반적인 Owner·Group·Other 권한만으로 해결하기 어려운 실행 권한 위임과 공유 디렉터리 정책을 보완한다. Set-UID는 실행 파일 소유자 권한, Set-GID는 실행 파일 그룹 권한 또는 디렉터리 그룹 상속, Sticky-bit는 공유 디렉터리의 삭제·이름 변경 제한에 사용한다.

---

## 1. 📖 개요 (Overview)

**특수 권한의 종류는?**

| 특수 권한 | 숫자 | 주요 대상 | 기능 |
|---|---:|---|---|
| Set-UID | 4 | 실행 파일 | 파일 소유자의 유효 UID로 실행 |
| Set-GID | 2 | 실행 파일·디렉터리 | 파일 그룹 권한으로 실행 또는 그룹 상속 |
| Sticky-bit | 1 | 디렉터리 | 타인 소유 엔트리 삭제·이름 변경 제한 |

**숫자 표기는 어떻게 하는가?**

```text
일반 권한: 755
Set-UID:   4755
Set-GID:   2755
Sticky:    1755
```

특수 권한 조합:

```text
Set-GID + Sticky-bit = 2 + 1 = 3
chmod 3770 /team-temp
```

**문자 표시는?**

```text
-rwsr-xr-x   Set-UID
-rwxr-sr-x   실행 파일 Set-GID
drwxrws---   디렉터리 Set-GID
drwxrwxrwt   Sticky-bit
drwxrws--T   Set-GID + Sticky, Other x 없음
```

소문자:

```text
s, t → 해당 위치에 x 권한도 있음
```

대문자:

```text
S, T → 특수 권한은 있지만 해당 위치의 x가 없음
```

## 2. 🛠️ 설정 및 해제

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

Set-UID:

```bash
chmod u+s executable
chmod 4755 executable
```

Set-GID:

```bash
chmod g+s directory
chmod 2770 directory
```

Sticky-bit:

```bash
chmod +t directory
chmod 1777 directory
```

조합:

```bash
chmod 3770 directory
```

해제:

```bash
chmod u-s executable
chmod g-s directory
chmod -t directory
```

---

## 3. 🔍 검증 및 감사

특수 권한 확인:

```bash
ls -ld <경로>
stat <경로>
```

Set-UID·Set-GID 실행 파일 검색:

```bash
find / -xdev -type f -perm /6000 -ls 2>/dev/null
```

Set-GID·Sticky 디렉터리 검색:

```bash
find / -xdev -type d -perm /3000 -ls 2>/dev/null
```

> 특수 권한은 일반 사용자에게 root 전체 권한을 직접 부여하는 기능이 아니다. 실행 프로그램의 로직, 파일 소유자, 마운트 옵션, capability, SELinux 정책이 함께 작동한다.

> 📌 **핵심 요약**
> - Set-UID: 실행 중 파일 소유자의 EUID
> - Set-GID 파일: 실행 중 파일 소유 그룹의 EGID
> - Set-GID 디렉터리: 새 객체의 그룹 상속
> - Sticky-bit: 타인 엔트리 삭제·이름 변경 제한
> - 관련: 🪪 특수권한 Set-UID — 실행 중 소유자 권한 위임 (4XXX) · 📌 특수권한 Sticky-bit — 공유 디렉터리 삭제 방지 (1XXX) · 👥 특수권한 Set-GID — 소유 그룹 자동 상속 (2XXX)
