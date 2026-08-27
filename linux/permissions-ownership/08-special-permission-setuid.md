# 🪪 특수권한 Set-UID — 실행 중 소유자 권한 위임 (4XXX)

> **Tag:** #Linux #SetUID #SUID #Security #Privilege  
> **핵심 요약:** Set-UID가 설정된 실행 파일은 실행한 사용자의 실제 UID는 유지하면서 실행 파일 소유자의 유효 UID로 동작한다. 파일 소유자가 root인 경우에만 EUID 0으로 동작하며, 프로그램 내부에서 허용한 작업을 제한적으로 수행한다.

---

## 1. 📖 개요 (Overview)

**Set-UID는 어떻게 동작하는가?**

일반 실행:

```text
실행 사용자의 UID
→ 프로세스의 실제 UID와 유효 UID로 사용
```

Set-UID 실행:

```text
실제 UID(RUID) = 실행한 사용자
유효 UID(EUID) = 실행 파일 소유자
```

예:

```text
실행 사용자: guest
파일 소유자: root
파일 권한:   -rwsr-xr-x
```

이 경우:

```text
RUID = guest의 UID
EUID = 0(root)
```

> Set-UID 파일 소유자가 일반 사용자라면 EUID도 그 사용자의 UID가 된다. Set-UID가 항상 root 권한을 의미하는 것은 아니다.

**대표적인 예는?**

```bash
ls -l /usr/bin/passwd
```

일반적인 결과:

```text
-rwsr-xr-x. 1 root root ... /usr/bin/passwd
```

일반 사용자는 `/etc/shadow`를 직접 수정할 수 없다.

```bash
ls -l /etc/shadow
```

`passwd` 프로그램은 실행 중 필요한 권한을 사용하지만 내부 검증을 통해 일반 사용자가 자신의 비밀번호만 변경하도록 제한한다.

**Set-UID는 파일 쓰기 권한을 직접 부여하는가?**

아니다.

- 일반 사용자에게 `/etc/shadow`의 `w`를 부여하지 않음
- 프로세스의 EUID를 실행 파일 소유자로 설정
- 프로그램 내부 로직이 허용한 작업만 수행
- 프로그램에 취약점이 있으면 권한 상승 위험이 발생할 수 있음

따라서 root 소유 Set-UID 실행 파일은 최소화하고 정기적으로 감사한다.

**숫자와 문자 표기는?**

숫자:

```text
4XXX
```

예:

```bash
chmod 4755 executable
```

표시:

```text
-rwsr-xr-x
```

Owner 실행 권한이 없는 경우:

```text
-rwSr-xr-x
```

```text
s → Set-UID와 Owner 실행 권한 모두 있음
S → Set-UID는 있으나 Owner 실행 권한 없음
```

## 2. 🛠️ 설정 및 안전한 확인

Set-UID 설정:

```bash
chmod u+s executable
```

숫자 방식:

```bash
chmod 4755 executable
```

해제:

```bash
chmod u-s executable
```

확인:

```bash
ls -l executable
stat executable
```

> 임의의 프로그램에 root 소유 Set-UID를 설정하면 심각한 권한 상승 취약점이 될 수 있다. 운영 시스템에서는 검증되지 않은 파일에 Set-UID를 설정하지 않는다.

시스템의 `passwd` 권한 확인:

```bash
ls -l /usr/bin/passwd
stat /usr/bin/passwd
```

소유 패키지 확인:

```bash
rpm -qf /usr/bin/passwd
```

패키지 이름만 확인:

```bash
pkg=$(rpm -qf --qf '%{NAME}\n' /usr/bin/passwd)
printf '%s\n' "$pkg"
```

> `/usr/bin/passwd`의 소유 패키지는 배포판과 버전에 따라 확인해야 한다. Rocky/RHEL 계열에서는 `passwd` 패키지인 경우가 많으므로 `shadow-utils`로 고정하지 않는다.

---

## 3. 🔍 검증 및 트러블슈팅

### 3-1. Set-UID 파일 검색

현재 파일시스템:

```bash
find / -xdev -type f -perm -4000 -ls 2>/dev/null
```

Set-UID·Set-GID 파일:

```bash
find / -xdev -type f -perm /6000 -ls 2>/dev/null
```

`-xdev`는 별도 마운트된 파일시스템을 검색하지 않는다. 중요 데이터 볼륨은 마운트포인트별로 반복 검사한다.

```bash
find /home -xdev -type f -perm /6000 -ls 2>/dev/null
find /data -xdev -type f -perm /6000 -ls 2>/dev/null
```

---

### 3-2. Set-UID가 동작하지 않는다

마운트 옵션:

```bash
findmnt -T <실행파일>
```

파일 권한:

```bash
stat <실행파일>
```

SELinux:

```bash
ls -lZ <실행파일>
ausearch -m AVC -ts recent
```

가능한 원인:

- 파일시스템이 `nosuid`
- Set-UID 비트 없음
- 실행 권한 없음
- 스크립트 파일에 설정
- 컨테이너·사용자 네임스페이스 제한
- SELinux 차단
- 파일 소유자가 예상과 다름

Linux에서는 보안상 Set-UID 스크립트가 일반적으로 기대한 방식으로 동작하지 않는다.

---

### 3-3. `/usr/bin/passwd` 권한을 잘못 변경했다

현재 권한:

```bash
ls -l /usr/bin/passwd
```

소유 패키지 확인:

```bash
pkg=$(rpm -qf --qf '%{NAME}\n' /usr/bin/passwd)
printf 'package=%s\n' "$pkg"
```

패키지 검증:

```bash
rpm -V "$pkg"
```

패키지 재설치:

```bash
dnf reinstall "$pkg"
```

복구 확인:

```bash
ls -l /usr/bin/passwd
rpm -V "$pkg"
```

일반 사용자의 비밀번호 변경 테스트:

```bash
sudo -u <사용자> passwd
```

> 실제 사용자 비밀번호에 영향을 주므로 테스트 계정에서 수행한다.

---

### 3-4. 알 수 없는 Set-UID 파일을 발견했다

```bash
rpm -qf <파일>
stat <파일>
sha256sum <파일>
```

패키지 파일:

```bash
pkg=$(rpm -qf --qf '%{NAME}\n' <파일>)
rpm -V "$pkg"
```

패키지에 속하지 않는 경우:

```bash
rpm -qf <파일>
```

예상:

```text
file ... is not owned by any package
```

조치:

1. 생성 경위 확인
2. 실행·접근 로그 확인
3. 네트워크와 프로세스 조사
4. 필요하면 격리
5. Set-UID 제거 전 서비스 영향 확인
6. 침해사고 대응 절차 수행

특수 권한 제거:

```bash
chmod u-s <파일>
```

> 출처를 알 수 없는 root 소유 Set-UID 파일은 보안 사고 지표일 수 있다.

---

> 📌 **핵심 요약**
> - Set-UID 숫자: `4XXX`
> - EUID는 실행 파일 소유자 UID
> - 파일 소유자가 root일 때만 EUID 0
> - root 소유 Set-UID 파일은 최소화·감사
> - `/usr/bin/passwd` 복구 전 `rpm -qf`로 실제 패키지 확인
> - 관련: 6-4. 👥 소유권 & 특수 권한 (chown & chgrp & SUID · SGID · Sticky) · 6-9. 🧩 허가권·소유권 통합 정리 · 6-10.  🚑 허가권·소유권 트러블슈팅 치트시트
