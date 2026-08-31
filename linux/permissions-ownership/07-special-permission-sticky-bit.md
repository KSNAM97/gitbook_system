# 특수권한 Sticky-bit — 공유 디렉터리 삭제 방지 (1XXX)

> **Tag:** #Linux #StickyBit #SharedDirectory #Security  
> **핵심 요약:** Sticky-bit는 여러 사용자가 쓰기 가능한 공유 디렉터리에서 일반 사용자가 다른 사용자의 파일과 디렉터리 엔트리를 삭제하거나 이름 변경하지 못하게 한다. 파일 읽기와 내용 수정은 파일 자체의 `r`, `w` 권한으로 별도 판정한다.

---

## 1. 개요 (Overview)

**`777` 공유 디렉터리의 문제점은?**

```bash
chmod 777 /share
```

모든 사용자는 `/share`에 대해 다음 권한을 가진다.

```text
r → 목록 조회
w → 엔트리 생성·삭제·이름 변경
x → 디렉터리 접근
```

Sticky-bit가 없으면 다른 사용자가 만든 파일도 삭제하거나 이름을 변경할 수 있다.

**Sticky-bit를 설정하면 누가 삭제할 수 있는가?**

Sticky-bit 디렉터리에서는 일반적으로 다음 주체만 엔트리를 삭제하거나 이름 변경할 수 있다.

- 해당 파일 또는 하위 디렉터리의 소유자
- Sticky-bit 디렉터리의 소유자
- root 또는 관련 권한을 가진 프로세스

**Sticky-bit가 파일 읽기와 수정을 막는가?**

아니다.

| 작업 | 판정 기준 |
|---|---|
| 파일 읽기 | 파일 자체 `r` |
| 파일 복사 | 원본 파일 `r` |
| 파일 내용 수정 | 파일 자체 `w` |
| 파일 삭제 | 상위 디렉터리 `w+x` + Sticky |
| 파일명 변경 | 상위 디렉터리 `w+x` + Sticky |

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 공개 임시 공유 디렉터리

```bash
mkdir /share
chown root:root /share
chmod 1777 /share
```

확인:

```bash
ls -ld /share
stat -c '%A %a %U:%G %n' /share
```

예상:

```text
drwxrwxrwt 1777 root:root /share
```

대표적인 시스템 디렉터리:

```bash
ls -ld /tmp
```

### Step 2. 팀 전용 Sticky 디렉터리

그룹 생성:

```bash
groupadd soldesk
```

사용자 추가:

```bash
usermod -aG soldesk solUser1
usermod -aG soldesk solUser2
usermod -aG soldesk solUser3
```

디렉터리 구성:

```bash
mkdir -p /solhome/sol_tmp
chown root:soldesk /solhome/sol_tmp
chmod 3770 /solhome/sol_tmp
```

권한:

```text
3    = Set-GID 2 + Sticky-bit 1
770  = Owner rwx, Group rwx, Other ---
```

확인:

```bash
ls -ld /solhome/sol_tmp
```

예상:

```text
drwxrws--T root soldesk /solhome/sol_tmp
```

Other에 `x`가 없으므로 Sticky 위치가 대문자 `T`로 표시된다.

### Step 3. 사용자별 검증

`solUser1`:

```bash
touch /solhome/sol_tmp/user1F
mkdir /solhome/sol_tmp/user1D
```

`solUser2`:

```bash
cat /solhome/sol_tmp/user1F
cp /solhome/sol_tmp/user1F ~/
mv /solhome/sol_tmp/user1F /solhome/sol_tmp/renamed
rm /solhome/sol_tmp/user1F
```

예상:

```text
읽기·복사 → 파일 r 권한에 따라 가능
내용 수정 → 파일 w 권한에 따라 결정
이름 변경 → Sticky-bit로 거부
삭제      → Sticky-bit로 거부
```

guest:

```bash
cd /solhome/sol_tmp
```

Other가 `---`이므로 거부된다.

---

## 3. 검증 및 트러블슈팅

Sticky-bit 확인:

```bash
stat -c '%A %a %n' /share
```

Sticky 디렉터리 검색:

```bash
find / -xdev -type d -perm -1000 -ls 2>/dev/null
```

### Sticky-bit인데 다른 사용자가 파일 내용을 수정했다

Sticky-bit는 내용 수정을 막지 않는다.

```bash
ls -l /share/file
```

파일에 Group 또는 Other `w`가 있다면 수정 가능하다.

필요한 경우:

```bash
chmod go-w /share/file
```

### 디렉터리 소유자가 다른 사용자의 파일을 삭제했다

Sticky-bit 디렉터리의 소유자는 내부 엔트리를 삭제할 수 있다. 디렉터리 소유자를 일반 사용자로 지정할 때 이 점을 고려한다.

공개 공유 공간은 일반적으로:

```bash
chown root:root /share
chmod 1777 /share
```

>  **핵심 요약**
> - Sticky-bit 숫자: `1XXX`
> - 공개 임시 디렉터리: `1777`
> - 그룹 제한 + 그룹 상속 + 삭제 보호: `3770`
> - 읽기·수정은 파일 권한으로 별도 판정
> - 관련: 6-1.  허가권 (Permission) — chmod & rwx·UGO 모델 · 6-6.  특수권한 Set-GID — 소유 그룹 자동 상속 (2XXX)
