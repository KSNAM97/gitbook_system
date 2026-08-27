# 🔗 마운트 & umount

> **Tag:** #Linux #Mount #umount #FileSystem #findmnt #blkid  
> **핵심 요약:** 마운트는 파일시스템을 Linux 디렉터리 트리에 연결하는 작업이다. 명령으로 수행한 마운트는 현재 셸에만 한정되는 것이 아니라 해제하거나 시스템을 재부팅할 때까지 유지되며, 재부팅 후에도 자동으로 연결하려면 `/etc/fstab` 등에 영구 설정을 구성한다.

---

## 1. 📖 개요 (Overview)

**마운트란?**

Windows가 일반적으로 드라이브 문자로 파일시스템을 연결하는 것과 달리 Linux는 하나의 디렉터리 트리에 파일시스템을 연결한다.

```text
파일시스템 장치: /dev/sdb1
마운트포인트:    /data
```

마운트:

```bash
mount /dev/sdb1 /data
```

이후 `/data`를 통해 `/dev/sdb1`의 파일시스템에 접근한다.

**마운트포인트는 반드시 빈 디렉터리여야 하는가?**

기술적으로 반드시 비어 있어야 하는 것은 아니다. 그러나 기존 파일이 있는 디렉터리에 마운트하면 기존 내용이 새 파일시스템 아래에 가려진다.

```text
/data에 old.txt 존재
→ /dev/sdb1을 /data에 마운트
→ old.txt가 보이지 않음
→ umount /data
→ old.txt가 다시 보임
```

데이터가 삭제된 것은 아니지만 운영 장애로 오인하기 쉬우므로 전용 빈 디렉터리를 권장한다.

**mount와 fstab의 차이는?**

```bash
mount /dev/sdb1 /data
```

- 즉시 적용
- 해제 또는 재부팅 전까지 유지
- 재부팅 후 자동 재연결 보장 없음

`/etc/fstab`:

- 부팅 시 또는 `mount -a` 실행 시 참조
- 영구 마운트 정책 관리
- UUID, 마운트포인트, 파일시스템 타입, 옵션 저장

표준 순서:

```text
수동 mount 테스트
→ 읽기·쓰기 확인
→ umount
→ fstab 등록
→ 검증
→ fstab를 이용해 mount
```

**“디렉터리에 30GB를 할당한다”는 의미는?**

일반 디렉터리 자체에 `mount` 명령으로 용량을 부여하는 것은 아니다.

```text
/dev/sdb1 → 30GiB 파일시스템
/data     → 마운트포인트
```

`/dev/sdb1`을 `/data`에 마운트하면 `/data`를 통해 해당 파일시스템의 용량을 사용하게 된다.

```bash
mount /dev/sdb1 /data
```

따라서 “`/data`에 30GB를 할당한다”는 표현은 일반적으로 다음 의미이다.

```text
30GiB 파티션 또는 LV 준비
→ 파일시스템 생성
→ /data에 마운트
```

실제 사용 가능한 용량은 파일시스템 메타데이터, 예약 공간 및 단위 차이 때문에 파티션의 표시 용량보다 작을 수 있다.

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 마운트 전 확인

```bash
lsblk -f
blkid /dev/sdb1
findmnt -S /dev/sdb1
```

마운트포인트 생성:

```bash
mkdir -p /data
```

기존 내용 확인:

```bash
ls -la /data
```

---

### 2-2. 기본 마운트

```bash
mount /dev/sdb1 /data
```

파일시스템 타입 명시:

```bash
mount -t xfs /dev/sdb1 /data
```

UUID 사용:

```bash
mount UUID="<uuid>" /data
```

LABEL 사용:

```bash
mount LABEL=DATA /data
```

읽기 전용:

```bash
mount -o ro /dev/sdb1 /data
```

옵션 지정:

```bash
mount -o rw,nosuid,nodev,noexec /dev/sdb1 /data
```

> `noexec`는 일반적인 직접 실행을 제한하지만 모든 실행 상황을 완전히 차단하는 보안 경계는 아니다.

---

### 2-3. 마운트 상태 확인

```bash
findmnt /data
findmnt -S /dev/sdb1
df -Th /data
lsblk -f
mountpoint /data
```

현재 옵션:

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /data
```

읽기·쓰기 테스트:

```bash
touch /data/.mount-test
ls -l /data/.mount-test
rm /data/.mount-test
```

---

### 2-4. 마운트 해제

마운트포인트 기준:

```bash
umount /data
```

장치 기준:

```bash
umount /dev/sdb1
```

확인:

```bash
findmnt /data
mountpoint /data
```

> 명령은 `unmount`가 아니라 `umount`이다.

---

### 2-5. CD/DVD 마운트

```bash
mkdir -p /media/cdrom
lsblk -f
ls -l /dev/sr0
mount -o ro /dev/sr0 /media/cdrom
```

검증:

```bash
findmnt /media/cdrom
ls -la /media/cdrom
```

해제:

```bash
cd /
umount /media/cdrom
```

---

### 2-6. 기존 데이터가 있는 마운트포인트 처리

새 파일시스템을 바로 마운트하면 기존 파일이 가려진다.

임시 마운트:

```bash
mkdir -p /mnt/newdata
mount /dev/sdb1 /mnt/newdata
```

기존 데이터 복사:

```bash
rsync -aHAX --numeric-ids /data/ /mnt/newdata/
```

비교:

```bash
du -sh /data /mnt/newdata
```

최종 동기화:

```bash
rsync -aHAX --numeric-ids --delete /data/ /mnt/newdata/
```

> 애플리케이션 일관성, SELinux 컨텍스트, 열린 파일, 권한과 ACL을 함께 확인한다.

---

### 2-7. 여러 파일시스템 통합 마운트 실습

준비된 파일시스템:

```text
/dev/sdb1 → ext4, 30GiB, LABEL=SOL
/dev/sdb2 → XFS,  30GiB, LABEL=USER
/dev/sdb5 → ext4, 10GiB, LABEL=CISCO
/dev/sdb6 → ext4, 10GiB, LABEL=GUEST
/dev/sdb7 → ext4, 10GiB, LABEL=NOBODY
```

마운트 목표:

```text
/dev/sdb1 → /sol
/dev/sdb5 → /cisco
/dev/sdb2 → /linux/user
/dev/sdb6 → /linux/guest
/dev/sdb7 → /linux/nobody
```

사전 확인:

```bash
lsblk -f /dev/sdb
findmnt -S /dev/sdb1
findmnt -S /dev/sdb2
findmnt -S /dev/sdb5
findmnt -S /dev/sdb6
findmnt -S /dev/sdb7
```

마운트포인트 생성:

```bash
mkdir -p /sol
mkdir -p /cisco
mkdir -p /linux/user
mkdir -p /linux/guest
mkdir -p /linux/nobody
```

기존 내용 확인:

```bash
ls -la /sol
ls -la /cisco
ls -la /linux/user
ls -la /linux/guest
ls -la /linux/nobody
```

장치명으로 마운트:

```bash
mount /dev/sdb1 /sol
mount /dev/sdb5 /cisco
mount /dev/sdb2 /linux/user
mount /dev/sdb6 /linux/guest
mount /dev/sdb7 /linux/nobody
```

LABEL을 사용할 수도 있다.

```bash
mount LABEL=SOL /sol
mount LABEL=CISCO /cisco
mount LABEL=USER /linux/user
mount LABEL=GUEST /linux/guest
mount LABEL=NOBODY /linux/nobody
```

> 장치명 방식과 LABEL 방식 중 하나만 사용한다. 이미 마운트된 장치를 중복으로 마운트하지 않는다.

검증:

```bash
findmnt /sol
findmnt /cisco
findmnt /linux/user
findmnt /linux/guest
findmnt /linux/nobody
```

전체 구조:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS /dev/sdb
df -Th /sol /cisco /linux/user /linux/guest /linux/nobody
```

읽기·쓰기 테스트:

```bash
touch /sol/.mount-test
touch /cisco/.mount-test
touch /linux/user/.mount-test
touch /linux/guest/.mount-test
touch /linux/nobody/.mount-test

rm /sol/.mount-test
rm /cisco/.mount-test
rm /linux/user/.mount-test
rm /linux/guest/.mount-test
rm /linux/nobody/.mount-test
```

해제:

```bash
cd /
umount /linux/nobody
umount /linux/guest
umount /linux/user
umount /cisco
umount /sol
```

> 부모·자식 경로에 각각 파일시스템을 마운트한다면 부모를 먼저 마운트하고 자식을 나중에 마운트한다. 해제할 때는 반대로 자식부터 해제한다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 진단

```bash
lsblk -f
blkid
findmnt
df -Th
```

특정 경로:

```bash
findmnt -T /data/file
```

---

### 3-2. `target is busy`

먼저 마운트포인트 밖으로 이동한다.

```bash
cd /
```

확인:

```bash
pwd
findmnt -R /data
fuser -vm /data
lsof /data
```

정상 종료:

```bash
systemctl stop <관련서비스>
kill <PID>
umount /data
```

Lazy unmount:

```bash
umount -l /data
```

강제 해제:

```bash
umount -f /data
```

> Lazy·강제 해제는 일반적인 첫 번째 해결책으로 사용하지 않는다.

---

### 3-3. `wrong fs type`, `bad superblock`

```bash
lsblk -f /dev/sdb1
blkid /dev/sdb1
file -s /dev/sdb1
dmesg --ctime | tail -n 50
```

가능한 원인:

- 파일시스템이 생성되지 않음
- 잘못된 파일시스템 타입 지정
- 파일시스템 손상
- 사용자 공간 도구 미설치
- 암호화·LVM·RAID 하위 장치를 직접 마운트
- 잘못된 장치 선택

XFS 검사:

```bash
xfs_repair -n /dev/sdb1
```

ext4 검사:

```bash
e2fsck -fn /dev/sdb1
```

---

### 3-4. 마운트 후 기존 파일이 사라졌다

```bash
findmnt /data
umount /data
ls -la /data
```

기존 파일이 다시 보이면 마운트에 의해 가려졌던 것이다.

---

### 3-5. 마운트는 됐지만 쓰기가 안 된다

```bash
findmnt -no OPTIONS /data
ls -ld /data
getfacl /data
ls -ldZ /data
```

확인 항목:

- `ro` 옵션
- 파일시스템 오류로 읽기 전용 전환
- 디렉터리 권한
- ACL
- SELinux
- 디스크 공간
- inode 부족

```bash
df -h /data
df -i /data
dmesg --ctime | tail -n 50
```

---

### 3-6. 디스크·파티션·파일시스템·마운트 상태 구분

1단계: 디스크 확인

```bash
lsblk /dev/sdb
```

2단계: 파티션 확인

```bash
fdisk -l /dev/sdb
lsblk /dev/sdb
```

3단계: 파일시스템 확인

```bash
lsblk -f /dev/sdb1
blkid /dev/sdb1
file -s /dev/sdb1
```

4단계: 마운트 확인

```bash
findmnt -S /dev/sdb1
findmnt /data
mountpoint /data
```

5단계: 사용량 확인

```bash
df -Th /data
df -i /data
```

상태 비교:

| 상태 | `lsblk` | `lsblk -f` | `findmnt`·`df` |
|---|---|---|---|
| 디스크만 연결 | 디스크 표시 | 파일시스템 없음 | 표시 안 됨 |
| 파티션 생성 | 디스크·파티션 표시 | 파일시스템 없음 | 표시 안 됨 |
| 포맷 완료 | 디스크·파티션 표시 | FSTYPE·UUID 표시 | 마운트 전에는 표시 안 됨 |
| 마운트 완료 | 마운트포인트 표시 | FSTYPE·UUID 표시 | 파일시스템 표시 |

> `df`에 보이지 않는다는 이유만으로 디스크가 인식되지 않았다고 판단하지 않는다.

> 📌 **핵심 요약**
> - mount는 파일시스템을 디렉터리 트리에 연결
> - 디렉터리에 용량을 직접 부여하는 것이 아니라 파일시스템을 연결하는 것
> - 수동 mount는 재부팅 후 유지되지 않음
> - 기존 파일이 있는 경로에 마운트하면 내용이 가려짐
> - busy 오류는 `cd /` → `fuser` → 정상 종료 → `umount`
> - 신규 디스크 확인은 `lsblk`부터 시작
> - 관련: 8-1. 💽 디스크 타입 & 파티션 구조 · 8-2. 🗂️ 파일 시스템 & Format · 8-4. ⚓ Automount
