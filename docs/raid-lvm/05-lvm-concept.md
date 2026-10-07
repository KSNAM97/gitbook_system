# LVM 개념 & 구조 (PV·VG·LV·PE)

> **Tag:** #Linux #LVM #PV #VG #LV #PE #Storage #Flexibility
> **핵심 요약:** LVM(Logical Volume Manager)은 여러 물리 디스크·파티션을 하나의 큰 저장 공간(VG)으로 묶고, 그 안에서 원하는 크기로 논리 볼륨(LV)을 나눠 쓰는 유연한 저장소 관리 기술이다. 구성 단위는 물리 볼륨(PV), 볼륨 그룹(VG), 논리 볼륨(LV)이며 할당 최소 단위는 PE(Physical Extent)이다. 데이터 이동 없이 용량을 확장·축소할 수 있지만, LVM 단독으로는 RAID 같은 결함 허용을 제공하지 않는다.

---

## 1. 개요 (Overview)

일반 파티션은 크기가 고정되어 사용 중에 유연하게 조정하기 어렵습니다. LVM은 여러 물리 디스크(또는 파티션)를 **하나의 큰 저장 공간처럼 묶어** 관리하거나, 반대로 큰 공간을 **논리적으로 여러 개로 나눠** 쓸 수 있게 해주는 기술입니다. 핵심은 **데이터 이동 없이 볼륨 크기를 늘리거나 줄일 수 있다**는 점입니다.

LVM은 세 계층과 하나의 할당 단위로 구성됩니다.

- **물리 볼륨(PV, Physical Volume):** 실제 디스크·파티션을 LVM이 쓸 수 있도록 초기화한 단위. `pvcreate`로 생성.
- **볼륨 그룹(VG, Volume Group):** 여러 PV를 묶은 하나의 논리적 저장 공간. `vgcreate`로 생성.
- **논리 볼륨(LV, Logical Volume):** VG 안에서 원하는 크기로 나눈 가상 파티션. 실제 파일시스템을 만들어 마운트하는 단위. `lvcreate`로 생성.
- **PE(Physical Extent):** LVM 공간 할당의 최소 단위(기본 4MB). 여러 PE가 모여 LV를 구성.

```text
물리 디스크/파티션
→ PV (pvcreate)
→ VG (vgcreate, 여러 PV 묶음)
→ LV (lvcreate, 원하는 크기로 분할)
→ 파일시스템 생성 후 마운트
```

LVM용 파티션 타입은 관례적으로 `8e`(Linux LVM)를 사용합니다. RAID용 `fd`와 혼동하지 않습니다.

LVM의 장점은 재파티셔닝 없이 크기를 늘리거나 줄일 수 있고, 용량이 부족하면 VG에 새 디스크를 추가해 확장할 수 있다는 점입니다. RAID보다 구조가 단순해 관리가 쉽습니다. 단점은 LVM 단독으로는 **데이터 미러링(복제)이나 결함 허용을 기본 제공하지 않는다**는 점입니다. VG를 구성하는 디스크 하나가 손상되면 그 위의 LV, 나아가 VG 전체에 영향을 줄 수 있습니다.

RAID와 LVM의 차이는 다음과 같습니다.

| 비교 항목 | RAID | LVM |
|---|---|---|
| 목적 | 안정성·성능·장애 허용 | 유연한 용량 관리·확장 |
| 데이터 보호 | O (RAID 1·5·6·10) | X (단독은 보호 없음) |
| 성능 향상 | RAID 0·10 가능 | 자체 성능 향상은 거의 없음 |
| 장애 허용 | RAID 5·6·10 가능 | 없음(LVM+RAID 조합은 가능) |
| 디스크 묶는 방식 | 스트라이핑·미러링·패리티 | 논리 볼륨으로 합쳐 큰 볼륨 제공 |
| 디스크 확장 | 어려움(종류별 제한) | 매우 쉬움(LV 확장·축소) |
| 사용 목적 | 서버 안정성·성능 중심 | 용량 확장·유연성 중심 |

> **참고:** RAID와 LVM은 목적이 다르므로 함께 쓰기도 합니다. 결함 허용이 필요하면 RAID(또는 LVM RAID)로 보호 계층을 만들고, 그 위에서 LVM으로 용량을 유연하게 관리하는 조합이 일반적입니다.

LVM 구성 절차는 다음과 같습니다.

```text
1. 새 디스크를 LVM 파티션 타입(8e)으로 생성 (fdisk·parted)
2. 파티션을 PV로 초기화        (pvcreate /dev/sdb1)
3. PV들을 묶어 VG 생성         (vgcreate vg_data /dev/sdb1 /dev/sdc1)
4. VG에서 원하는 크기로 LV 생성 (lvcreate -L 10G -n lv_backup vg_data)
5. LV를 파일시스템으로 포맷      (mkfs.ext4 /dev/vg_data/lv_backup)
6. 디렉터리 생성 후 마운트       (mkdir /backup → mount /dev/vg_data/lv_backup /backup)
7. /etc/fstab에 등록해 자동 마운트
```

---

## 2. 표준 확인 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. LVM 상태 확인 명령

```bash
pvs                                        # PV 요약 정보
vgs                                        # VG 요약 정보
lvs                                        # LV 요약 정보
```

상세 정보:

```bash
pvdisplay                                  # PV 상세(소속 VG, PE 수 등)
vgdisplay                                  # VG 상세(전체·여유 용량)
lvdisplay                                  # LV 상세(크기, 경로)
```

전체 구조:

```bash
lsblk -f                                   # LVM2_member·LV 매핑 확인
```

### Step 2. 주요 필드 의미

PV 관련 값에서 `PSize`는 PV 전체 용량, `PFree`는 아직 사용하지 않은 여유 공간입니다. `PE Size`는 PE 1개 크기(기본 4MB), `Total PE`는 생성 가능한 PE 개수, `Free PE`는 아직 LV에 할당되지 않은 PE 개수입니다.

VG 관련 값에서 `#PV`는 VG에 포함된 PV 개수, `#LV`는 생성된 LV 개수, `VSize`는 VG 전체 크기, `VFree`는 아직 사용하지 않은 여유 공간입니다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 계층별 상태 점검

```text
디스크·파티션  → lsblk, fdisk -l
PV            → pvs, pvdisplay
VG            → vgs, vgdisplay
LV            → lvs, lvdisplay
파일시스템     → lsblk -f, blkid
마운트         → findmnt, df -Th
```

### 3-2. LVM 삭제 순서

LVM을 제거할 때는 생성 순서의 역순으로 진행합니다.

```text
LV 삭제(lvremove) → VG 삭제(vgremove) → PV 삭제(pvremove)
```

> **주의:**  LVM 계층은 위에서 아래(LV→VG→PV) 순으로 삭제해야 안전합니다. 하위 계층부터 지우면 참조 오류가 발생할 수 있습니다.

>  **핵심 요약**
> - LVM은 PV → VG → LV 계층으로 유연한 용량 관리 제공
> - PE는 할당 최소 단위(기본 4MB)
> - 데이터 이동 없이 확장·축소 가능
> - LVM 단독은 결함 허용을 제공하지 않음
> - 삭제는 LV → VG → PV 역순
> - 관련:  종합실습 RAID 5 + Spare 구성 & 장애 복구 ·  LVM 구성 & 확장·축소 (pvcreate·vgcreate·lvcreate) ·  종합실습 LVM 구성 → 확장 → 축소
