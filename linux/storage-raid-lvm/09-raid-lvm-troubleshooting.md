# 🚑 RAID·LVM 트러블슈팅 치트시트

> **Tag:** #Linux #Troubleshooting #RAID #LVM #mdadm #CheatSheet
> **핵심 요약:** RAID·LVM 관련 장애를 "증상 → 원인 → 명령어"로 즉시 대응하는 조회용 문서. 반복 사고의 대부분은 **슈퍼블록 미초기화(md127)**, **`lvextend` 후 파일시스템 확장 누락**, **ext4 축소 순서 위반**, **파괴적 명령 대상 착오** 네 가지에서 나온다.

---

## 1. 📖 개요 (Overview)

RAID·LVM 장애에서 가장 먼저 봐야 할 것은 ① **`mdadm --detail` / `pvs·vgs·lvs`로 계층별 상태**, ② **`lsblk -f`로 실제 블록 장치 구조**, ③ **`mount` / `df -h`로 마운트·용량 반영 여부**입니다. 대부분의 장애는 이 세 가지 확인만으로 원인이 드러납니다.

"확장했는데 용량이 그대로다"가 흔한 이유는 `mdadm`·`lvextend`는 **장치(블록) 계층만** 확장하기 때문입니다. 그 위의 **파일시스템은 별도 명령**(`resize2fs`, `xfs_growfs`)으로 확장해야 실제 사용 가능한 용량이 늘어납니다.

---

## 2. 🛠️ 증상별 즉시 대응표 (Configuration)

### 1. RAID
| 증상 | 원인 | 조치 |
|---|---|---|
| 재부팅 후 `/dev/md127`로 잡힘 | 이전 RAID 슈퍼블록 잔존 | `mdadm --stop` → `--zero-superblock` → 재생성 → `/etc/mdadm.conf` + `dracut -fv` |
| `mount \| grep md0` 결과 없음 | fstab 미등록 또는 재부팅 후 마운트 해제 | `blkid`로 UUID 추출 후 fstab 등록, `mount -a` |
| RAID 5 디스크 1개 장애 후 접근 지연 | Spare 복구(rebuild) 진행 중 | `cat /proc/mdstat`로 진행률 확인, 완료까지 대기 |
| RAID 5 디스크 2개 동시 장애 | 결함 허용 초과(패리티 1개) | 복구 불가 → 백업에서 복원, 향후 RAID 6·10 검토 |
| `wipefs`/`--zero-superblock` 실행 후 데이터 손실 | 대상 디스크 착오 | 실행 전 `lsblk`, `mdadm --detail`로 재확인(예방) |

### 2. LVM
| 증상 | 원인 | 조치 |
|---|---|---|
| `lvextend` 후에도 `df -h` 용량 그대로 | 파일시스템 확장 누락 | `resize2fs`(ext4) / `xfs_growfs`(XFS) 실행 |
| `resize2fs`가 e2fsck 먼저 하라고 함 | 축소 전 파일시스템 미검사 | `umount` 후 `e2fsck -f` 실행, 재시도 |
| VG에 여유 공간 없음(`vgs` VFree=0) | 디스크 부족 | 새 디스크 `pvcreate` 후 `vgextend` |
| XFS LV 축소 시도가 실패함 | XFS는 축소 미지원 | 백업 후 더 작은 LV로 재생성, 또는 애초에 ext4 채택 |
| ext4 축소 후 마운트 실패·데이터 손상 | `umount → e2fsck → resize2fs → lvreduce` 순서 위반 | 백업에서 복원, 이후 순서 엄수 |

### 3. 핵심 진단 명령어
```bash
# RAID
cat /proc/mdstat                # 현재 RAID 상태·진행률
mdadm --detail /dev/md0         # RAID 상세 상태
mdadm --examine /dev/sdb1       # 디스크 슈퍼블록
lsblk -f                        # RAID 멤버·파일시스템 확인

# LVM
pvs; vgs; lvs                   # 계층별 요약
pvdisplay; vgdisplay; lvdisplay # 계층별 상세
lsblk -f                        # LV 매핑 확인

# 공통
df -h / df -Th                  # 실제 마운트·용량 확인
findmnt <MOUNTPOINT>            # 마운트 상태 확인
blkid                           # UUID 조회
```

---

## 3. 🔍 트러블슈팅 시나리오 (Verification & Troubleshooting)

### 🚨 시나리오 1. RAID를 `/dev/md0`으로 만들었는데 재부팅 후 `/dev/md127`로 바뀜
```bash
mdadm --stop /dev/md127
mdadm --zero-superblock /dev/sdb1 /dev/sdc1
mdadm --create /dev/md0 --level=linear --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm --detail --scan > /etc/mdadm.conf
dracut -fv
```
- **예방:** RAID 생성 직후 항상 `/etc/mdadm.conf` 저장 + `dracut -fv`를 습관화한다.

### 🚨 시나리오 2. RAID 5에서 디스크 1개 장애 후 Spare가 자동으로 안 붙는 것처럼 보임
```bash
mdadm --detail /dev/md5          # spare rebuilding 상태 확인
cat /proc/mdstat                 # 진행률(%) 확인
```
- **원인:** 실제로는 정상 동작 중이며, 대용량 디스크일수록 복구(rebuild) 시간이 길어 즉시 `active sync`로 안 보일 뿐이다.

### 🚨 시나리오 3. `lvextend -L +2G`를 했는데 `df -h` 용량이 그대로
```bash
lvs                               # LV 크기는 실제로 커졌는지 확인
resize2fs /dev/vg_project/lv_log  # ext4 파일시스템 확장 (누락분 반영)
# 또는 XFS
xfs_growfs /LOG
```

### 🚨 시나리오 4. ext4 LV 축소 중 `resize2fs`가 e2fsck 요구
```bash
umount /GS
e2fsck -f /dev/SOLLVM/6G_LV2
resize2fs /dev/SOLLVM/6G_LV2 5G
lvreduce --size 5G /dev/SOLLVM/6G_LV2
mount /dev/SOLLVM/6G_LV2 /GS
```

### 🚨 시나리오 5. VG에 여유 공간이 없어 LV를 더 만들 수 없음
```bash
vgs                                # VFree 확인(0에 가까움)
pvcreate /dev/sdd1                 # 새 디스크 PV 초기화
vgextend vg_project /dev/sdd1      # VG 확장
vgs                                # VFree 증가 확인
```

> 📌 **핵심 요약**
> - RAID·LVM 장애는 계층별(디스크→RAID/LVM→파일시스템→마운트) 순서로 진단
> - `md127` 예방은 `--zero-superblock` + `/etc/mdadm.conf` + `dracut -fv`
> - 확장은 장치 확장 + 파일시스템 확장이 **한 세트**
> - ext4 축소는 `umount → e2fsck → resize2fs → lvreduce` 순서 고정
> - 파괴적 명령 전 대상 재확인은 모든 시나리오의 공통 예방책
> - 관련: 🧩 RAID 개념 & Hardware vs Software RAID · ⚙️ mdadm 명령어 & RAID 관리 · ⚙️ LVM 구성 & 확장·축소 (pvcreate·vgcreate·lvcreate) · 🧩 RAID·LVM 통합 정리 · ⚡ RAID·LVM 명령어 퀵 레퍼런스
