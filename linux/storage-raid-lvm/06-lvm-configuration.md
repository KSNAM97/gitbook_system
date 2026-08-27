# ⚙️ LVM 구성 & 확장·축소 (pvcreate·vgcreate·lvcreate)

> **Tag:** #Linux #LVM #pvcreate #vgcreate #lvcreate #lvextend #lvreduce #resize2fs
> **핵심 요약:** LVM은 `pvcreate`로 PV를 만들고, `vgcreate`로 VG를 묶고, `lvcreate`로 LV를 나눈 뒤 파일시스템을 포맷·마운트한다. 용량이 부족하면 `vgextend`로 디스크를 추가하고 `lvextend` + `resize2fs`(ext4)/`xfs_growfs`(XFS)로 확장한다. 축소는 ext4에서만 가능하며 반드시 마운트 해제 → `e2fsck` → 파일시스템 축소 → LV 축소 순서를 지켜야 하고, XFS는 축소를 지원하지 않는다.

---

## 1. 📖 개요 (Overview)

LVM 생성 명령의 흐름은 `파티션(8e) → pvcreate → vgcreate → lvcreate → mkfs → mount → fstab` 순서로 진행합니다. `pvcreate`는 파티션을 PV로 초기화하고, `vgcreate`는 PV들을 묶어 VG를 만들며, `lvcreate`는 VG 안에서 원하는 크기로 LV를 만듭니다. LV는 일반 블록 장치처럼 `mkfs.ext4`, `mkfs.xfs`로 포맷해 사용합니다.

`lvcreate`에서 크기 지정은 `--size`(고정 용량)와 `--extents`(PE 비율·개수) 두 가지 방식이 있습니다.

```bash
lvcreate --size 8G --name LV1 VGNAME           # 8GB 고정 크기
lvcreate --extents 100%FREE --name LV3 VGNAME  # VG 남은 공간 전부
```

짧은 옵션은 `-L`(대문자, 절대 용량)과 `-l`(소문자, PE 비율·개수)입니다. `100%FREE`는 소문자 `-l`과 함께 사용합니다.

`lvextend`는 **LVM 계층(논리 볼륨)의 크기만** 늘립니다. 그 위의 파일시스템은 자동으로 늘어나지 않으므로 파일시스템 확장 명령을 따로 실행해야 실제 사용 용량이 증가합니다. ext4는 `resize2fs`, XFS는 `xfs_growfs`를 사용하며, 둘 다 마운트된 상태에서 온라인 확장이 가능합니다.

```text
lvextend (LV 확장)
→ resize2fs / xfs_growfs (파일시스템 확장)
→ 실제 용량 증가
```

LV 축소가 위험한 이유는, 파일시스템을 먼저 줄이지 않고 LV를 줄이면 파일시스템이 존재하지 않는 영역을 참조하게 되어 **데이터가 깨질 수 있기** 때문입니다. ext4 축소는 반드시 **마운트 해제 → `e2fsck` 검사 → `resize2fs`로 파일시스템 축소 → `lvreduce`로 LV 축소** 순서를 지켜야 합니다. XFS는 축소 자체를 지원하지 않으므로, 줄여야 하면 백업 후 재생성해야 합니다.

> **참고:** ⚠️ XFS는 온라인 확장은 가능하지만 축소는 불가능합니다. 축소가 필요할 수 있는 볼륨은 ext4를 선택하거나 용량 설계를 신중히 합니다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. LVM 파티션 타입(8e) 생성

```bash
fdisk /dev/sdb                             # sdc도 동일 반복
```

대표 입력:

```text
n       새 파티션
p       Primary
Enter   기본 번호·시작·끝
t       타입 변경
8e      Linux LVM
w       저장
```

확인:

```bash
fdisk -l | grep 8e                         # 8e Linux LVM 파티션 확인
```

### Step 2. PV 생성

```bash
pvcreate /dev/sdb1                          # 파티션을 PV로 초기화
pvcreate /dev/sdc1
pvdisplay                                  # PV 상세 확인
pvs                                        # PV 요약 확인
```

### Step 3. VG 생성

```bash
vgcreate SOLLVM /dev/sdb1 /dev/sdc1        # PV들을 묶어 VG(SOLLVM) 생성
vgs                                        # VG 요약(VSize·VFree) 확인
vgdisplay SOLLVM                           # VG 상세 확인
```

### Step 4. LV 생성

```bash
lvcreate --size 8G --name 8G_LV1 SOLLVM        # 8GB LV
lvcreate --size 6G --name 6G_LV2 SOLLVM        # 6GB LV
lvcreate --extents 100%FREE --name 6G_LV3 SOLLVM   # 남은 공간 전부
lvs                                            # LV 목록 확인
```

LV 장치 경로 확인:

```bash
ls -l /dev/SOLLVM/                          # LV 심볼릭 링크(dm-*) 확인
lsblk -f                                    # LV 매핑 확인
```

### Step 5. 파일시스템 생성·마운트

```bash
mkfs.ext4 /dev/SOLLVM/8G_LV1                # LV를 ext4로 포맷
mkdir /CU
mount /dev/SOLLVM/8G_LV1 /CU               # 마운트
mount | grep LV                            # 마운트 확인
df -h                                      # 용량 확인
```

fstab 등록:

```bash
UUID_CU=$(blkid -s UUID -o value /dev/mapper/SOLLVM-8G_LV1)
echo $UUID_CU

cat <<EOF >> /etc/fstab
UUID=$UUID_CU  /CU  ext4  defaults  0 0
EOF
```

> **참고:** LVM LV는 `/dev/VG/LV` 또는 `/dev/mapper/VG-LV` 두 경로로 접근할 수 있습니다. fstab에는 안정적인 UUID 사용을 권장합니다.

### Step 6. VG 확장(디스크 추가)

새 디스크를 8e 타입으로 만든 뒤 VG에 추가합니다.

```bash
fdisk /dev/sdd                             # n → t → 8e → w
pvcreate /dev/sdd1                          # PV로 초기화
vgextend SOLLVM /dev/sdd1                   # VG에 PV 추가
vgs                                         # VFree 증가 확인
```

### Step 7. LV 확장

```bash
lvextend --size +1G /dev/SOLLVM/8G_LV1     # LV를 1GB 확장
```

ext4 파일시스템 확장(마운트 상태에서 가능):

```bash
resize2fs /dev/SOLLVM/8G_LV1               # ext4 파일시스템 확장
df -h                                      # 확장 결과 확인
```

XFS인 경우:

```bash
lvextend --size +1G /dev/SOLLVM/xfs_lv
xfs_growfs /마운트포인트                    # XFS는 마운트포인트 기준 확장
```

### Step 8. LV 축소 (ext4 전용)

축소는 순서가 중요합니다.

```bash
umount /GS                                 # 1) 마운트 해제
e2fsck -f /dev/SOLLVM/6G_LV2               # 2) 파일시스템 검사
resize2fs /dev/SOLLVM/6G_LV2 5G            # 3) 파일시스템을 5G로 축소
lvreduce --size 5G /dev/SOLLVM/6G_LV2      # 4) LV를 5G로 축소
mount /dev/SOLLVM/6G_LV2 /GS               # 5) 재마운트
```

확인:

```bash
lvs                                        # LV 크기 확인
df -h                                      # 실제 용량 확인
```

> **참고:** ⚠️ 축소 전 반드시 백업합니다. 순서를 어기거나 파일시스템 축소 크기보다 LV를 더 작게 줄이면 데이터가 손상됩니다. XFS에는 이 절차를 적용할 수 없습니다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 확장 후 용량이 그대로다

`lvextend`만 하고 `resize2fs`/`xfs_growfs`를 빠뜨린 경우입니다.

```bash
lvs                                        # LV는 커졌는지 확인
df -h                                      # 파일시스템 용량 확인
resize2fs /dev/VG/LV                       # ext4 파일시스템 확장
```

### 3-2. resize2fs가 e2fsck를 먼저 하라고 한다

```text
Please run 'e2fsck -f /dev/SOLLVM/6G_LV2' first.
```

```bash
umount /GS
e2fsck -f /dev/SOLLVM/6G_LV2               # 검사 후 다시 resize2fs
```

### 3-3. VG에 여유 공간이 없다

```bash
vgs                                        # VFree 확인
pvcreate /dev/sdX1                          # 새 PV 생성
vgextend VGNAME /dev/sdX1                   # VG에 추가
```

### 3-4. LVM 삭제 순서

```text
lvremove → vgremove → pvremove
```

> 📌 **핵심 요약**
> - 파티션(8e) → pvcreate → vgcreate → lvcreate → mkfs → mount
> - LV 확장은 lvextend 후 resize2fs(ext4)/xfs_growfs(XFS) 필수
> - VG 확장은 pvcreate + vgextend
> - ext4 축소는 umount → e2fsck → resize2fs → lvreduce 순서
> - XFS는 확장만 가능, 축소 불가
> - 관련: 🧱 LVM 개념 & 구조 (PV·VG·LV·PE) · 🏗️ 종합실습 LVM 구성 → 확장 → 축소 · 🧩 RAID·LVM 통합 정리
