# 🏗️  종합실습 HDD 추가 → 파티션 → 포맷 → 마운트 → Automount

> **Tag:** #Linux #Lab #Partition #GPT #XFS #ext4 #Mount #fstab #Automount  
> **핵심 요약:** 신규 HDD 3개를 추가해 `/GIT`, `/homeSK`, `/homeLG` 볼륨을 구성하는 종합 실습이다. 디스크 식별 → GPT 파티션 생성 → 파일시스템 포맷 → 임시 마운트 → `/etc/fstab` 영구화 → 계정·그룹·SELinux 연동 → 재부팅 검증 순서로 진행한다.

---

## 1. 🎯 실습 목표 (Scenario)

### 1-1. 요구사항

신규 디스크:

```text
/dev/sdb
/dev/sdc
/dev/sdd
```

구성 목표:

| 장치 | 파일시스템 | 마운트포인트 | 용도 |
|---|---|---|---|
| `/dev/sdb1` | XFS | `/GIT` | 개발팀 공유 데이터 |
| `/dev/sdc1` | ext4 | `/homeSK` | SK팀 사용자 홈 |
| `/dev/sdd1` | XFS | `/homeLG` | LG팀 사용자 홈 |

추가 요구사항:

- 모든 디스크는 GPT 파티션 테이블 사용
- 각 디스크 전체 용량을 파티션 하나로 구성
- 재부팅 후 자동 마운트
- `/homeLG`에 `mUser1`, `mUser2`, `mUser3` 계정 홈 생성
- LG팀 공유 디렉터리는 Set-GID로 그룹 상속
- 사용자 개별 홈과 팀 공유 영역 분리
- fstab 수정 후 재부팅 전 검증
- SELinux Enforcing 환경에서도 홈 디렉터리 정상 사용

---

### 1-2. 예상 최종 구조

```text
/dev/sdb
└─/dev/sdb1  xfs   /GIT

/dev/sdc
└─/dev/sdc1  ext4  /homeSK

/dev/sdd
└─/dev/sdd1  xfs   /homeLG
                         ├─ mUser1
                         ├─ mUser2
                         ├─ mUser3
                         └─ shared
```

권한 정책:

```text
/homeLG/mUser1 → mUser1 개인 홈
/homeLG/mUser2 → mUser2 개인 홈
/homeLG/mUser3 → mUser3 개인 홈
/homeLG/shared → lgTeam 팀 공유
```

> `/homeLG` 전체를 `2770 root:lgTeam`으로 설정하면 팀 구성원이 다른 사용자의 홈 디렉터리 이름을 변경하거나 삭제할 위험이 있다. 개인 홈의 상위 디렉터리는 root가 관리하고, 협업 공간은 `/homeLG/shared`처럼 별도로 구성한다.

---

## 2. 🛠️ 단계별 실습 (Configuration)

### STEP 0. 작업 전 주의사항

이 실습의 파티션·포맷 명령은 다음 디스크의 기존 데이터를 파괴한다.

```text
/dev/sdb
/dev/sdc
/dev/sdd
```

운영 서버에서는 콘솔 접근과 백업을 확보하고 작업한다.

현재 상태 기록:

```bash
date
hostnamectl
lsblk -f
findmnt
blkid
```

결과 백업:

```bash
mkdir -p /root/storage-lab-backup

lsblk -f \
  > /root/storage-lab-backup/lsblk.before.txt

findmnt \
  > /root/storage-lab-backup/findmnt.before.txt

blkid \
  > /root/storage-lab-backup/blkid.before.txt
```

---

### STEP 1. 신규 디스크 식별

전체 디스크 확인:

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,WWN,TRAN,ROTA
```

대상 디스크 상세 확인:

```bash
lsblk -f /dev/sdb
lsblk -f /dev/sdc
lsblk -f /dev/sdd
```

기존 파티션 테이블:

```bash
fdisk -l /dev/sdb
fdisk -l /dev/sdc
fdisk -l /dev/sdd
```

기존 시그니처를 삭제하지 않고 확인:

```bash
wipefs -n /dev/sdb
wipefs -n /dev/sdc
wipefs -n /dev/sdd
```

마운트 여부:

```bash
findmnt -S /dev/sdb
findmnt -S /dev/sdc
findmnt -S /dev/sdd
```

LVM·RAID·Swap 확인:

```bash
pvs
vgs
lvs

cat /proc/mdstat
swapon --show
```

확인 체크리스트:

```text
[ ] OS 디스크가 아님
[ ] 마운트되지 않음
[ ] Swap으로 사용되지 않음
[ ] LVM PV가 아님
[ ] RAID 구성원이 아님
[ ] 모델·일련번호가 작업 대상과 일치
[ ] 삭제해도 되는 빈 디스크
```

> `/dev/sdb`라는 이름만 보고 작업하지 않는다. 실제 환경에서는 장치명이 달라질 수 있으므로 모델과 일련번호를 함께 확인한다.

---

### STEP 2. GPT 파티션 생성

#### `/dev/sdb`

```bash
parted -s /dev/sdb mklabel gpt

parted -s -a optimal /dev/sdb \
  mkpart primary 1MiB 100%
```

#### `/dev/sdc`

```bash
parted -s /dev/sdc mklabel gpt

parted -s -a optimal /dev/sdc \
  mkpart primary 1MiB 100%
```

#### `/dev/sdd`

```bash
parted -s /dev/sdd mklabel gpt

parted -s -a optimal /dev/sdd \
  mkpart primary 1MiB 100%
```

커널에 파티션 테이블 재인식 요청:

```bash
partprobe /dev/sdb
partprobe /dev/sdc
partprobe /dev/sdd

udevadm settle
```

검증:

```bash
lsblk /dev/sdb /dev/sdc /dev/sdd
```

파티션 테이블 확인:

```bash
parted /dev/sdb print
parted /dev/sdc print
parted /dev/sdd print
```

예상 구조:

```text
sdb
└─sdb1

sdc
└─sdc1

sdd
└─sdd1
```

---

### STEP 3. 파일시스템 생성

> 포맷 직전 대상 장치를 다시 확인한다.

```bash
lsblk -f /dev/sdb /dev/sdc /dev/sdd
```

마운트 여부 재확인:

```bash
findmnt -S /dev/sdb1
findmnt -S /dev/sdc1
findmnt -S /dev/sdd1
```

기존 시그니처 확인:

```bash
wipefs -n /dev/sdb1
wipefs -n /dev/sdc1
wipefs -n /dev/sdd1
```

#### `/GIT`용 XFS

```bash
mkfs.xfs -L GITDATA /dev/sdb1
```

#### `/homeSK`용 ext4

```bash
mkfs.ext4 -L HOMESK /dev/sdc1
```

#### `/homeLG`용 XFS

```bash
mkfs.xfs -L HOMELG /dev/sdd1
```

검증:

```bash
lsblk -f /dev/sdb /dev/sdc /dev/sdd
```

UUID·타입 확인:

```bash
blkid /dev/sdb1
blkid /dev/sdc1
blkid /dev/sdd1
```

예상:

```text
/dev/sdb1 TYPE="xfs"  LABEL="GITDATA"
/dev/sdc1 TYPE="ext4" LABEL="HOMESK"
/dev/sdd1 TYPE="xfs"  LABEL="HOMELG"
```

---

### STEP 4. 마운트포인트 생성

```bash
mkdir -p /GIT
mkdir -p /homeSK
mkdir -p /homeLG
```

기존 내용 확인:

```bash
ls -la /GIT
ls -la /homeSK
ls -la /homeLG
```

권한 확인:

```bash
ls -ld /GIT /homeSK /homeLG
```

> 마운트포인트가 반드시 비어 있어야 하는 기술적 제약은 없지만, 기존 파일이 있으면 마운트 후 가려진다. 신규 전용 디렉터리를 사용하는 것이 안전하다.

---

### STEP 5. 임시 마운트 테스트

```bash
mount /dev/sdb1 /GIT
mount /dev/sdc1 /homeSK
mount /dev/sdd1 /homeLG
```

마운트 상태:

```bash
findmnt /GIT
findmnt /homeSK
findmnt /homeLG
```

통합 확인:

```bash
df -Th | grep -E '/GIT|/homeSK|/homeLG'
```

블록 장치 구조:

```bash
lsblk -f
```

예상:

```text
/dev/sdb1 xfs  → /GIT
/dev/sdc1 ext4 → /homeSK
/dev/sdd1 xfs  → /homeLG
```

읽기·쓰기 테스트:

```bash
touch /GIT/.mount-test
touch /homeSK/.mount-test
touch /homeLG/.mount-test

ls -l \
  /GIT/.mount-test \
  /homeSK/.mount-test \
  /homeLG/.mount-test
```

테스트 파일 삭제:

```bash
rm \
  /GIT/.mount-test \
  /homeSK/.mount-test \
  /homeLG/.mount-test
```

---

### STEP 6. `/etc/fstab` 영구 등록

UUID 변수 확인:

```bash
blkid -s UUID -o value /dev/sdb1
blkid -s UUID -o value /dev/sdc1
blkid -s UUID -o value /dev/sdd1
```

fstab 백업:

```bash
cp -a /etc/fstab \
  "/etc/fstab.bak.$(date +%F-%H%M%S)"
```

편집:

```bash
sudoedit /etc/fstab
```

등록 형식:

```fstab
# GIT 개발팀 공유 볼륨
UUID=<sdb1-xfs-uuid>   /GIT      xfs   defaults  0 0

# SK팀 홈 볼륨
UUID=<sdc1-ext4-uuid>  /homeSK   ext4  defaults  0 2

# LG팀 홈 볼륨
UUID=<sdd1-xfs-uuid>   /homeLG   xfs   defaults  0 0
```

주의:

- 예시 UUID를 그대로 사용하지 않는다.
- `blkid`에서 확인한 실제 UUID를 입력한다.
- XFS의 마지막 필드는 일반적으로 `0`
- ext4 일반 데이터 파일시스템은 fsck 순서로 `2`를 사용할 수 있음
- `/homeSK`, `/homeLG`는 사용자 로그인에 필요한 중요 볼륨이므로 무조건 `nofail`을 적용하지 않음

---

### STEP 7. fstab 검증

문법·구조 검증:

```bash
findmnt --verify --verbose
```

systemd 반영:

```bash
systemctl daemon-reload
```

현재 수동 마운트 해제:

```bash
umount /GIT
umount /homeSK
umount /homeLG
```

해제 확인:

```bash
findmnt /GIT
findmnt /homeSK
findmnt /homeLG
```

fstab 전체 마운트:

```bash
mount -a
```

실제 상태 확인:

```bash
findmnt /GIT
findmnt /homeSK
findmnt /homeLG
```

```bash
df -Th | grep -E '/GIT|/homeSK|/homeLG'
```

fstab 항목별 테스트:

```bash
umount /GIT
mount /GIT
findmnt /GIT
```

다시 전체 마운트:

```bash
mount -a
```

> `mount -a`가 출력 없이 끝났더라도 `findmnt`로 실제 장치, 파일시스템 타입, 마운트포인트가 맞는지 확인한다.

---

### STEP 8. 그룹 구성

개발팀 그룹:

```bash
groupadd gitTeam
```

SK팀 그룹:

```bash
groupadd skTeam
```

LG팀 그룹:

```bash
groupadd lgTeam
```

확인:

```bash
getent group gitTeam
getent group skTeam
getent group lgTeam
```

---

### STEP 9. 사용자 계정 생성

LG팀 계정:

```bash
useradd -m -d /homeLG/mUser1 -G lgTeam mUser1
useradd -m -d /homeLG/mUser2 -G lgTeam mUser2
useradd -m -d /homeLG/mUser3 -G lgTeam mUser3
```

비밀번호 설정:

```bash
passwd mUser1
passwd mUser2
passwd mUser3
```

계정 확인:

```bash
getent passwd mUser1
getent passwd mUser2
getent passwd mUser3
```

그룹 확인:

```bash
id mUser1
id mUser2
id mUser3
```

홈 디렉터리 확인:

```bash
ls -ld \
  /homeLG/mUser1 \
  /homeLG/mUser2 \
  /homeLG/mUser3
```

예상:

```text
/homeLG/mUser1 → mUser1 소유
/homeLG/mUser2 → mUser2 소유
/homeLG/mUser3 → mUser3 소유
```

> 홈 디렉터리 생성 전에 `/homeLG` 파일시스템의 fstab 등록과 마운트를 완료해야 한다. 계정을 먼저 만들고 나중에 파일시스템을 마운트하면 기존 홈 내용이 마운트 아래에 가려질 수 있다.

---

### STEP 10. 팀 공유 디렉터리 구성

#### `/GIT` 개발팀 공유

```bash
chown root:gitTeam /GIT
chmod 2770 /GIT
```

확인:

```bash
ls -ld /GIT
```

예상:

```text
drwxrws--- root gitTeam /GIT
```

#### `/homeLG/shared` LG팀 공유

```bash
mkdir -p /homeLG/shared
chown root:lgTeam /homeLG/shared
chmod 2770 /homeLG/shared
```

확인:

```bash
ls -ld /homeLG/shared
```

예상:

```text
drwxrws--- root lgTeam /homeLG/shared
```

Set-GID 효과:

```text
/homeLG/shared 아래에서 생성된 객체
→ 상위 디렉터리의 lgTeam 그룹 상속
```

그룹 공동 수정에는 Group 쓰기 권한도 필요하다. 사용자 세션에서 다음 정책을 검토한다.

```bash
umask 0002
```

기본 ACL 방식:

```bash
setfacl -m g:lgTeam:rwx /homeLG/shared
setfacl -d -m g:lgTeam:rwx /homeLG/shared
setfacl -d -m m::rwx /homeLG/shared
```

확인:

```bash
getfacl /homeLG/shared
```

> Set-GID는 그룹 소유권을 상속하지만 새 파일의 그룹 쓰기 권한까지 자동으로 보장하지 않는다. `umask` 또는 기본 ACL을 함께 설계한다.

---

### STEP 11. SELinux 사용자 홈 경로 설정

SELinux 상태 확인:

```bash
getenforce
```

`/homeLG`와 `/homeSK`를 사용자 홈 루트로 사용할 경우 `/home`의 SELinux 컨텍스트 규칙을 복제할 수 있다.

`semanage` 확인:

```bash
command -v semanage
```

없으면 Rocky Linux에서 관련 패키지 설치:

```bash
dnf install -y policycoreutils-python-utils
```

`/homeLG`에 `/home` 규칙 복제:

```bash
semanage fcontext -a -e /home /homeLG
restorecon -RFv /homeLG
```

`/homeSK`도 홈 루트로 사용할 경우:

```bash
semanage fcontext -a -e /home /homeSK
restorecon -RFv /homeSK
```

확인:

```bash
ls -ldZ /homeLG
ls -ldZ /homeLG/mUser1
ls -ldZ /homeSK
```

> 기존에 같은 경로에 사용자 정의 SELinux 규칙이 있으면 `-a`가 실패할 수 있다. `semanage fcontext -l`로 기존 규칙을 확인하고 필요하면 `-m`을 사용한다.

---

### STEP 12. 계정·공유 디렉터리 실제 검증

`mUser1` 홈 로그인 확인:

```bash
su - mUser1
```

현재 위치:

```bash
pwd
```

예상:

```text
/homeLG/mUser1
```

개인 파일 생성:

```bash
touch ~/private-file
ls -l ~/private-file
```

공유 디렉터리 파일 생성:

```bash
touch /homeLG/shared/mUser1-file
ls -l /homeLG/shared/mUser1-file
```

종료:

```bash
exit
```

`mUser2`로 확인:

```bash
su - mUser2
```

읽기·수정 테스트:

```bash
ls -l /homeLG/shared
echo "team data" >> /homeLG/shared/mUser1-file
```

종료:

```bash
exit
```

그룹이 적용되지 않으면:

```bash
id mUser2
getfacl /homeLG/shared
ls -l /homeLG/shared/mUser1-file
```

---

### STEP 13. 최종 검증

디스크·파티션·파일시스템:

```bash
lsblk -f
```

마운트:

```bash
findmnt /GIT
findmnt /homeSK
findmnt /homeLG
```

파일시스템 타입과 용량:

```bash
df -Th | grep -E '/GIT|/homeSK|/homeLG'
```

fstab:

```bash
findmnt --verify --verbose
grep -E '/GIT|/homeSK|/homeLG' /etc/fstab
```

계정:

```bash
for user in mUser1 mUser2 mUser3; do
  id "$user"
  getent passwd "$user"
done
```

홈:

```bash
ls -ld /homeLG/mUser*
```

공유 디렉터리:

```bash
stat -c '%A %a %U:%G %n' \
  /GIT \
  /homeLG/shared
```

SELinux:

```bash
ls -ldZ /homeLG /homeLG/mUser* /homeLG/shared
```

---

### STEP 14. 재부팅 검증

재부팅 전에 마지막 검증:

```bash
findmnt --verify --verbose
mount -a
```

fstab 백업 존재 확인:

```bash
ls -l /etc/fstab*
```

콘솔 또는 VM 관리 화면 접근 수단을 확인한 후:

```bash
reboot
```

재접속 후:

```bash
lsblk -f
```

```bash
findmnt /GIT
findmnt /homeSK
findmnt /homeLG
```

```bash
df -Th | grep -E '/GIT|/homeSK|/homeLG'
```

사용자 로그인:

```bash
su - mUser1
pwd
exit
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 🚨 시나리오 1. 새 디스크가 `lsblk`에 없다

VMware 또는 하이퍼바이저에서 디스크가 실제 연결되었는지 확인한다.

SCSI 재검색:

```bash
for host in /sys/class/scsi_host/host*; do
  echo "- - -" > "$host/scan"
done
```

확인:

```bash
udevadm settle
lsblk
dmesg --ctime | tail -n 50
```

> `partprobe`는 신규 하드웨어 검색용 명령이 아니다.

---

### 🚨 시나리오 2. 파티션 생성 후 `/dev/sdb1`이 없다

```bash
parted /dev/sdb print
partprobe /dev/sdb
udevadm settle
lsblk /dev/sdb
```

사용 중인 디스크의 파티션 테이블을 변경한 경우 재부팅이 필요할 수 있다.

---

### 🚨 시나리오 3. `mkfs`에서 기존 파일시스템 경고가 발생한다

```bash
wipefs -n /dev/sdb1
blkid /dev/sdb1
lsblk -f /dev/sdb1
```

기존 데이터가 필요하면 즉시 중단한다.

실습용 빈 장치가 확실한 경우에만:

```bash
mkfs.xfs -f /dev/sdb1
```

또는:

```bash
wipefs -a /dev/sdb1
```

> 두 명령 모두 기존 데이터 접근을 파괴할 수 있다.

---

### 🚨 시나리오 4. `mount -a`에서 UUID 오류가 발생한다

현재 UUID:

```bash
blkid
```

fstab:

```bash
grep -n -E '/GIT|/homeSK|/homeLG' /etc/fstab
```

검증:

```bash
findmnt --verify --verbose
```

수정 후:

```bash
systemctl daemon-reload
mount -a
```

---

### 🚨 시나리오 5. 재부팅 후 사용자 홈이 비어 있다

마운트 확인:

```bash
findmnt /homeLG
lsblk -f
```

fstab 확인:

```bash
grep '/homeLG' /etc/fstab
blkid /dev/sdd1
```

`/homeLG`가 마운트되지 않았다면 fstab 오류를 수정한다.

```bash
findmnt --verify --verbose
systemctl daemon-reload
mount /homeLG
```

> 홈 볼륨이 마운트되지 않은 상태에서 로그인하면 루트 파일시스템의 마운트포인트 아래에 새로운 파일이 생성될 수 있다. 원인을 해결하기 전 사용자 로그인을 제한하는 것이 안전하다.

---

### 🚨 시나리오 6. 계정을 먼저 만들고 나중에 `/homeLG`를 마운트했다

기존 홈이 마운트 아래에 가려진 상태다.

서비스·사용자 세션을 종료한 후:

```bash
umount /homeLG
```

기존 홈을 임시 위치로 이동:

```bash
mkdir -p /root/homeLG-hidden-backup
mv /homeLG/mUser* /root/homeLG-hidden-backup/
```

다시 마운트:

```bash
mount /homeLG
```

데이터 복원:

```bash
rsync -aHAX --numeric-ids \
  /root/homeLG-hidden-backup/ \
  /homeLG/
```

SELinux 컨텍스트 복구:

```bash
restorecon -RFv /homeLG
```

검증 후 임시 백업을 정리한다.

---

### 🚨 시나리오 7. LG팀 사용자가 공유 파일을 수정하지 못한다

현재 그룹:

```bash
id mUser1
id mUser2
```

디렉터리:

```bash
ls -ld /homeLG/shared
getfacl /homeLG/shared
```

파일 권한:

```bash
ls -l /homeLG/shared
```

가능한 원인:

- 기존 로그인 세션에 그룹 미반영
- 파일 생성자의 `umask 0022`
- 기본 ACL 없음
- SELinux 컨텍스트 문제

조치:

```bash
chmod 2770 /homeLG/shared
chown root:lgTeam /homeLG/shared
```

기본 ACL:

```bash
setfacl -d -m g:lgTeam:rwx /homeLG/shared
setfacl -d -m m::rwx /homeLG/shared
```

사용자는 재로그인한다.

---

### 🚨 시나리오 8. `umount`가 busy로 실패한다

```bash
cd /
findmnt -R /homeLG
fuser -vm /homeLG
```

사용자 세션 확인:

```bash
who
loginctl list-sessions
```

서비스 또는 세션을 정상 종료한 뒤:

```bash
umount /homeLG
```

`kill -9`과 `umount -l`은 최후 수단으로 사용한다.

---

### 🚨 시나리오 9. fstab 오류로 emergency mode에 진입했다

콘솔에서 root로 로그인한다.

루트가 읽기 전용이면:

```bash
mount -o remount,rw /
```

현재 fstab 백업:

```bash
cp -a /etc/fstab /etc/fstab.emergency
```

수정:

```bash
vi /etc/fstab
```

문제 항목을 수정하거나 임시 주석 처리한 후:

```bash
findmnt --verify --verbose
systemctl daemon-reload
mount -a
```

오류가 없으면:

```bash
reboot
```

---

## 4. ✅ 최종 체크리스트

```text
[ ] sdb·sdc·sdd 모델과 일련번호 확인
[ ] 세 디스크가 실습용 빈 디스크인지 확인
[ ] GPT 파티션 생성
[ ] sdb1 XFS, sdc1 ext4, sdd1 XFS 확인
[ ] /GIT, /homeSK, /homeLG 임시 마운트 검증
[ ] 읽기·쓰기 테스트 성공
[ ] fstab 백업
[ ] 실제 UUID로 fstab 등록
[ ] findmnt --verify --verbose 성공
[ ] fstab를 이용한 재마운트 성공
[ ] 사용자 생성 전에 /homeLG 마운트 완료
[ ] mUser1~3 홈 경로 확인
[ ] /homeLG/shared Set-GID와 ACL 확인
[ ] SELinux 컨텍스트 확인
[ ] 재부팅 후 자동 마운트 확인
[ ] 사용자 로그인 확인
```

> 📌 **핵심 요약**
> - 디스크 이름만 보지 말고 모델·일련번호 확인
> - 파티션과 포맷은 비가역적일 수 있음
> - 임시 마운트로 먼저 테스트
> - 홈 볼륨은 계정 생성 전에 fstab 영구화
> - 개인 홈과 팀 공유 디렉터리 분리
> - Set-GID만으로 그룹 쓰기 권한이 보장되지 않으므로 umask·ACL 검토
> - 최종 검증은 재부팅 후 `findmnt`, `df -Th`, 사용자 로그인
> - 관련: 8-1. 💽 디스크 타입 & 파티션 구조 · 8-2. 🗂️ 파일 시스템 & Format · 8-3. 🔗 마운트 & umount · 8-4. ⚓ Automount · 8-5. 📋 파티션·마운트 통합 정리 · 8-6. 🚨 파티션·마운트 트러블슈팅 치트시트 · 8-7. ⚡ Partition & Mount 명령어 퀵 레퍼런스
