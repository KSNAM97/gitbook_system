# 파티션·마운트 통합 정리

> **Tag:** #Linux #Partition #FileSystem #Mount #fstab #Summary  
> **핵심 요약:** 신규 디스크의 표준 구성 흐름은 디스크 식별 → 파티션 또는 LVM 구성 → 파일시스템 생성 → 임시 마운트 검증 → `/etc/fstab` 영구화이다. 각 단계에서 대상 장치와 결과를 확인해야 하며, 가장 위험한 작업은 잘못된 장치에 대한 파티션 초기화·포맷과 검증되지 않은 fstab 설정이다.

---

## 1. 개요 (Overview)

신규 디스크 구성의 표준 5단계는 다음과 같습니다.

```text
1. 디스크 인식·식별
2. 파티션 생성
3. 파일시스템 생성
4. 임시 마운트와 읽기·쓰기 검증
5. fstab 등록과 영구 마운트 검증
```

대표 검증:

```text
인식       → lsblk, 모델·일련번호
파티션     → fdisk -l, partprobe, lsblk
파일시스템 → lsblk -f, blkid
마운트     → findmnt, df -Th
영구화     → findmnt --verify, mount -a
```

> **참고:** 파티션을 생략하고 디스크 전체를 사용하거나 LVM·RAID·암호화를 추가하는 구성도 가능하다. 위 순서는 일반적인 파티션 기반 교육·운영 템플릿이다.

가장 위험한 지점은 다음과 같다.

1. 잘못된 디스크에 파티션 테이블 생성
2. 사용 중인 파일시스템에 `mkfs`
3. 기존 시그니처를 `wipefs -a`로 제거
4. UUID·타입이 틀린 fstab 등록
5. 기존 데이터 디렉터리에 마운트해 내용 가림
6. 백업 없이 파일시스템 축소

---

## 2. 표준 전체 워크플로 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

> **참고:** 아래의 `/dev/sdb`는 예시다. 실제 장치의 모델·일련번호·크기·사용 상태를 확인한 뒤 대체한다.

### Step 1. 디스크 식별

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,WWN
lsblk -f
```

대상 확인:

```bash
fdisk -l /dev/sdb
wipefs -n /dev/sdb
findmnt -S /dev/sdb
```

### Step 2. GPT 파티션 생성

```bash
parted -s /dev/sdb mklabel gpt
parted -s -a optimal /dev/sdb \
  mkpart primary 1MiB 100%

partprobe /dev/sdb
udevadm settle
lsblk /dev/sdb
```

### Step 3. 파일시스템 생성

```bash
mkfs.xfs -L DATA /dev/sdb1
```

검증:

```bash
lsblk -f /dev/sdb
blkid /dev/sdb1
```

### Step 4. 임시 마운트

```bash
mkdir -p /data
ls -la /data

mount /dev/sdb1 /data
findmnt /data
df -Th /data
```

쓰기 테스트:

```bash
touch /data/.mount-test
rm /data/.mount-test
```

### Step 5. fstab 영구 등록

UUID 확인:

```bash
blkid -s UUID -o value /dev/sdb1
```

fstab 백업:

```bash
cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"
```

등록 예:

```fstab
UUID=<실제-UUID>  /data  xfs  defaults  0 0
```

검증을 위해 수동 마운트 해제:

```bash
umount /data
```

검증:

```bash
findmnt --verify --verbose
systemctl daemon-reload
mount /data
findmnt /data
df -Th /data
```

전체 fstab 확인:

```bash
mount -a
```

---

### Step 1. 단계별 요약표

| 단계 | 작업 | 대표 명령 | 검증 |
|---|---|---|---|
| 인식 | 장치 식별 | `lsblk` | 모델·SERIAL |
| 파티션 | GPT/MBR 구성 | `fdisk`, `parted` | `fdisk -l` |
| 재인식 | 커널 테이블 반영 | `partprobe` | `lsblk` |
| 포맷 | XFS·ext4 생성 | `mkfs.*` | `blkid` |
| 임시 마운트 | 연결 테스트 | `mount` | `findmnt` |
| 영구화 | fstab 등록 | `sudoedit` | `findmnt --verify` |
| 최종 검증 | fstab 기반 연결 | `mount /data` | `df -Th` |

---

### Step 2. 파일시스템 선택

| 파일시스템 | 특징 | 크기 변경 |
|---|---|---|
| XFS | RHEL/Rocky 기본, 병렬·대용량 I/O | 온라인 확장, 축소 불가 |
| ext4 | 범용·안정적 | 확장 가능, 오프라인 축소 가능 |
| Swap | 메모리 페이지 임시 저장 | `mkswap`, `swapon` 사용 |

---

### Step 3. 마운트 옵션 선택

일반 데이터:

```fstab
UUID=<uuid> /data xfs defaults 0 0
```

비필수 외장 볼륨:

```fstab
UUID=<uuid> /archive xfs defaults,nofail,x-systemd.device-timeout=10s 0 0
```

임시 업로드 공간 보안 강화 예:

```fstab
UUID=<uuid> /upload xfs defaults,nosuid,nodev,noexec 0 0
```

> **참고:** 마운트 옵션은 애플리케이션 요구사항을 검토한 뒤 적용한다.

---

## 3. 전 단계 검증 및 트러블슈팅

통합 확인:

```bash
lsblk -f
blkid
findmnt
df -Th
```

fstab:

```bash
findmnt --verify --verbose
mount -a
```

Swap:

```bash
swapon --show
free -h
```

대표 장애:

| 증상 | 우선 확인 |
|---|---|
| 새 디스크 없음 | 하이퍼바이저·SCSI 재검색 |
| 파티션 장치 없음 | `partprobe`, `udevadm settle` |
| mkfs 대상 오류 | `lsblk`, `wipefs -n`, `findmnt` |
| mount 타입 오류 | `blkid`, `file -s` |
| umount busy | `cd /`, `fuser -vm` |
| 기존 파일이 안 보임 | 해당 경로 마운트 여부 |
| 재부팅 후 누락 | fstab, UUID, journal |
| emergency mode | fstab 오류·필수 장치 미존재 |

>  **핵심 요약**
> - 식별 → 파티션 → 포맷 → 테스트 마운트 → fstab
> - 포맷 전 장치 모델·일련번호·마운트 상태 확인
> - fstab 변경 전 백업
> - 변경 후 `findmnt --verify --verbose`
> - 관련: 8-1.  디스크 타입 & 파티션 구조 · 8-2.  파일 시스템 & Format · 8-3.  마운트 & umount · 8-4.  Automount
