# 종합실습 RAID 5 + Spare 구성 & 장애 복구

> **Tag:** #Linux #Lab #RAID5 #mdadm #Spare #FaultTolerance #fstab
> **핵심 요약:** RAID용 파티션(타입 `fd`)을 만들고 활성 디스크 4개와 Spare 디스크 1개로 RAID 5를 구성한 뒤, ext4 포맷·마운트·`/etc/fstab` 영구화까지 진행하는 종합 실습이다. 이후 `mdadm --fail`로 장애를 주입해 Spare가 자동 투입·복구(rebuild)되는 과정을 확인하고, RAID 5가 디스크 1개 장애까지 결함 허용을 제공하되 2개 이상 동시 장애 시 복구가 불가능함을 검증한다.

---

## 1. 실습 목표 (Scenario)

### 1-1. 요구사항

`/dev/sdb`, `/dev/sdc`, `/dev/sdd`, `/dev/sde`를 활성 디스크로, `/dev/sdf`를 Spare 디스크로 사용해 RAID 5를 구성한다. 활성 디스크 중 하나에 장애가 발생하면 Spare가 즉시 투입되어 RAID 5로 계속 동작해야 한다.

```text
/dev/sdb1 ~ /dev/sde1 → RAID 5 활성 디스크 4개
/dev/sdf1             → Spare(예비) 디스크 1개
/dev/md5              → RAID 5 장치
/RAID55               → 마운트포인트
```

### 1-2. 예상 최종 구조

```text
/dev/md5 (RAID 5, ext4)
├─ /dev/sdb1  active sync
├─ /dev/sdc1  active sync
├─ /dev/sdd1  active sync
├─ /dev/sde1  active sync
└─ /dev/sdf1  spare
```

---

## 2. 단계별 실습 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### STEP 0. 기존 디스크 초기화

이전 RAID나 파일시스템이 남아 있으면 먼저 정리한다.

```bash
mount | grep md                            # 마운트 여부 확인
umount /RAID55                             # 마운트되어 있으면 해제
mdadm --stop /dev/md5                      # RAID 동작 중지
```

슈퍼블록·시그니처 초기화:

```bash
mdadm --zero-superblock --force /dev/sdb1
mdadm --zero-superblock --force /dev/sdc1
mdadm --zero-superblock --force /dev/sdd1
mdadm --zero-superblock --force /dev/sde1
mdadm --zero-superblock --force /dev/sdf1

wipefs -a /dev/sdb
wipefs -a /dev/sdc
wipefs -a /dev/sdd
wipefs -a /dev/sde
wipefs -a /dev/sdf
```

>  이 명령들은 대상 디스크의 데이터 접근을 파괴합니다. 실습용 빈 디스크가 맞는지 `lsblk`, `mdadm --detail`로 반드시 먼저 확인합니다.

### STEP 1. RAID용 파티션 생성

각 디스크에 파티션을 만들고 타입을 `fd`(Linux raid autodetect)로 지정한다.

```bash
fdisk /dev/sdb                             # sdc·sdd·sde·sdf도 동일 반복
```

대표 입력 흐름:

```text
n       새 파티션
p       Primary
1       파티션 번호
Enter   기본 시작 섹터
Enter   남은 공간 전체 사용
t       파티션 타입 변경
fd      Linux raid autodetect
w       저장
```

확인:

```bash
lsblk                                      # sdb1 ~ sdf1 파티션 생성 확인
fdisk -l | grep raid                       # raid autodetect 타입 확인
```

### STEP 2. RAID 5 + Spare 생성

```bash
mdadm --create /dev/md5 \
  --level=5 --raid-devices=4 \
  /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 \
  --spare-devices=1 /dev/sdf1
```

`write-intent bitmap` 활성화 여부를 물으면 `y`를 선택한다.

생성 직후 확인:

```bash
mdadm --detail --scan                      # RAID 요약(spares=1 확인)
mdadm --detail /dev/md5                    # 상세 상태 확인
lsblk -f                                   # 멤버 UUID 확인
```

`mdadm --detail /dev/md5` 출력에서 활성 4개는 `active sync`, Spare 1개는 `spare`로 표시된다. 생성 직후에는 초기 동기화 때문에 `State`가 `clean, degraded, recovering`으로 나타날 수 있다.

### STEP 3. md127 문제 방지 설정

```bash
mdadm --detail --scan > /etc/mdadm.conf    # RAID 구성 정보 저장
dracut -fv                                 # initramfs 갱신
```

> 이 단계를 생략하면 재부팅 후 `/dev/md127` 등 다른 번호로 자동 조립될 수 있다. (참고:  mdadm 명령어 & RAID 관리)

### STEP 4. 파일시스템 생성

```bash
mkfs.ext4 /dev/md5                         # RAID 장치를 ext4로 포맷
lsblk -f                                   # md5의 ext4 UUID 확인
```

### STEP 5. 마운트

```bash
mkdir /RAID55                              # 마운트포인트 생성
mount /dev/md5 /RAID55                     # 임시 마운트
mount | grep md5                           # 마운트 상태 확인 (stripe=384 등 옵션 표시)
df -h | grep md5                           # 용량 확인
```

### STEP 6. `/etc/fstab` 영구 등록

```bash
UUID1=$(blkid -s UUID -o value /dev/md5)   # md5의 파일시스템 UUID 추출
echo $UUID1                                # UUID 값 확인

cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"   # 백업
```

fstab에 추가:

```bash
cat <<EOF >> /etc/fstab
UUID=$UUID1  /RAID55  ext4  defaults  0 0
EOF
```

검증:

```bash
findmnt --verify --verbose                 # fstab 문법·항목 검증
systemctl daemon-reload                    # systemd 반영
umount /RAID55
mount -a                                   # fstab 기반 마운트
findmnt /RAID55                            # 최종 확인
```

> RAID 자동 마운트를 fstab에 등록한 상태에서 멤버 디스크를 제거하고 재부팅하면 emergency mode로 진입할 수 있다. 디스크를 물리적으로 제거·교체하는 실습을 할 때는 이 점을 인지한다.

### STEP 7. 테스트 데이터 복사

```bash
cp -r /etc/c* /RAID55                      # /etc의 c로 시작하는 파일 복사
ls -l /RAID55                              # 복사 결과 확인
```

---

## 3. 장애 복구 검증 (Verification & Troubleshooting)

### 3-1. 디스크 1개 장애 → Spare 자동 투입

활성 디스크 하나에 장애를 주입한다.

```bash
mdadm --fail /dev/md5 /dev/sdd1            # sdd1을 논리적 장애로 표시
mdadm --detail /dev/md5                    # 상태 확인
```

이때 Spare였던 `/dev/sdf1`이 `spare rebuilding` 상태로 바뀌며 복구를 시작한다.

```text
State : clean, degraded, recovering
Rebuild Status : 3% complete

/dev/sdf1  spare rebuilding   ← Spare가 RAID 5로 투입
/dev/sdd1  faulty
```

복구가 완료되면 `/dev/sdf1`은 `active sync`로 전환된다.

```bash
cat /proc/mdstat                           # 복구 진행률 실시간 확인
ls -l /RAID55                              # 장애 중에도 데이터 정상 확인
```

RAID 5는 디스크 1개 장애 시 패리티로 데이터를 유지하므로 파일이 정상적으로 보인다.

### 3-2. 장애 디스크 제거·재추가

```bash
mdadm --remove /dev/md5 /dev/sdd1          # faulty 디스크 RAID에서 제거
mdadm --add /dev/md5 /dev/sdd1             # 교체·복구된 디스크 다시 추가
mdadm --detail /dev/md5                    # 재추가 후 상태 확인
```

슈퍼블록이 남아 있으면 `--add`로 다시 넣을 수 있고, 이 디스크는 새로운 Spare 또는 복구 대상이 된다.

### 3-3. 디스크 2개 동시 장애 → 복구 불가

Spare 복구가 끝난 뒤 추가로 또 하나의 활성 디스크에 장애가 나면 RAID 5는 결함 허용을 초과한다.

```bash
mdadm --fail /dev/md5 /dev/sdb1            # 추가 장애 주입
mdadm --detail /dev/md5                    # removed·faulty 다수 확인
ls -l /RAID55                              # 접근 실패 확인
```

RAID 5는 패리티가 1개이므로 동시에 디스크 2개가 손실되면 데이터를 복구할 수 없다. 이때 파일 접근이 불가능해질 수 있다.

>  RAID 5는 디스크 1개 장애까지만 결함 허용을 제공한다. 복구(rebuild)가 끝나기 전이나 동시에 2개가 고장 나면 데이터를 잃는다. 이중 장애 대비가 필요하면 RAID 6 또는 RAID 10을 검토한다.

---

## 4. 최종 체크리스트

```text
[ ] sdb ~ sdf 모델·용량 확인, 실습용 빈 디스크 확인
[ ] RAID용 파티션(fd) 생성
[ ] RAID 5 + Spare 1개 생성
[ ] mdadm --detail로 active 4 + spare 1 확인
[ ] /etc/mdadm.conf 저장 및 dracut -fv
[ ] ext4 포맷 및 임시 마운트
[ ] UUID로 fstab 등록 및 findmnt --verify
[ ] 테스트 데이터 복사
[ ] --fail로 장애 주입 후 Spare 자동 복구 확인
[ ] 장애 중 데이터 정상 접근 확인
[ ] 2개 동시 장애 시 복구 불가 확인
```

>  **핵심 요약**
> - RAID 5는 활성 N개 + Spare로 구성 가능
> - Spare는 활성 디스크 장애 시 자동 투입·복구
> - 디스크 1개 장애까지 결함 허용, 2개 동시 장애는 복구 불가
> - md127 방지를 위해 `/etc/mdadm.conf` + `dracut -fv`
> - 파괴적 명령 전 대상 디스크 재확인
> - 관련:  RAID 개념 & Hardware vs Software RAID ·  RAID 레벨별 특징 (Linear·0·1·5·6) ·  mdadm 명령어 & RAID 관리 ·  LVM 개념 & 구조 (PV·VG·LV·PE)
