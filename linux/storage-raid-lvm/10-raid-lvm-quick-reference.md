# RAID·LVM 명령어 퀵 레퍼런스

> **Tag:** #Linux #QuickReference #RAID #mdadm #LVM #pvcreate #vgcreate #lvcreate #CheatSheet
> **핵심 요약:** `mdadm`(RAID)과 `pvcreate/vgcreate/lvcreate`(LVM) 문법을 빠르게 조회하는 암기 카드. 이해가 아니라 "조회·복붙"이 목적이다.

---

## 1. 명령어 문법 (Configuration)

### 1. 파티션 준비

```bash
fdisk /dev/sdb                          # 파티션 생성/타입 지정 (n → t → w)
# RAID 멤버 타입: fd (Linux raid autodetect)
# LVM 멤버 타입: 8e (Linux LVM)
lsblk                                    # 파티션 결과 확인
fdisk -l | grep -E 'raid|LVM'            # 타입 지정 결과 확인
```

### 2. mdadm (RAID 생성/관리)

```bash
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb1 /dev/sdc1
                                          # RAID 0 생성
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
                                          # RAID 1 생성
mdadm --create /dev/md5 --level=5 --raid-devices=4 \
  /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 --spare-devices=1 /dev/sdf1
                                          # RAID 5 + Spare 생성
mdadm --create /dev/md6 --level=6 --raid-devices=4 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1
                                          # RAID 6 생성

mdadm --detail /dev/md0                  # RAID 상세 정보
mdadm --detail --scan                    # 조립된 RAID 요약
mdadm --examine /dev/sdb1                # 디스크 슈퍼블록 확인

mdadm --fail /dev/md5 /dev/sdc1          # 논리적 장애 주입
mdadm --remove /dev/md5 /dev/sdc1        # RAID에서 디스크 제거
mdadm --add /dev/md5 /dev/sdc1           # RAID에 디스크 재추가

mdadm --stop /dev/md0                    # RAID 동작 중지(구성 유지)
mdadm --zero-superblock /dev/sdb1        # 슈퍼블록 초기화
mdadm --zero-superblock --force /dev/sdb1 # 강제 초기화
mdadm --assemble --scan                  # 자동 재조립
mdadm --assemble /dev/md0 /dev/sdb1 /dev/sdc1
                                          # 수동 재조립
mdadm --detail --scan > /etc/mdadm.conf  # RAID 설정 영구 저장
dracut -fv                               # initramfs 갱신 (md127 방지)
```

### 3. LVM (PV·VG·LV 생성/관리)

```bash
pvcreate /dev/sdb1                       # PV 초기화
pvs ; pvdisplay                          # PV 요약·상세

vgcreate vg_project /dev/sdb1 /dev/sdc1  # VG 생성
vgs ; vgdisplay vg_project                # VG 요약·상세
vgextend vg_project /dev/sdd1             # VG 확장(PV 추가)

lvcreate -L 12G -n lv_data vg_project     # 절대 용량으로 LV 생성
lvcreate -l 100%FREE -n lv_log vg_project # VG 남은 공간 전부로 LV 생성
lvs ; lvdisplay /dev/vg_project/lv_data   # LV 요약·상세

lvextend -L +2G /dev/vg_project/lv_log    # LV 확장
resize2fs /dev/vg_project/lv_log          # ext4 파일시스템 확장
xfs_growfs /LOG                           # XFS 파일시스템 확장(마운트포인트 기준)

umount /DATA                              # LV 축소 STEP 1
e2fsck -f /dev/vg_project/lv_data         # LV 축소 STEP 2
resize2fs /dev/vg_project/lv_data 10G     # LV 축소 STEP 3 (ext4 전용)
lvreduce --size 10G /dev/vg_project/lv_data # LV 축소 STEP 4
mount /dev/vg_project/lv_data /DATA       # LV 축소 STEP 5

lvremove /dev/vg_project/lv_data          # LV 삭제
vgremove vg_project                       # VG 삭제
pvremove /dev/sdb1                        # PV 삭제
```

### 4. 파일시스템·마운트·fstab

```bash
mkfs.ext4 /dev/md0                        # ext4 포맷
mkfs.xfs  /dev/vg_project/lv_data         # xfs 포맷
mkdir /RAID55                             # 마운트포인트 생성
mount /dev/md5 /RAID55                    # 임시 마운트

UUID1=$(blkid -s UUID -o value /dev/md5)  # UUID 추출
cat <<EOF >> /etc/fstab
UUID=$UUID1  /RAID55  ext4  defaults  0 0
EOF

findmnt --verify --verbose                # fstab 문법 검증
systemctl daemon-reload                   # systemd 반영
mount -a                                  # fstab 기반 전체 마운트
```

---

## 2. 빠른 조회표 (Configuration)

### 1. 파티션 타입 코드
| 코드 | 별칭 | 용도 |
|---|---|---|
| `fd` | raid | Linux raid autodetect (RAID 멤버) |
| `8e` | lvm | Linux LVM (LVM 멤버) |

### 2. RAID 레벨 최소 디스크·결함 허용
| 레벨 | 최소 디스크 | 결함 허용 | 공간 효율 |
|---|---|---|---|
| Linear | 2 | 없음 | 매우 높음(합산) |
| RAID 0 | 2 | 없음 | 높음 |
| RAID 1 | 2 | 있음(1개) | 낮음(~50%) |
| RAID 5 | 3 | 있음(1개) | N−1 |
| RAID 6 | 4 | 있음(2개) | N−2 |

### 3. RAID `State` 값
| State | 의미 |
|---|---|
| clean | 정상 |
| degraded | 디스크 부족(장애) |
| recovering | 복구(rebuild) 진행 중 |

### 4. LVM `-L`/`-l` 옵션 구분
```text
-L, --size     → 절대 용량 (예: -L 8G)
-l, --extents  → PE 개수/비율 (예: -l 100%FREE)
```

### 5. ext4 vs XFS 확장·축소
| 파일시스템 | 확장 | 축소 |
|---|---|---|
| ext4 | O (`resize2fs`) | O (`umount`+`e2fsck`+`resize2fs`+`lvreduce`) |
| XFS | O (`xfs_growfs`) | X (미지원) |

---

## 3. 검증 명령어 모음 (Verification)

```bash
lsblk -f                                  # 전체 블록 장치·UUID·RAID/LVM 멤버 확인
cat /proc/mdstat                          # RAID 상태·동기화 진행률
mdadm --detail /dev/md0                   # RAID 상세
pvs; vgs; lvs                             # LVM 계층 요약
df -h / df -Th                            # 마운트·용량 확인
findmnt <MOUNTPOINT>                      # 특정 마운트 상태 확인
blkid                                     # UUID 조회
```

>  **핵심 요약**
> - RAID 생성: `mdadm --create --level=<N> --raid-devices=<수>`
> - LVM 생성: `pvcreate → vgcreate → lvcreate`
> - 확장 후 파일시스템 확장 필수: `resize2fs`/`xfs_growfs`
> - ext4 축소: `umount → e2fsck → resize2fs → lvreduce`
> - md127 방지: `--zero-superblock` + `/etc/mdadm.conf` + `dracut -fv`
> - 관련:  RAID 개념 & Hardware vs Software RAID ·  mdadm 명령어 & RAID 관리 ·  LVM 구성 & 확장·축소 (pvcreate·vgcreate·lvcreate) ·  RAID·LVM 통합 정리 ·  RAID·LVM 트러블슈팅 치트시트
