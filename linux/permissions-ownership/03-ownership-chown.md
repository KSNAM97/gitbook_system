# 👤  소유권 (Ownership) — chown & UID·GID 소유 모델

> **Tag:** #Linux #Ownership #chown #chgrp #UID #GID  
> **핵심 요약:** Linux 파일시스템은 소유자 이름이 아니라 UID와 GID를 저장한다. `chown`과 `chgrp`으로 소유자·소유 그룹을 변경할 수 있으며, 일반 사용자는 파일 소유자를 임의로 변경할 수 없고 자신이 소유한 파일의 그룹만 자신이 속한 그룹으로 제한적으로 변경할 수 있다.

---

## 1. 📖 개요 (Overview)

**파일과 디렉터리의 소유권은 무엇으로 구성되는가?**

Linux의 모든 파일과 디렉터리는 다음 두 가지 소유권 정보를 가진다.

```text
사용자 소유권(Owner) → UID
그룹 소유권(Group)   → GID
```

확인:

```bash
ls -l <경로>
```

예:

```text
-rw-r----- 1 user1 teamA 1024 Jul 10 10:00 report.txt
```

숫자 UID·GID:

```bash
ls -ln report.txt
```

예:

```text
-rw-r----- 1 1001 2001 1024 Jul 10 10:00 report.txt
```

이름 매핑:

```text
/etc/passwd → 사용자명과 UID
/etc/group  → 그룹명과 GID
```

NSS 기반 조회:

```bash
getent passwd user1
getent group teamA
```

LDAP·FreeIPA·Active Directory 등 외부 계정 저장소가 연결된 환경에서는 `grep /etc/passwd`보다 `getent`가 정확하다.

**계정 삭제 후 소유자가 숫자로 표시되는 이유는?**

파일에는 UID와 GID가 남아 있지만 이를 이름으로 변환할 계정·그룹 정보가 없기 때문이다.

```text
-rw-r--r-- 1 1200 1220 file.txt
```

같은 UID를 새 계정에 재사용하면 기존 파일이 새 계정 소유처럼 표시될 수 있다.

삭제 전:

```bash
id <사용자>
find / -xdev -uid <UID> -ls 2>/dev/null
```

> `-xdev`는 시작점과 다른 파일시스템으로 넘어가지 않는다. `/home`, `/data` 등이 별도 마운트라면 각 마운트포인트를 별도로 검사한다.

로컬 XFS·ext4 마운트포인트 확인:

```bash
findmnt -rn -t xfs,ext4 -o TARGET
```

마운트포인트별 검색 예:

```bash
while IFS= read -r mountpoint; do
  find "$mountpoint" -xdev -uid <UID> -ls 2>/dev/null
done < <(findmnt -rn -t xfs,ext4 -o TARGET)
```

삭제된 계정·그룹 잔재:

```bash
find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
```

별도 마운트:

```bash
find /home -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
find /data -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
```

**소유권은 허가권 판정에 어떻게 사용되는가?**

```text
-rw-r----- 1 user1 teamA report.txt
```

```text
user1              → Owner rw-
teamA 그룹 사용자  → Group r--
그 외 사용자       → Other ---
```

Owner가 `teamA` 그룹에도 속하더라도 Owner와 Group 권한을 합산하지 않는다. Owner UID가 일치하면 Owner 권한을 적용한다.

**일반 사용자가 소유자를 변경할 수 있는가?**

일반적으로 불가능하다.

파일 소유자를 임의로 변경할 수 있으면 다음 문제가 발생한다.

- 악성 파일 소유권 전가
- 디스크 쿼터 우회
- 감사 추적 방해
- 특수 권한 악용

소유자 변경에는 일반적으로 root 또는 `CAP_CHOWN`이 필요하다.

```bash
sudo chown user2 file
```

일반 사용자:

```bash
chown user2 myfile
```

예상:

```text
Operation not permitted
```

**일반 사용자가 소유 그룹을 변경할 수 있는가?**

일반적으로 다음 조건이 필요하다.

1. 자신이 해당 파일의 소유자
2. 변경하려는 그룹에 자신이 속함

현재 그룹:

```bash
id
```

변경:

```bash
chgrp groupB myfile
```

또는:

```bash
chown :groupB myfile
```

속하지 않은 그룹으로 변경하면 거부된다.

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 소유자만 변경

```bash
chown user1 file
```

```text
변경 전: root:root
변경 후: user1:root
```

---

### 2-2. 소유자와 그룹 변경

```bash
chown user1:teamA file
```

결과:

```text
user1:teamA
```

과거 점 표기보다 콜론 형식을 권장한다.

```bash
chown user1:teamA file
```

---

### 2-3. 그룹만 변경

```bash
chown :teamA file
```

또는:

```bash
chgrp teamA file
```

그룹 변경 목적이 명확할 때 `chgrp`가 읽기 쉽다.

---

### 2-4. `chown user:` 형식

```bash
chown user1: file
```

소유자를 `user1`로 변경하고 그룹을 `user1`의 기본 그룹으로 변경한다.

기본 그룹:

```bash
id user1
```

---

### 2-5. 재귀 소유권 변경

```bash
chown -R user1:teamA /project
```

적용 전:

```bash
find /project -printf '%u:%g %p\n'
```

적용 후:

```bash
find /project -printf '%u:%g %p\n'
```

> `chown -R`은 사용자별 소유권을 모두 변경할 수 있다. 전체 하위 객체의 사용자 소유자를 정말 통일해야 하는 경우에만 사용한다.

그룹만 재귀 변경:

```bash
chgrp -R teamA /project
```

---

### 2-6. 기준 파일과 동일하게 변경

```bash
chown --reference=reference-file target-file
```

검증:

```bash
stat -c '%U:%G %n' reference-file target-file
```

---

### 2-7. 실습 — 소유자와 그룹 변경

```bash
mkdir -p /home/imsi
touch /home/imsi/aliases

chown guest:global /home/imsi/aliases
chmod 770 /home/imsi/aliases
```

확인:

```bash
ls -l /home/imsi/aliases
```

예상:

```text
-rwxrwx--- guest global /home/imsi/aliases
```

---

### 2-8. 일반 사용자의 그룹 변경

root:

```bash
cp /etc/networks /home/user1/
chown user1:teamA /home/user1/networks
```

확인:

```bash
id user1
ls -l /home/user1/networks
```

`user1`이 `teamB`에도 속한다면 user1로 실행:

```bash
chgrp teamB /home/user1/networks
```

검증:

```bash
ls -l /home/user1/networks
```

---

## 3. 🔍 검증 및 트러블슈팅

### 3-1. 필수 확인

```bash
ls -l <경로>
ls -ln <경로>
stat -c '%U:%G %u:%g %n' <경로>

id <사용자>
getent passwd <사용자>
getent group <그룹>
```

소유자·그룹 없는 파일:

```bash
find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
```

별도 파일시스템은 마운트포인트별로 확인한다.

---

### 3-2. 일반 사용자의 `chown` 실패

```bash
chown user2 myfile
```

일반 사용자는 소유자를 임의 변경할 수 없다.

```bash
sudo chown user2 myfile
```

그룹만 변경:

```bash
chgrp teamA myfile
```

---

### 3-3. `chgrp` 실패

```bash
ls -l myfile
id
```

확인:

- 자신이 파일 소유자인가?
- 변경할 그룹에 속해 있는가?
- 파일시스템이 읽기 전용인가?
- NFS·컨테이너 UID/GID 매핑 문제가 있는가?

---

### 3-4. 소유자가 숫자로 표시된다

```bash
ls -ln <경로>
getent passwd <UID>
getent group <GID>
```

계정·그룹 삭제 또는 NSS 조회 실패 여부를 확인한다.

---

### 3-5. 소유권 변경 후 특수 권한이 사라졌다

소유자나 그룹 변경 시 Set-UID·Set-GID 비트가 보안상 해제될 수 있다.

```bash
find /project -type f -perm /6000 -ls
```

패키지 파일:

```bash
pkg=$(rpm -qf --qf '%{NAME}\n' <파일>)
rpm -V "$pkg"
```

특수 권한을 임의 복구하지 말고 원래 패키지 또는 운영 정책을 확인한다.

---

> 📌 **핵심 요약**
> - 소유권은 UID·GID로 저장
> - 소유자 변경: `chown`
> - 그룹 변경: `chgrp`, `chown :group`
> - 일반 사용자는 소유자 변경 불가
> - 자신의 파일 그룹은 자신이 속한 그룹으로만 변경 가능
> - `-xdev`는 다른 파일시스템을 검색하지 않음
> - 관련: 🔐 허가권 (Permission) — chmod & rwx·UGO 모델 · 🔧 허가권 상세 (chmod & 8진수 · 심볼릭 표기) · 👥 소유권 & 특수 권한 (chown & chgrp & SUID · SGID · Sticky)
