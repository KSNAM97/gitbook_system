# 💽 디스크 타입 & 파티션 구조

> **Tag:** #Linux #Disk #Partition #MBR #GPT #Storage #NVMe  
> **핵심 요약:** Linux는 HDD·SSD·SAN·가상 디스크 같은 저장장치를 블록 디바이스로 인식한다. SATA·SAS·SCSI·USB 디스크는 주로 `/dev/sdX`, NVMe SSD는 `/dev/nvmeXnY`, VirtIO 디스크는 `/dev/vdX`로 표시된다. 파티션 테이블은 MBR과 GPT로 구분되며, 신규 시스템은 용량과 파티션 개수에 여유가 있고 복구 구조가 강화된 GPT 사용을 우선 검토한다.

---

## 1. 📖 개요 (Overview)

**Linux에서 디스크는 어떻게 표시되는가?**

Linux는 저장장치를 `/dev` 아래의 블록 디바이스로 표현한다.

대표 장치명:

| 장치 유형 | 전체 디스크 | 첫 번째 파티션 |
|---|---|---|
| SATA·SAS·SCSI·USB | `/dev/sda` | `/dev/sda1` |
| 두 번째 sd 디스크 | `/dev/sdb` | `/dev/sdb1` |
| NVMe SSD | `/dev/nvme0n1` | `/dev/nvme0n1p1` |
| VirtIO 가상 디스크 | `/dev/vda` | `/dev/vda1` |
| MMC·SD 카드 | `/dev/mmcblk0` | `/dev/mmcblk0p1` |

```text
/dev/sdb   → 디스크 전체
/dev/sdb1  → /dev/sdb의 첫 번째 파티션
/dev/sdb2  → /dev/sdb의 두 번째 파티션
```

NVMe처럼 디스크명 자체가 숫자로 끝나는 장치는 파티션 번호 앞에 `p`가 붙는다.

```text
/dev/nvme0n1    → NVMe 디스크 전체
/dev/nvme0n1p1  → 첫 번째 파티션
```

> **참고:** `/dev/sdb`와 같은 이름은 장치 인식 순서에 따라 달라질 수 있다. 영구 마운트에는 파일시스템 UUID 또는 LABEL 사용을 권장한다.

**IDE·SATA·SCSI·SAS·SSD·NVMe의 차이는?**

#### IDE/PATA

- 과거에 사용된 병렬 전송 방식
- 40핀 또는 80선 케이블 사용
- 최신 서버에서는 거의 사용하지 않음
- 과거 Linux 장치명은 `/dev/hda`, `/dev/hdb` 형태
- 최신 커널에서는 드라이버 계층에 따라 `/dev/sdX`로 표시될 수 있음

#### SATA

- IDE를 대체한 직렬 ATA 인터페이스
- 일반 PC와 서버의 HDD·SSD에서 널리 사용
- 이론상 링크 속도는 세대별로 증가
  - SATA I: 1.5 Gbit/s
  - SATA II: 3 Gbit/s
  - SATA III: 6 Gbit/s
- 컨트롤러와 운영 환경이 지원하면 Hot-plug 가능
- 일반적인 Linux 장치명은 `/dev/sdX`

#### SCSI

- 서버·워크스테이션·스토리지 장비에서 사용된 명령 및 인터페이스 계열
- Linux의 SCSI 계층은 SATA, SAS, USB 저장장치도 통합해 처리할 수 있음
- 최신 Linux에서는 주로 `/dev/sdX` 형태로 표시

#### SAS

- Serial Attached SCSI
- 기업용 서버와 스토리지에서 사용
- 다중 경로, 높은 신뢰성, 엔터프라이즈 장치 지원에 적합
- SAS 컨트롤러는 SATA 장치를 지원하는 경우가 많지만 SATA 컨트롤러에 SAS 디스크를 직접 연결할 수는 없음
- 일반적으로 `/dev/sdX` 형태로 표시

#### SSD

SSD는 연결 인터페이스가 아니라 플래시 메모리를 사용하는 저장장치 유형이다.

```text
SATA SSD → /dev/sdX
SAS SSD  → /dev/sdX
NVMe SSD → /dev/nvmeXnY
```

특징:

- 기계적 회전 부품 없음
- 낮은 지연 시간
- 높은 랜덤 I/O 성능
- 소음과 진동이 적음
- 쓰기 수명과 TRIM 관리 고려 필요

#### NVMe

- PCI Express 기반 고성능 저장장치 프로토콜
- SATA보다 높은 병렬성과 낮은 지연 시간 제공
- 장치명은 `/dev/nvme0n1`, `/dev/nvme1n1` 형식

확인:

```bash
lsblk -d -o NAME,MODEL,SERIAL,SIZE,TRAN,ROTA,TYPE
```

`ROTA`:

```text
1 → 회전식 디스크일 가능성
0 → 비회전식 SSD·가상 장치일 가능성
```

> **참고:** 가상환경이나 스토리지 추상화 계층에서는 `ROTA`, `TRAN` 정보가 실제 하드웨어와 정확히 일치하지 않을 수 있다.

**파티션이란?**

파티션은 하나의 디스크 저장공간을 논리적인 영역으로 나누는 구조이다.

```text
/dev/sdb
├─ /dev/sdb1
├─ /dev/sdb2
└─ /dev/sdb3
```

파티션별로 다음 정책을 독립적으로 적용할 수 있다.

- 파일시스템 종류
- 마운트 위치
- 용량 사용 범위
- 백업 정책
- 쿼터
- 보안 마운트 옵션
- 암호화
- Swap
- LVM Physical Volume

파티션 분리 목적:

1. `/var` 로그 폭주가 루트 파일시스템 전체를 채우는 위험 감소
2. `/home` 사용자 데이터와 시스템 영역 분리
3. 백업·복구 범위 분리
4. 파일시스템별 정책 적용
5. 다중 운영체제 구성
6. 장애 영향 범위 제한

> **주의:** 파티션 분리는 모든 장애를 완전히 격리하지 않는다. 물리 디스크 자체가 고장 나면 같은 디스크의 모든 파티션에 영향을 줄 수 있다.

**MBR과 GPT의 차이는?**

| 구분 | MBR/DOS | GPT |
|---|---|---|
| 구조 | 전통적 파티션 테이블 | GUID Partition Table |
| 대표 한계 | 512바이트 섹터 기준 약 2TiB | 매우 큰 디스크 지원 |
| 기본 파티션 수 | Primary 최대 4개 | 일반적으로 128개 |
| 확장 파티션 | 필요할 수 있음 | 불필요 |
| 복구 구조 | 디스크 앞부분 중심 | 주·백업 헤더 및 CRC |
| 신규 시스템 | 호환 목적 | 일반적으로 우선 권장 |

MBR에서는 다음 두 구성이 가능하다.

```text
Primary 4개
```

또는:

```text
Primary 1~3개 + Extended 1개
                    └─ Logical 파티션 여러 개
```

MBR 논리 파티션 번호는 일반적으로 5번부터 시작한다.

```text
/dev/sdb1 → Primary
/dev/sdb2 → Extended
/dev/sdb5 → Logical
/dev/sdb6 → Logical
```

GPT에서는 Primary·Extended·Logical 구분 없이 일반 파티션을 생성한다.

> **참고:** “2TB 이하이면 반드시 MBR”은 아니다. 작은 디스크도 GPT로 구성할 수 있으며, 신규 서버는 특별한 호환성 요구가 없다면 GPT가 관리하기 편하다.

**`fdisk`, `gdisk`, `parted`는 어떻게 선택하는가?**

| 도구 | 특징 |
|---|---|
| `fdisk` | 최신 util-linux에서 MBR과 GPT 모두 지원 |
| `gdisk` | GPT 중심 대화형 도구 |
| `parted` | GPT·MBR 지원, 스크립트화와 대용량 디스크 작업에 유용 |
| `sfdisk` | 파티션 테이블 출력·스크립트·백업에 유용 |

현재 파티션 테이블 확인:

```bash
fdisk -l /dev/sdb
parted /dev/sdb print
```

파티션 테이블 백업:

```bash
sfdisk -d /dev/sdb > /root/sdb-partition-table.backup
```

> **주의:** 파티션 생성·삭제·테이블 초기화는 데이터 손실을 일으킬 수 있다. 디스크 식별 정보를 확인하고 필요한 경우 파티션 테이블과 데이터를 먼저 백업한다.

**Primary·Extended·Logical 구분은 모든 디스크에 적용되는가?**

Primary·Extended·Logical 파티션 구분은 **MBR/DOS 파티션 테이블에서 사용하는 구조**이다.

```text
MBR/DOS
├─ Primary
├─ Extended
└─ Logical
```

GPT에서는 이러한 구분을 사용하지 않는다.

```text
GPT
├─ 일반 파티션 1
├─ 일반 파티션 2
├─ 일반 파티션 3
└─ ...
```

따라서 최신 시스템에서 파티션을 여러 개 구성해야 한다면 GPT를 사용하면 Extended 파티션을 별도로 만들 필요가 없다.

MBR을 사용하는 대표적인 이유:

- 구형 BIOS·운영체제와의 호환성
- MBR 구조 학습
- 기존 MBR 디스크 유지보수
- 특정 장비나 소프트웨어의 호환성 요구

> **참고:** “Primary 파티션은 최대 4개”라는 설명은 MBR에 해당한다. GPT 파티션을 Primary·Extended·Logical로 분류하지 않는다.

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 디스크 식별

전체 블록 장치:

```bash
lsblk
```

파일시스템·UUID 포함:

```bash
lsblk -f
```

디스크 모델·일련번호 포함:

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,TRAN,ROTA,TYPE
```

특정 디스크:

```bash
fdisk -l /dev/sdb
```

파티션 테이블 종류:

```bash
parted /dev/sdb print
```

udev 안정 경로:

```bash
ls -l /dev/disk/by-id/
ls -l /dev/disk/by-uuid/
```

> **참고:** 포맷이나 파티션 변경 전에는 크기뿐 아니라 `MODEL`, `SERIAL`, 기존 파일시스템, 마운트 여부를 함께 확인한다.

---

### Step 2. GPT 파티션 생성 — parted 예시

> **참고:** 다음 명령은 `/dev/sdb`의 기존 파티션 테이블과 데이터에 영향을 줄 수 있다. 반드시 실습용 빈 디스크인지 확인한다.

사전 확인:

```bash
lsblk -f /dev/sdb
fdisk -l /dev/sdb
wipefs -n /dev/sdb
```

GPT 테이블 생성:

```bash
parted -s /dev/sdb mklabel gpt
```

전체 영역에 파티션 하나 생성:

```bash
parted -s -a optimal /dev/sdb \
  mkpart primary 1MiB 100%
```

커널에 파티션 테이블 재인식 요청:

```bash
partprobe /dev/sdb
udevadm settle
```

검증:

```bash
lsblk /dev/sdb
parted /dev/sdb print
```

> **참고:** `parted mkpart`에서 파일시스템 이름을 지정해도 실제 파일시스템을 생성하는 것은 아니다. 실제 포맷은 `mkfs.xfs`, `mkfs.ext4` 등으로 별도 수행한다.

---

### Step 3. GPT 파티션 생성 — fdisk 예시

```bash
fdisk /dev/sdb
```

대표 입력 흐름:

```text
g       GPT 파티션 테이블 생성
n       새 파티션
Enter   기본 파티션 번호
Enter   기본 시작 섹터
Enter   남은 공간 전체 사용
p       구성 확인
w       저장 후 종료
```

저장 전 취소:

```text
q       저장하지 않고 종료
```

반영:

```bash
partprobe /dev/sdb
udevadm settle
lsblk /dev/sdb
```

---

### Step 4. MBR 파티션 생성 실습

MBR 테이블 생성:

```bash
fdisk /dev/sdb
```

대표 입력:

```text
o       새 DOS/MBR 파티션 테이블
n       새 파티션
p       Primary
1       파티션 번호
Enter   기본 시작 섹터
+30G    30GiB 할당
p       확인
w       저장
```

커널 반영:

```bash
partprobe /dev/sdb
udevadm settle
```

검증:

```bash
lsblk /dev/sdb
fdisk -l /dev/sdb
```

---

### Step 5. MBR Extended·Logical 구조 실습

예시 목표:

```text
/dev/sdb1 → Primary 30GiB
/dev/sdb2 → Extended
/dev/sdb5 → Logical 20GiB
/dev/sdb6 → Logical 20GiB
```

대화형 `fdisk` 흐름:

```text
n → p → 1 → Enter → +30G
n → e → 2 → Enter → Enter
n → Enter → +20G
n → Enter → +20G
p
w
```

결과 확인:

```bash
fdisk -l /dev/sdb
lsblk /dev/sdb
```

> **참고:** Extended 파티션은 데이터를 직접 저장하는 일반 파티션이 아니라 Logical 파티션을 담는 컨테이너이다. `lsblk`에서 매우 작은 크기로 표시되는 경우가 있다.

---

### Step 6. MBR 다중 파티션 종합 실습

100GiB 실습용 디스크 `/dev/sdb`를 다음과 같이 구성한다.

```text
/dev/sdb1 → Primary 30GiB
/dev/sdb2 → Primary 30GiB
/dev/sdb3 → Extended, 남은 영역
/dev/sdb5 → Logical 10GiB
/dev/sdb6 → Logical 10GiB
/dev/sdb7 → Logical 10GiB
/dev/sdb8 → Logical, 남은 영역
```

> **참고:** 정렬 공간과 Extended Boot Record 때문에 마지막 파티션의 실제 섹터 수는 정확히 10GiB보다 조금 작을 수 있다. `lsblk`에서는 반올림되어 10G로 표시될 수 있다.

사전 확인:

```bash
lsblk -f /dev/sdb
fdisk -l /dev/sdb
wipefs -n /dev/sdb
findmnt -S /dev/sdb
```

파티션 편집:

```bash
fdisk /dev/sdb
```

대표 입력 흐름:

```text
o
n → p → 1 → Enter → +30G
n → p → 2 → Enter → +30G
n → e → 3 → Enter → Enter
n → Enter → +10G
n → Enter → +10G
n → Enter → +10G
n → Enter → Enter
p
w
```

입력 의미:

| 입력 | 의미 |
|---|---|
| `o` | 새 DOS/MBR 파티션 테이블 생성 |
| `n` | 새 파티션 생성 |
| `p` | 파티션 생성 중에는 Primary 선택 |
| `e` | Extended 파티션 선택 |
| `p` | 메인 프롬프트에서는 파티션 테이블 출력 |
| `w` | 변경 사항 저장 |
| `q` | 저장하지 않고 종료 |

커널 반영:

```bash
partprobe /dev/sdb
udevadm settle
```

검증:

```bash
fdisk -l /dev/sdb
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/sdb
```

> **참고:** 신규 운영 시스템이라면 같은 구성을 GPT 일반 파티션 6개로 만드는 편이 단순하다. 이 실습은 MBR의 Extended·Logical 구조를 이해하기 위한 예제이다.

---

### Step 7. 파티션 삭제

현재 상태 확인:

```bash
lsblk -f /dev/sdb
findmnt -S /dev/sdb1
```

마운트되어 있다면 먼저 해제한다.

```bash
umount /dev/sdb1
```

파티션 편집:

```bash
fdisk /dev/sdb
```

입력:

```text
p       현재 목록
d       삭제
번호    삭제할 파티션 번호
p       결과 확인
w       저장
```

반영:

```bash
partprobe /dev/sdb
udevadm settle
lsblk /dev/sdb
```

> **주의:** 파티션 삭제는 파일시스템과 데이터에 접근할 수 없게 만든다. 번호와 디스크를 잘못 선택하지 않도록 주의한다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
lsblk -f
fdisk -l /dev/sdb
parted /dev/sdb print
wipefs -n /dev/sdb
```

---

### 3-2. 새 디스크가 보이지 않는다

`partprobe`는 새 하드웨어를 검색하는 명령이 아니라 변경된 파티션 테이블을 다시 읽도록 요청하는 명령이다.

먼저 가상머신·스토리지 설정에서 디스크가 실제 연결되었는지 확인한다.

SCSI 호스트 확인:

```bash
ls /sys/class/scsi_host/
```

모든 SCSI 호스트 재검색:

```bash
for host in /sys/class/scsi_host/host*; do
  echo "- - -" > "$host/scan"
done
```

그 후:

```bash
udevadm settle
lsblk
dmesg --ctime | tail -n 50
```

그래도 보이지 않으면:

- VM 디스크 연결 상태 확인
- 하이퍼바이저 재검색 기능 사용
- 스토리지 경로·컨트롤러 확인
- 유지보수 시간에 재부팅 검토

---

### 3-3. 파티션을 만들었는데 `/dev/sdb1`이 없다

```bash
partprobe /dev/sdb
udevadm settle
lsblk /dev/sdb
```

사용 중인 디스크라 커널이 테이블을 다시 읽지 못하면 재부팅이 필요할 수 있다.

---

### 3-4. 디스크를 잘못 선택할까 걱정된다

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,WWN
findmnt
wipefs -n /dev/sdb
```

다음 항목을 모두 확인한다.

```text
장치명
크기
모델
일련번호 또는 WWN
마운트 여부
기존 파일시스템·RAID·LVM 시그니처
```

---

### 3-5. `lsblk`, `fdisk`, `df`의 차이

| 명령 | 확인 관점 | 마운트되지 않은 장치 |
|---|---|---|
| `lsblk` | 디스크·파티션·LVM 등 블록 장치 구조 | 표시 가능 |
| `fdisk -l` | 디스크의 파티션 테이블과 섹터 정보 | 표시 가능 |
| `df` | 마운트된 파일시스템의 사용량 | 일반적으로 표시하지 않음 |

블록 장치 구조:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

파티션 테이블:

```bash
fdisk -l /dev/sdb
```

마운트된 파일시스템 사용량:

```bash
df -Th
```

`/dev/sdb1`에 파일시스템이 있어도 마운트하지 않았다면 `lsblk -f`에는 표시되지만 `df -Th`에는 나타나지 않을 수 있다.

> 📌 **핵심 요약**
> - 전체 디스크와 파티션을 구분
> - 신규 환경은 GPT 우선 검토
> - Primary·Extended·Logical 구조는 MBR에 해당
> - 최신 `fdisk`는 MBR·GPT 지원
> - `partprobe`는 파티션 테이블 재인식용
> - 신규 디스크 확인은 `df`보다 `lsblk` 사용
> - 영구 마운트에는 UUID 또는 안정적인 식별자 사용
> - 관련: 8-2. 🗂️ 파일 시스템 & Format · 8-3. 🔗 마운트 & umount · 8-4. ⚓ Automount
