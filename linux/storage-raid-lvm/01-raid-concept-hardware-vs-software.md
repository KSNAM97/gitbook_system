# 🧩 RAID 개념 & Hardware vs Software RAID

> **Tag:** #Linux #RAID #Storage #HardwareRAID #SoftwareRAID #mdadm #FaultTolerance
> **핵심 요약:** RAID는 여러 개의 물리 디스크(HDD·SSD)를 하나의 논리 디스크처럼 묶어 성능 향상, 데이터 보호, 대용량 확보를 목적으로 하는 기술이다. 전용 컨트롤러가 연산을 처리하는 Hardware RAID와, 운영체제(Linux는 `mdadm`)가 연산을 처리하는 Software RAID로 나뉜다. 두 방식 모두 RAID 0·1·5·6 같은 레벨을 구성할 수 있지만 성능, 비용, 장애 처리 방식이 다르므로 환경에 맞게 선택한다.

---

## 1. 📖 개요 (Overview)

RAID(Redundant Array of Inexpensive/Independent Disks)는 **여러 개의 물리 디스크를 하나의 논리적 디스크처럼 묶어** 사용하는 기술입니다. 목적은 크게 세 가지, **성능 향상 · 데이터 보호(중복 저장을 통한 결함 허용) · 대용량 저장공간 확보**이며, RAID 레벨에 따라 이 셋 중 어디에 중점을 두는지가 달라집니다.

```text
여러 물리 디스크(HDD·SSD)
→ 하나의 논리 디스크로 묶음
→ 성능 향상 · 데이터 보호(중복 저장) · 대용량 확보
```

**RAID는 백업을 대체하지 않는다.** RAID 1·5·6 같은 결함 허용 레벨도 실수로 삭제한 파일, 랜섬웨어, 논리적 손상까지는 보호하지 못하므로 RAID와 별도로 정식 백업 정책을 유지해야 합니다. RAID는 디스크 **물리적 고장**에 대한 대응이지, 사람의 실수나 소프트웨어 결함까지 막아주지 않습니다.

Hardware RAID는 RAID 전용 컨트롤러 카드(RAID Card, HBA) 또는 스토리지 장비가 **RAID 연산(스트라이핑·미러링·패리티 계산)을 직접 수행**합니다. 디스크 I/O를 컨트롤러가 처리하므로 서버 OS·CPU 부담이 줄고, 일반적으로 **고성능·고안정성**을 제공합니다.

| 구분 | 내용 |
|---|---|
| 고성능 | RAID 컨트롤러 칩셋이 대부분의 연산을 처리 |
| 높은 안정성 | 기업 서버·스토리지에서 널리 사용 |
| Hot Swap | 서비스 중단 없이 장애 디스크 교체 가능 |
| 대규모 환경 | 대규모 서버 환경에서 사실상 필수적 |

단점은 **비용이 높다**는 점과, RAID 카드 고장 시 **같은 제조사·모델·펌웨어 버전**이 필요할 수 있어 호환성 관리 부담이 있다는 점입니다.

Software RAID는 운영체제가 RAID 기능을 직접 수행합니다. Linux는 `mdadm`, Windows는 디스크 관리 기능으로 구성하며, **스트라이핑·미러링·패리티 계산을 CPU가 직접** 수행합니다. 추가 하드웨어가 필요 없어 저렴하고 클라우드 환경에서도 자주 사용됩니다.

| 구분 | 내용 |
|---|---|
| 비용 절감 | HDD·SSD만 있으면 구성 가능 |
| 간단한 구성 | OS 명령어만으로 생성·관리 |
| 유연성 | RAID 레벨 변경, 디스크 확장 등이 비교적 쉬움 |

단점은 **CPU 사용량 증가**(패리티 연산 부담), **제한적인 Hot Swap**(디스크 분리·재인식 과정 필요), 대규모 환경에서 Hardware RAID 대비 장애 처리 능력·성능이 떨어질 수 있다는 점입니다.

Hardware RAID와 Software RAID를 비교하면, 연산 주체가 컨트롤러(Hardware)냐 OS/CPU(Software, `mdadm`)냐가 핵심 차이이며, 이에 따라 성능·비용·Hot Swap 지원 여부가 달라집니다.

| 구분 | Hardware RAID | Software RAID |
|---|---|---|
| 연산 주체 | 전용 컨트롤러·스토리지 | OS·CPU (`mdadm`) |
| 성능 | 일반적으로 고성능 | CPU 부담 존재 |
| 비용 | 높음(전용 장비) | 낮음(디스크만 필요) |
| Hot Swap | 지원 | 제한적 |
| 구성 난이도 | 장비 의존 | OS 명령어로 구성 |
| 적합 환경 | 대규모·엔터프라이즈 | 소규모·학습·클라우드 |

"Software RAID는 무조건 느리다"는 단정은 정확하지 않습니다. 최신 CPU 환경에서는 RAID 1·0 같은 단순 레벨의 부담이 크지 않을 수 있으나, 패리티 연산이 많은 RAID 5·6은 CPU와 워크로드를 함께 고려해야 합니다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. mdadm 패키지 확인 및 설치

```bash
rpm -qa | grep mdadm          # mdadm 설치 여부 확인
dnf install -y mdadm          # Rocky·RHEL 계열에서 mdadm 설치
```

### Step 2. 현재 디스크·RAID 상태 확인

```bash
ls -ld /dev/sd*               # 인식된 디스크 장치 목록 확인
lsblk                         # 전체 블록 장치와 계층 구조 확인
lsblk -f                      # 파일시스템·UUID·RAID 멤버 정보 확인
fdisk -l                      # 디스크·파티션 상세 정보 확인
cat /proc/mdstat              # 현재 조립된 Software RAID 상태 확인
mdadm --detail --scan         # 조립된 RAID 요약 정보 확인
```

> **참고:** 실습 환경에서는 신규 디스크(`/dev/sdb`, `/dev/sdc`…)가 파티션 테이블이 없는 **Raw 디스크** 상태로 인식됩니다. `fdisk`로 파티션을 먼저 생성한 뒤 RAID 멤버로 사용하는 것이 표준 흐름입니다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. RAID 방식 선택 판단

```text
대규모·성능·Hot Swap 필수  → Hardware RAID 검토
비용 절감·학습·클라우드     → Software RAID(mdadm) 검토
```

### 3-2. RAID 멤버 디스크 확인

```bash
lsblk -f                      # linux_raid_member 표시 여부 확인
mdadm --examine /dev/sdb1     # 디스크에 저장된 RAID 슈퍼블록 확인
```

`linux_raid_member`로 표시되면 해당 디스크(파티션)가 어떤 RAID 배열의 구성원인지를 의미합니다.

### 3-3. 대표 주의사항

- 실습·구성 전에는 항상 `lsblk`, `fdisk -l`로 **대상 디스크가 맞는지** 먼저 확인합니다. 잘못된 디스크에 파티션·RAID를 구성하면 기존 데이터가 손상될 수 있습니다.
- RAID는 데이터 **보호(중복)** 수단이지 백업이 아니므로, 중요 데이터는 RAID 구성과 무관하게 별도 백업을 유지합니다.

> 📌 **핵심 요약**
> - RAID는 여러 물리 디스크를 하나의 논리 디스크로 묶는 기술
> - Hardware RAID는 전용 컨트롤러, Software RAID는 OS·CPU(`mdadm`)가 연산
> - Linux Software RAID는 `mdadm`으로 구성
> - RAID는 백업을 대체하지 않음
> - RAID 레벨에 따라 성능·안정성·공간효율 균형이 다름
> - 관련: 📊 RAID 레벨별 특징 (Linear·0·1·5·6) · ⚙️ mdadm 명령어 & RAID 관리 · 🚑 RAID·LVM 트러블슈팅 치트시트

---
