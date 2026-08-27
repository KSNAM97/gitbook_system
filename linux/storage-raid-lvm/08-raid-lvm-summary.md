# 🧩 RAID·LVM 통합 정리 — 스토리지 관리 한눈에

> **Tag:** #Linux #RAID #LVM #mdadm #Storage #Summary
> **핵심 요약:** RAID는 **결함 허용·성능**을, LVM은 **유연한 용량 관리**를 목적으로 한다. RAID는 `mdadm`으로 `/dev/mdX` 장치를 만들고, LVM은 `pvcreate → vgcreate → lvcreate`로 PV·VG·LV 계층을 쌓는다. 이 문서는 🧩 RAID 개념 & Hardware vs Software RAID~🏗️ 종합실습 LVM 구성 → 확장 → 축소를 한 장으로 닫는 색인이다.

---

## 1. 📖 개요 (Overview)

RAID와 LVM을 관통하는 단 하나의 차이는, RAID는 **디스크 결함에 대비한 데이터 보호·성능** 기술이고 LVM은 **용량을 유연하게 관리**하는 기술이라는 점입니다. RAID는 디스크가 죽어도 데이터를 지키는 것이 목적이고, LVM은 디스크가 부족해질 때 데이터 손실 없이 용량을 늘리는 것이 목적입니다. LVM 단독으로는 결함 허용을 제공하지 않으므로, 안정성이 필요하면 **RAID(보호 계층) 위에 LVM(유연성 계층)** 을 얹는 조합이 실무 표준입니다.

두 기술 모두에서 반복되는 실수 패턴은 ① **확장/생성만 하고 상위 계층(파일시스템)을 갱신하지 않음**(`lvextend` 후 `resize2fs` 누락), ② **슈퍼블록·시그니처를 정리하지 않고 재사용**(RAID `md127` 문제, LVM 잔존 PV), ③ **파괴적 명령 전 대상 확인 생략**(`wipefs`, `--zero-superblock`, `lvreduce`)이 대표적입니다.

---

## 2. 🛠️ 표준 개념 정리 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. RAID 3대 명령 흐름
```text
fdisk(fd) → mdadm --create → mkfs → mount → fstab
mdadm --detail / --examine   → 상태 확인
mdadm --fail / --remove / --add → 장애 대응
mdadm --stop / --zero-superblock → 해체
```

### Step 2. LVM 3대 명령 흐름
```text
fdisk(8e) → pvcreate → vgcreate → lvcreate → mkfs → mount → fstab
vgextend + lvextend + resize2fs/xfs_growfs → 확장
umount + e2fsck + resize2fs + lvreduce → 축소(ext4 전용)
```

### Step 3. 파티션 타입 구분 ★
| 용도 | 타입 코드 | 별칭 |
|---|---|---|
| RAID 멤버 | `fd` | `raid` |
| LVM 멤버 | `8e` | `lvm` |

### Step 4. RAID 레벨 요약
| 레벨 | 최소 디스크 | 결함 허용 | 공간 효율 |
|---|---|---|---|
| Linear | 2 | 없음 | 매우 높음(합산) |
| RAID 0 | 2 | 없음 | 높음 |
| RAID 1 | 2 | 있음(1개) | 낮음(~50%) |
| RAID 5 | 3 | 있음(1개) | 높음(N−1) |
| RAID 6 | 4 | 있음(2개) | 중간(N−2) |

### Step 5. RAID vs LVM 비교
| 구분 | RAID | LVM |
|---|---|---|
| 목적 | 안정성·성능·장애 허용 | 유연한 용량 관리·확장 |
| 데이터 보호 | O (RAID 1·5·6) | X (단독은 보호 없음) |
| 디스크 확장 | 어려움 | 매우 쉬움(LV 확장·축소) |
| 핵심 도구 | `mdadm` | `pvcreate`/`vgcreate`/`lvcreate` |

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 생성 직후 5-Point 검증 (RAID)
```bash
mdadm --detail /dev/md5        # 레벨·상태·구성원
cat /proc/mdstat               # 동기화 진행률
lsblk -f                       # 멤버·UUID
mount | grep md5                # 마운트 확인
df -h | grep md5                # 용량 확인
```

### 3-2. 생성 직후 5-Point 검증 (LVM)
```bash
pvs                             # PV 요약
vgs                             # VG 요약
lvs                             # LV 요약
lsblk -f                        # LV 매핑
df -Th | grep vg_project        # 마운트·용량 확인
```

### 3-3. 대표 함정 요약
| 함정 | 결과 | 정답 |
|---|---|---|
| RAID 해체 후 슈퍼블록 미초기화 | 재부팅 시 `md127` 자동 조립 | `--zero-superblock` + `/etc/mdadm.conf` + `dracut -fv` |
| RAID 5 복구 전 추가 장애 | 데이터 복구 불가 | 이중 장애 대비는 RAID 6·10 |
| `lvextend`만 하고 `resize2fs` 생략 | LV는 커졌는데 실사용 용량 그대로 | `lvextend` 후 파일시스템 확장 명령 필수 |
| ext4 축소 시 순서 위반 | 데이터 손상 | `umount → e2fsck → resize2fs → lvreduce` |
| XFS 축소 시도 | 지원 안 됨(실패) | 축소 필요 볼륨은 ext4 채택 |
| `wipefs`/`--zero-superblock` 대상 착오 | 데이터 파괴 | 실행 전 `lsblk`로 대상 재확인 |

> 📌 **핵심 요약**
> - RAID = 결함 허용·성능, LVM = 유연한 용량 관리
> - 파티션 타입: RAID는 `fd`, LVM은 `8e`
> - 확장 후에는 항상 파일시스템도 함께 확장
> - 파괴적 명령 전 대상 디스크 재확인은 공통 원칙
> - 관련: 🧩 RAID 개념 & Hardware vs Software RAID · ⚙️ mdadm 명령어 & RAID 관리 · ⚙️ LVM 구성 & 확장·축소 (pvcreate·vgcreate·lvcreate) · 🚑 RAID·LVM 트러블슈팅 치트시트 · ⚡ RAID·LVM 명령어 퀵 레퍼런스
