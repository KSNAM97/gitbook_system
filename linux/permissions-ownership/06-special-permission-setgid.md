# 특수권한 Set-GID — 소유 그룹 자동 상속 (2XXX)

> **Tag:** #Linux #SetGID #SGID #GroupInheritance #Collaboration  
> **핵심 요약:** Set-GID 디렉터리에서 새로 생성된 파일과 하위 디렉터리는 생성자의 기본 그룹이 아니라 상위 디렉터리의 소유 그룹을 상속한다. 다만 그룹 쓰기 권한까지 자동으로 보장하지 않으므로 `umask` 또는 기본 ACL을 함께 설계해야 한다.

---

## 1. 개요 (Overview)

**일반 디렉터리에서 파일을 생성하면?**

일반적으로 새 파일의 그룹은 생성자의 기본 그룹으로 설정된다.

```text
user1의 기본 그룹 = user1
user2의 기본 그룹 = user2
```

결과:

```text
user1이 생성 → user1:user1
user2가 생성 → user2:user2
```

**Set-GID 디렉터리에서는?**

상위 디렉터리의 그룹이 `teamA`라면:

```text
user1이 생성 → user1:teamA
user2가 생성 → user2:teamA
```

새 하위 디렉터리는 Linux에서 일반적으로 Set-GID 비트도 상속한다.

**Set-GID만 설정하면 공동 수정이 가능한가?**

아니다.

Set-GID는 그룹 소유권을 통일하지만 그룹 쓰기 권한은 `umask`에 따라 달라진다.

```text
umask 0002 → 파일 664, 디렉터리 775
umask 0022 → 파일 644, 디렉터리 755
```

`umask 0022`라면 새 파일의 그룹이 `teamA`여도 Group 권한이 `r--`이므로 다른 팀원이 수정하지 못한다.

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 그룹과 사용자 준비

```bash
groupadd solGroup

useradd -md /homesol/userSol1 userSol1
useradd -md /homesol/userSol2 userSol2
useradd -md /homesol/userSol3 userSol3

usermod -aG solGroup userSol1
usermod -aG solGroup userSol2
usermod -aG solGroup userSol3
```

검증:

```bash
id userSol1
id userSol2
id userSol3
```

그룹 추가 후 사용자는 재로그인해야 한다.

### Step 2. Set-GID 공유 디렉터리

```bash
mkdir -p /homesol/sol_tmp1
chown root:solGroup /homesol/sol_tmp1
chmod 2770 /homesol/sol_tmp1
```

검증:

```bash
ls -ld /homesol/sol_tmp1
```

예상:

```text
drwxrws--- root solGroup /homesol/sol_tmp1
```

### Step 3. 그룹 상속 확인

`userSol1`:

```bash
umask 0002
touch /homesol/sol_tmp1/userSol1F
mkdir /homesol/sol_tmp1/userSol1D
ls -l /homesol/sol_tmp1
```

예상:

```text
-rw-rw-r-- userSol1 solGroup userSol1F
drwxrwsr-x userSol1 solGroup userSol1D
```

### Step 4. 그룹 구성원의 삭제 정책

`2770` 디렉터리에서 그룹 구성원은 상위 디렉터리에 `w+x`가 있으므로 다른 그룹 구성원이 만든 파일도 삭제하거나 이름 변경할 수 있다.

```bash
mv /homesol/sol_tmp1/userSol1F /homesol/sol_tmp1/renamed
rm /homesol/sol_tmp1/renamed
```

서로의 파일 삭제를 막으려면 Sticky-bit를 추가한다.

```bash
chmod 3770 /homesol/sol_tmp1
```

### Step 5. 기본 ACL 사용

팀 파일에 그룹 쓰기 권한을 안정적으로 적용하려면 기본 ACL을 사용할 수 있다.

```bash
setfacl -m g:solGroup:rwx /homesol/sol_tmp1
setfacl -d -m g:solGroup:rwx /homesol/sol_tmp1
setfacl -d -m m::rwx /homesol/sol_tmp1
```

확인:

```bash
getfacl /homesol/sol_tmp1
```

---

## 3. 검증 및 트러블슈팅

Set-GID 디렉터리 검색:

```bash
find / -xdev -type d -perm -2000 -ls 2>/dev/null
```

### 그룹은 상속됐는데 수정이 안 된다

확인:

```bash
ls -l <파일>
umask
getfacl <파일>
```

파일이 다음과 같다면:

```text
-rw-r--r-- userSol1 solGroup file
```

Group에는 `w`가 없으므로 다른 사용자가 수정할 수 없다.

일회성 조치:

```bash
chmod g+w file
```

지속 정책:

```bash
umask 0002
```

또는 기본 ACL을 사용한다.

### Set-GID를 설정했는데 그룹이 상속되지 않는다

확인:

```bash
ls -ld <디렉터리>
stat <디렉터리>
```

다음 표시가 있어야 한다.

```text
drwxrws---
```

추가 원인:

- 파일시스템 또는 네트워크 스토리지 정책
- ACL
- 애플리케이션이 생성 후 소유권 변경
- 컨테이너 UID/GID 매핑

>  **핵심 요약**
> - Set-GID 숫자: `2XXX`
> - 디렉터리 Group의 `x` 자리에 `s`
> - 팀 공유 디렉터리: `2770`
> - 타인 파일 삭제도 막으려면 `3770`
> - 공동 수정에는 `umask 0002` 또는 기본 ACL 검토
> - 관련: 6-5.   Umask — 기본 권한 마스크 (User Mask) · 6-7.  특수권한 Sticky-bit — 공유 디렉터리 삭제 방지 (1XXX)
