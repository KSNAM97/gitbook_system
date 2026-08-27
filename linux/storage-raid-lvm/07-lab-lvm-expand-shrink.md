# 🏗️ 종합실습 LVM 구성 → 확장 → 축소

> **Tag:** #Linux #Lab #LVM #pvcreate #vgcreate #lvcreate #lvextend #lvreduce
> **핵심 요약:** 두 개의 디스크를 LVM 파티션(8e)으로 만들어 PV·VG를 구성하고, VG 안에서 여러 LV를 나눠 ext4로 포맷·마운트·fstab 영구화까지 진행하는 종합 실습이다. 이후 새 디스크를 `vgextend`로 추가해 LV를 `lvextend` + `resize2fs`로 확장하고, ext4 LV를 안전한 순서로 축소하는 과정을 검증한다. XFS는 확장만 가능하고 축소가 불가능하다는 점을 함께 확인한다.

---

## 1. 🎯 실습 목표 (Scenario)

### 1-1. 요구사항

`/dev/sdb`, `/dev/sdc`를 LVM으로 묶어 `vg_project` VG를 만들고, 그 안에 `lv_data`(12GB)와 남은 공간 전부를 쓰는 `lv_log`를 생성한다. 두 LV를 ext4로 포맷해 `/DATA`, `/LOG`에 마운트하고 fstab에 등록한다. 이후 새 디스크 `/dev/sdd`를 추가해 `lv_log`를 2GB 확장한다.

```text
/dev/sdb1, /dev/sdc1 → PV → vg_project (VG)
vg_project
├─ lv_data (12GB)    → /DATA
└─ lv_log (100%FREE) → /LOG
추가: /dev/sdd1 → vg_project 확장 → lv_log +2GB
```

---

## 2. 🛠️ 단계별 실습 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### STEP 0. 기존 디스크 초기화

이전 RAID·LVM·파일시스템이 남아 있으면 정리한다.

```bash
mount | grep -E 'md|SOLLVM|vg_'            # 기존 마운트 확인
umount /RAID55                             # 필요 시 해제
mdadm --stop /dev/md5                      # RAID 있으면 중지
```

디스크 시그니처 삭제:

```bash
wipefs -a /dev/sdb
wipefs -a /dev/sdc
wipefs -a /dev/sdd
```

fstab에서 관련 자동 마운트 항목을 삭제한다.

```bash
cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"
vi /etc/fstab                              # 이전 RAID·LVM 항목 삭제
lsblk -f                                   # 초기화 결과 확인
```

> ⚠️ `wipefs`는 데이터 접근을 파괴합니다. 실습용 빈 디스크가 맞는지 확인합니다.

### STEP 1. LVM 파티션(8e) 생성 — EX1

`/dev/sdb`, `/dev/sdc`를 각각 LVM 타입으로 만든다.

```bash
fdisk /dev/sdb                             # sdc도 동일 반복
```

대표 입력:

```text
d       (기존 파티션 있으면) 삭제
n       새 파티션
p       Primary
t       타입 변경
8e      Linux LVM
w       저장
```

확인:

```bash
fdisk -l | grep LVM                        # 8e Linux LVM 확인
```

### STEP 2. PV 생성 — EX2

```bash
pvcreate /dev/sdb1                          # PV 초기화
pvcreate /dev/sdc1
pvdisplay                                  # PV 상세 확인
pvs                                        # PV 요약(PSize·PFree) 확인
```

### STEP 3. VG 생성 — EX3

```bash
vgcreate vg_project /dev/sdb1 /dev/sdc1    # 두 PV를 묶어 VG 생성
vgs                                        # VSize·VFree 확인
vgdisplay vg_project                       # VG 상세 확인
```

### STEP 4. LV 생성 — EX4·EX5

12GB의 `lv_data` 생성:

```bash
lvcreate -L 12G -n lv_data vg_project      # 12GB 고정 크기 LV
lvs                                        # LV 목록 확인
```

남은 공간 전부를 쓰는 `lv_log` 생성:

```bash
lvcreate -l 100%FREE -n lv_log vg_project  # VG 남은 공간 전부
lvs
lvdisplay /dev/vg_project/lv_log           # LV 상세 확인
```

> `-L`(대문자)은 절대 용량, `-l`(소문자)은 PE 비율·개수를 의미합니다. `100%FREE`는 소문자 `-l`과 함께 사용합니다.

### STEP 5. 파일시스템 생성 — EX6

ext4로 포맷:

```bash
mkfs.ext4 /dev/vg_project/lv_data
mkfs.ext4 /dev/vg_project/lv_log
```

(참고) XFS로 만들 경우:

```bash
mkfs.xfs /dev/vg_project/lv_data
mkfs.xfs /dev/vg_project/lv_log
```

### STEP 6. 마운트 — EX7

```bash
mkdir /DATA
mkdir /LOG
mount /dev/vg_project/lv_data /DATA
mount /dev/vg_project/lv_log /LOG

mount | grep vg_project                     # 마운트 확인
df -h | egrep 'DATA|LOG'                    # 용량 확인
```

### STEP 7. fstab 영구 등록 — EX8

```bash
UUID_DATA=$(blkid -s UUID -o value /dev/vg_project/lv_data)
UUID_LOG=$(blkid -s UUID -o value /dev/vg_project/lv_log)
echo $UUID_DATA
echo $UUID_LOG
```

fstab 추가:

```bash
cat <<EOF >> /etc/fstab
UUID=$UUID_DATA  /DATA  ext4  defaults  0 0
UUID=$UUID_LOG   /LOG   ext4  defaults  0 0
EOF
```

검증:

```bash
findmnt --verify --verbose
systemctl daemon-reload
mount -a
findmnt /DATA
findmnt /LOG
```

### STEP 8. VG 확장(디스크 추가) — EX9

새 디스크 `/dev/sdd`를 8e로 만든 뒤 VG에 추가한다.

```bash
fdisk /dev/sdd                             # n → t → 8e → w
pvcreate /dev/sdd1                          # PV 초기화
vgextend vg_project /dev/sdd1               # VG 확장
vgs                                         # VFree 증가 확인
```

### STEP 9. LV 확장 (lv_log +2GB) — EX10

ext4인 경우:

```bash
lvextend -L +2G /dev/vg_project/lv_log     # LV 2GB 확장
resize2fs /dev/vg_project/lv_log            # ext4 파일시스템 확장
df -h | grep LOG                            # 확장 결과 확인
```

XFS인 경우(마운트포인트 기준):

```bash
lvextend -L +2G /dev/vg_project/lv_log
xfs_growfs /LOG
```

> ⚠️ XFS는 확장만 가능하고 축소는 지원하지 않는다.

### STEP 10. LV 축소 (ext4 전용, 선택 실습)

`lv_data`를 예로 축소해 본다. 순서를 반드시 지킨다.

```bash
umount /DATA                                # 1) 마운트 해제
e2fsck -f /dev/vg_project/lv_data           # 2) 파일시스템 검사
resize2fs /dev/vg_project/lv_data 10G       # 3) 파일시스템 10G로 축소
lvreduce --size 10G /dev/vg_project/lv_data # 4) LV 10G로 축소
mount /dev/vg_project/lv_data /DATA         # 5) 재마운트
```

확인:

```bash
lvs                                         # LV 크기 확인
df -h | grep DATA                           # 실제 용량 확인
```

> ⚠️ 축소 전 반드시 백업합니다. 파일시스템을 먼저 줄이지 않고 LV를 줄이면 데이터가 손상됩니다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 계층별 최종 확인

```bash
pvs
vgs
lvs
lsblk -f
findmnt /DATA
findmnt /LOG
df -Th | egrep 'DATA|LOG'
```

### 3-2. 확장했는데 용량이 그대로

```bash
lvs                                         # LV는 커졌는지
df -h                                       # 파일시스템 용량
resize2fs /dev/vg_project/lv_log            # 파일시스템 확장 누락 시
```

### 3-3. VG 여유 공간 부족

```bash
vgs                                         # VFree 확인
pvcreate /dev/sdX1
vgextend vg_project /dev/sdX1
```

---

## 4. ✅ 최종 체크리스트

```text
[ ] sdb·sdc·sdd 실습용 빈 디스크 확인
[ ] 8e LVM 파티션 생성
[ ] pvcreate로 PV 생성
[ ] vgcreate로 vg_project 생성
[ ] lv_data(12G), lv_log(100%FREE) 생성
[ ] ext4 포맷 및 /DATA, /LOG 마운트
[ ] UUID로 fstab 등록 및 findmnt --verify
[ ] sdd 추가 후 vgextend
[ ] lv_log 2G 확장 + resize2fs
[ ] (선택) ext4 축소 순서 검증
```

> 📌 **핵심 요약**
> - 파티션(8e) → PV → VG → LV → 포맷 → 마운트 → fstab
> - VG 확장은 pvcreate + vgextend
> - LV 확장은 lvextend + resize2fs/xfs_growfs
> - ext4 축소는 umount → e2fsck → resize2fs → lvreduce
> - XFS는 축소 불가
> - 관련: 🧱 LVM 개념 & 구조 (PV·VG·LV·PE) · ⚙️ LVM 구성 & 확장·축소 (pvcreate·vgcreate·lvcreate) · 🧩 RAID·LVM 통합 정리
