# 🗂️ 파일 시스템 & Format

> **Tag:** #Linux #FileSystem #Format #mkfs #XFS #ext4 #Swap #Journaling  
> **핵심 요약:** 파티션은 저장공간의 경계이고 파일시스템은 파일·디렉터리·권한·소유권·메타데이터를 저장하는 구조이다. `mkfs.xfs`, `mkfs.ext4`, `mkswap`은 기존 데이터를 파괴할 수 있으므로 실행 전 대상 장치, 마운트 여부, 기존 시그니처를 반드시 확인한다.

---

## 1. 📖 개요 (Overview)

**파티션과 파일시스템의 차이는?**

```text
파티션
→ 디스크 공간을 논리적으로 구분한 영역

파일시스템
→ 해당 영역에 파일과 디렉터리를 저장·검색·관리하는 구조
```

일반적인 흐름:

```text
/dev/sdb
→ 파티션 생성
→ /dev/sdb1
→ mkfs.xfs 또는 mkfs.ext4
→ 파일시스템 생성
→ 디렉터리에 mount
→ 데이터 저장
```

파티션 없이 디스크 전체에 직접 파일시스템을 만들 수도 있다.

```bash
mkfs.xfs /dev/sdb
```

하지만 일반적인 서버 운영과 도구 호환성을 위해 파티션 또는 LVM을 구성한 뒤 파일시스템을 생성하는 방식을 주로 사용한다.

**파일시스템은 무엇을 관리하는가?**

- 파일명과 디렉터리 구조
- 데이터 블록 위치
- UID·GID 소유권
- `rwx` 권한
- 타임스탬프
- 링크
- 확장 속성
- ACL
- 파일시스템 상태와 저널

확인:

```bash
lsblk -f
blkid
df -Th
```

**ext4와 XFS는 어떻게 선택하는가?**

| 구분 | XFS | ext4 |
|---|---|---|
| RHEL/Rocky 기본 | 일반적으로 기본 데이터 파일시스템 | 지원되는 범용 파일시스템 |
| 온라인 확장 | 지원 | 지원 |
| 축소 | 지원하지 않음 | 오프라인 축소 가능 |
| 대규모 병렬 I/O | 강점 | 범용적으로 안정적 |
| 복구 도구 | `xfs_repair` | `e2fsck` |
| 확장 도구 | `xfs_growfs` | `resize2fs` |

XFS:

- 64비트 저널링 파일시스템
- 대규모 파일과 병렬 I/O에 적합
- 작은 볼륨에서도 사용 가능
- “최소 2TB 이상에서만 사용”해야 하는 것은 아님
- 축소가 불가능하므로 용량 설계 주의

ext4:

- 범용 저널링 파일시스템
- 온라인 확장 가능
- 축소는 일반적으로 마운트 해제 후 수행
- 다양한 Linux 환경에서 널리 지원

> 최대 파일·파일시스템 크기는 블록 크기, 커널, 배포판 지원 범위와 도구 버전에 따라 달라지므로 운영 설계에서는 해당 배포판 공식 지원 범위를 확인한다.

**Journaling이란?**

파일시스템 변경과 관련된 메타데이터를 저널에 기록해 비정상 종료 후 일관성을 빠르게 복구할 수 있도록 돕는 기능이다.

```text
변경 예정 기록
→ 실제 파일시스템 반영
→ 완료 처리
```

저널링은 갑작스러운 종료 후 복구 시간을 줄이지만 다음을 대신하지 않는다.

- 정식 백업
- 스냅샷
- 데이터베이스 일관성 보장
- 실수로 삭제한 파일의 복원
- 물리 디스크 장애 대비

**Swap은 파일시스템인가?**

Swap은 일반 파일 저장용 파일시스템이 아니라 메모리 페이지를 임시 저장하는 전용 영역이다.

생성:

```bash
mkswap /dev/sdb2
```

활성화:

```bash
swapon /dev/sdb2
```

확인:

```bash
swapon --show
free -h
```

Swap 크기는 다음 조건에 따라 결정한다.

- 실제 워크로드 메모리 사용량
- OOM 위험 허용 수준
- 하이버네이션 필요 여부
- 커널 덤프 정책
- 스토리지 성능
- 애플리케이션 권장사항

고정된 하나의 공식으로 모든 서버의 Swap 크기를 결정하지 않는다.

**ext, ext2, ext3, ext4는 어떤 관계인가?**

EXT 계열은 Linux에서 발전해 온 파일시스템 계열이다.

| 파일시스템 | 주요 특징 |
|---|---|
| ext | 초기 Linux용 확장 파일시스템 |
| ext2 | 저널이 없는 전통적인 Linux 파일시스템 |
| ext3 | ext2에 저널링 기능 추가 |
| ext4 | ext3를 확장해 용량·성능·신뢰성 기능 개선 |

운영 환경에서는 단순히 상위 버전이라는 이유로 바로 변환하지 않는다.

다음 항목을 함께 확인한다.

- 배포판과 커널의 지원 범위
- 활성화된 파일시스템 기능
- 부트로더 지원 여부
- 파일시스템 크기와 블록 크기
- 백업 및 복구 가능 여부

과거 자료에 표시된 최대 크기는 당시 커널·도구·배포판 제한일 수 있다. 실제 운영 설계에서는 현재 배포판의 공식 지원 범위를 기준으로 판단한다.

**포맷은 데이터를 안전하게 완전 삭제하는 작업인가?**

아니다. `mkfs`는 새로운 파일시스템 메타데이터를 생성하는 작업이다.

```text
mkfs 실행
→ 기존 파일시스템 구조 일부가 덮어써짐
→ 기존 파일에 정상적으로 접근하기 어려워짐
```

모든 데이터 블록을 반드시 덮어쓰는 것은 아니므로 보안 목적의 완전 삭제 절차가 아니다.

`wipefs -a`도 파일시스템·RAID·파티션 테이블 시그니처를 제거할 뿐 장치 전체 데이터를 안전하게 덮어쓰지 않는다.

보안 폐기가 필요하다면 다음 항목을 별도로 검토한다.

- 저장장치 종류
- SSD Wear Leveling
- 암호화 키 폐기
- Secure Erase 또는 Sanitize
- 조직의 데이터 폐기 정책
- 물리적 폐기 필요 여부

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 포맷 전 필수 확인

```bash
lsblk -f /dev/sdb
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/sdb
findmnt -S /dev/sdb1
wipefs -n /dev/sdb1
blkid /dev/sdb1
```

LVM·RAID 사용 여부:

```bash
pvs
lvs
cat /proc/mdstat
```

> 포맷 대상이 맞다는 확신이 없으면 `mkfs`, `wipefs -a` 또는 강제 포맷을 실행하지 않는다.

---

### 2-2. XFS 파일시스템 생성

```bash
mkfs.xfs /dev/sdb1
```

레이블 지정:

```bash
mkfs.xfs -L DATA /dev/sdb1
```

기존 시그니처가 있을 때 강제:

```bash
mkfs.xfs -f /dev/sdb1
```

> `-f`는 기존 파일시스템을 덮어쓸 수 있는 파괴적 옵션이다.

검증:

```bash
lsblk -f /dev/sdb1
blkid /dev/sdb1
xfs_info /dev/sdb1
```

읽기 전용 검사:

```bash
xfs_repair -n /dev/sdb1
```

실제 복구:

```bash
umount /dev/sdb1
xfs_repair /dev/sdb1
```

---

### 2-3. ext4 파일시스템 생성

```bash
mkfs.ext4 /dev/sdb2
```

동일한 일반 형식:

```bash
mkfs -t ext4 /dev/sdb2
```

레이블 지정:

```bash
mkfs.ext4 -L BACKUP /dev/sdb2
```

검증:

```bash
lsblk -f /dev/sdb2
blkid /dev/sdb2
tune2fs -l /dev/sdb2 | head
```

ext4 검사:

```bash
umount /dev/sdb2
e2fsck -f /dev/sdb2
```

> 읽기·쓰기로 마운트된 ext4에 `e2fsck`를 실행하지 않는다.

---

### 2-4. 파일시스템 레이블 관리

통합 확인:

```bash
lsblk -f
blkid
```

XFS 레이블 확인:

```bash
xfs_admin -l /dev/sdb1
```

XFS 레이블 변경 전 마운트 여부를 확인하고 해제한다.

```bash
findmnt -S /dev/sdb1
umount /dev/sdb1
xfs_admin -L DATA /dev/sdb1
```

ext4 레이블 확인·변경:

```bash
e2label /dev/sdb2
e2label /dev/sdb2 BACKUP
```

LABEL로 마운트:

```bash
mkdir -p /data
mount LABEL=DATA /data
```

> 같은 시스템에 중복 LABEL이 있으면 장치 식별이 모호해질 수 있다.

---

### 2-5. Swap 파티션 생성

```bash
lsblk -f /dev/sdb2
findmnt -S /dev/sdb2
mkswap /dev/sdb2
swapon /dev/sdb2
```

확인:

```bash
swapon --show
free -h
```

비활성화:

```bash
swapoff /dev/sdb2
```

fstab 예시:

```fstab
UUID=<swap-uuid>  none  swap  defaults  0 0
```

---

### 2-6. Swap 파일 생성 예시

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

검증:

```bash
swapon --show
ls -lh /swapfile
```

fstab:

```fstab
/swapfile  none  swap  defaults  0 0
```

> Copy-on-Write 파일시스템에서는 별도 절차가 필요할 수 있다.

---

### 2-7. XFS 확장

```bash
findmnt /data
xfs_growfs /data
```

검증:

```bash
df -Th /data
xfs_info /data
```

> 하위 블록 장치, 파티션 또는 LVM LV의 용량을 먼저 늘려야 한다.

---

### 2-8. ext4 확장·축소 개념

확장:

```bash
resize2fs /dev/sdb2
```

축소 순서:

```text
백업
→ 마운트 해제
→ e2fsck 검사
→ resize2fs로 파일시스템 축소
→ 파티션 또는 LV 축소
→ 재마운트
→ 검증
```

> 축소 순서를 잘못 처리하면 데이터가 손상될 수 있다.

---

### 2-9. 여러 파티션에 파일시스템 생성 실습

구성 목표:

```text
/dev/sdb1 → ext4, LABEL=SOL
/dev/sdb2 → XFS,  LABEL=USER
/dev/sdb5 → ext4, LABEL=CISCO
/dev/sdb6 → ext4, LABEL=GUEST
/dev/sdb7 → ext4, LABEL=NOBODY
/dev/sdb8 → ext4, LABEL=SPARE
```

사전 확인:

```bash
lsblk -f /dev/sdb
swapon --show
pvs
cat /proc/mdstat
```

기존 시그니처 확인:

```bash
for dev in /dev/sdb1 /dev/sdb2 /dev/sdb5 /dev/sdb6 /dev/sdb7 /dev/sdb8; do
  wipefs -n "$dev"
done
```

파일시스템 생성:

```bash
mkfs.ext4 -L SOL /dev/sdb1
mkfs.xfs -L USER /dev/sdb2
mkfs.ext4 -L CISCO /dev/sdb5
mkfs.ext4 -L GUEST /dev/sdb6
mkfs.ext4 -L NOBODY /dev/sdb7
mkfs.ext4 -L SPARE /dev/sdb8
```

검증:

```bash
lsblk -f /dev/sdb
blkid /dev/sdb1 /dev/sdb2 /dev/sdb5 /dev/sdb6 /dev/sdb7 /dev/sdb8
```

> Extended 파티션 자체인 `/dev/sdb3`에는 파일시스템을 생성하지 않는다. 실제 파일시스템은 Logical 파티션에 생성한다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 포맷 결과 검증

```bash
lsblk -f
blkid /dev/sdb1
file -s /dev/sdb1
```

마운트 후:

```bash
df -Th
findmnt /data
```

---

### 3-2. 기존 파일시스템이 있다는 경고

```bash
wipefs -n /dev/sdb1
blkid /dev/sdb1
```

삭제해도 되는 것이 확실한 경우에만:

```bash
wipefs -a /dev/sdb1
```

또는:

```bash
mkfs.xfs -f /dev/sdb1
```

> `wipefs -a`는 데이터 전체를 안전하게 삭제하지 않는다.

---

### 3-3. `device is busy`

```bash
findmnt -S /dev/sdb1
lsblk /dev/sdb1
fuser -vm /data
```

마운트 해제:

```bash
cd /
umount /data
```

Swap이면:

```bash
swapoff /dev/sdb1
```

LVM·RAID 구성원이라면 해당 계층부터 확인한다.

---

### 3-4. 포맷 후 UUID가 바뀌었다

새 파일시스템을 생성하면 UUID가 새로 만들어진다.

```bash
blkid /dev/sdb1
grep -n '/data' /etc/fstab
```

fstab 검증:

```bash
findmnt --verify --verbose
mount -a
```

---

### 3-5. UUID와 장치명의 차이

장치명은 인식 순서에 따라 달라질 수 있다.

```text
/dev/sdb1
/dev/sdc1
/dev/nvme0n1p1
```

UUID 확인:

```bash
blkid /dev/sdb1
lsblk -f /dev/sdb1
```

파일시스템이 그대로라면 장치명이 변경되어도 UUID는 일반적으로 유지된다.

UUID가 변경될 수 있는 작업:

- `mkfs`로 파일시스템 재생성
- 파일시스템 복제 후 UUID 변경
- 관리 도구로 UUID 직접 변경
- 백업 이미지를 다른 장치에 복원

중복 확인:

```bash
blkid
ls -l /dev/disk/by-uuid/
```

fstab 예시:

```fstab
UUID=<filesystem-uuid>  /data  ext4  defaults  0 2
```

> 📌 **핵심 요약**
> - 파티션 생성 후 `mkfs`로 파일시스템 생성
> - XFS: 온라인 확장 가능, 축소 불가
> - ext4: 확장 가능, 축소는 일반적으로 오프라인
> - Swap: `mkswap` → `swapon`
> - 포맷과 `wipefs`는 보안 목적의 완전 삭제가 아님
> - 포맷 전 `lsblk`, `findmnt`, `wipefs -n` 확인
> - 관련: 8-1. 💽 디스크 타입 & 파티션 구조 · 8-3. 🔗 마운트 & umount · 8-4. ⚓ Automount
