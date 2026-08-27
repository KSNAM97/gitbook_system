# ⚡ Partition & Mount 명령어 퀵 레퍼런스

> **Tag:** #Linux #Partition #Mount #fstab #XFS #ext4 #Swap #QuickReference  
> **핵심 요약:** 디스크 식별, GPT·MBR 파티션 생성, 파일시스템 포맷, Swap 구성, 임시 마운트, `/etc/fstab` 영구 마운트 및 장애 진단 명령을 빠르게 조회하는 문서. 파티션·포맷 명령은 데이터를 파괴할 수 있으므로 실행 전 `lsblk`, `findmnt`, `wipefs -n`으로 대상을 반드시 확인한다.

---

## 1. 🛠️ 디스크·파티션 확인

### 1-1. 전체 블록 장치

```bash
lsblk                                      # 전체 블록 장치와 계층 구조 확인
lsblk -f                                   # 파일시스템·LABEL·UUID·마운트 정보 확인
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS
                                           # 필요한 필드만 지정해서 출력
```

디스크 모델·일련번호:

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,WWN,TRAN,ROTA
                                           # 디스크 식별자·전송 방식·회전 여부 확인
```

특정 디스크:

```bash
lsblk /dev/sdb                             # /dev/sdb의 디스크·파티션 구조 확인
lsblk -f /dev/sdb                          # /dev/sdb의 파일시스템 정보 확인
fdisk -l /dev/sdb                          # 파티션 테이블·섹터·크기 확인
parted /dev/sdb print                      # 파티션 테이블 종류와 구성 확인
```

파일시스템·UUID:

```bash
blkid                                      # 전체 블록 장치의 UUID·LABEL·TYPE 확인
blkid /dev/sdb1                            # 특정 파티션의 파일시스템 정보 확인
```

기존 시그니처를 삭제하지 않고 확인:

```bash
wipefs -n /dev/sdb                         # 디스크의 파티션·파일시스템 시그니처 조회
wipefs -n /dev/sdb1                        # 파티션의 파일시스템·RAID 시그니처 조회
```

안정적인 장치 경로:

```bash
ls -l /dev/disk/by-id/                     # 장치 ID 기반 심볼릭 링크 확인
ls -l /dev/disk/by-uuid/                   # 파일시스템 UUID 기반 링크 확인
ls -l /dev/disk/by-label/                  # 파일시스템 LABEL 기반 링크 확인
```

> `/dev/sdb` 같은 장치명은 인식 순서에 따라 달라질 수 있다. 운영 환경에서는 모델·일련번호·WWN을 함께 확인한다.

---

## 2. 🛠️ 신규 디스크 재검색

SCSI 호스트 확인:

```bash
ls /sys/class/scsi_host/                   # 시스템의 SCSI 호스트 목록 확인
```

모든 SCSI 호스트 재검색:

```bash
for host in /sys/class/scsi_host/host*; do
  echo "- - -" > "$host/scan"
done
# 모든 채널·대상·LUN에 대해 장치 재검색 요청
```

udev 처리 대기:

```bash
udevadm settle                             # udev 이벤트 처리가 끝날 때까지 대기
```

검증:

```bash
lsblk                                      # 신규 디스크 인식 여부 확인
dmesg --ctime | tail -n 50                 # 최근 커널 장치 인식 메시지 확인
```

> `partprobe`는 새 디스크를 검색하는 명령이 아니라 변경된 파티션 테이블을 커널이 다시 읽도록 요청하는 명령이다.

---

## 3. 🛠️ GPT 파티션 생성

### 3-1. parted 사용

> 다음 명령은 기존 파티션 테이블과 데이터에 영향을 줄 수 있다. 반드시 빈 실습용 디스크인지 확인한다.

GPT 파티션 테이블 생성:

```bash
parted -s /dev/sdb mklabel gpt             # 기존 테이블을 지우고 GPT 생성
```

전체 공간에 파티션 하나 생성:

```bash
parted -s -a optimal /dev/sdb \
  mkpart primary 1MiB 100%
# 1MiB부터 디스크 끝까지 최적 정렬로 파티션 생성
```

> `parted`의 `mkpart`는 파티션만 생성한다. 실제 파일시스템은 `mkfs.xfs`, `mkfs.ext4` 등으로 별도 생성한다.

파티션 테이블 재인식:

```bash
partprobe /dev/sdb                         # 커널에 파티션 테이블 재인식 요청
udevadm settle                             # 파티션 장치 노드 처리 완료 대기
```

검증:

```bash
parted /dev/sdb print                      # GPT와 파티션 구성 확인
lsblk /dev/sdb                             # 커널이 인식한 파티션 구조 확인
```

### 3-2. fdisk 사용

```bash
fdisk /dev/sdb                             # 대화형 파티션 편집 시작
```

대화형 입력:

```text
g       새 GPT 파티션 테이블 생성
n       새 파티션 생성
Enter   기본 파티션 번호 사용
Enter   기본 시작 섹터 사용
Enter   남은 공간 전체 사용
p       구성 확인
w       변경 사항 저장 후 종료
```

저장하지 않고 종료:

```text
q       변경 사항을 저장하지 않고 종료
```

변경 반영:

```bash
partprobe /dev/sdb                         # 파티션 테이블 재인식
udevadm settle                             # udev 이벤트 처리 완료 대기
```

### 3-3. gdisk 사용

```bash
gdisk /dev/sdb                             # GPT 중심 대화형 파티션 편집 시작
```

대표 입력:

```text
n       새 파티션 생성
Enter   기본 파티션 번호
Enter   기본 시작 섹터
Enter   기본 끝 섹터
Enter   기본 파티션 타입
p       파티션 목록 확인
w       변경 사항 저장
```

---

## 4. 🛠️ MBR 파티션 생성

### 4-1. Primary 파티션

```bash
fdisk /dev/sdb                             # MBR 파티션 편집 시작
```

대표 입력:

```text
o       새 DOS/MBR 파티션 테이블 생성
n       새 파티션 생성
p       Primary 파티션 선택
1       파티션 번호 1번 사용
Enter   기본 시작 섹터
+30G    30GiB 할당
p       구성 확인
w       변경 사항 저장
```

### 4-2. Extended·Logical 파티션

대표 구조:

```text
/dev/sdb1 → Primary 파티션
/dev/sdb2 → Extended 파티션
/dev/sdb5 → Logical 파티션
/dev/sdb6 → Logical 파티션
```

대화형 흐름:

```text
n → p → 1 → Enter → +30G
n → e → 2 → Enter → Enter
n → Enter → +20G
n → Enter → +20G
p
w
```

변경 반영:

```bash
partprobe /dev/sdb                         # 변경된 파티션 테이블 재인식
udevadm settle                             # udev 장치 처리 완료 대기
```

검증:

```bash
fdisk -l /dev/sdb                         # MBR 파티션 상세 정보 확인
lsblk /dev/sdb                            # 디스크와 파티션 계층 구조 확인
```

> Primary·Extended·Logical 구분은 MBR 파티션 테이블에 해당한다. GPT에서는 Extended·Logical 구분 없이 일반 파티션을 생성한다.

> 신규 서버는 특별한 호환성 요구가 없다면 MBR보다 GPT를 우선 검토한다.

---

## 5. 🛠️ 파티션 삭제

사용 상태 확인:

```bash
lsblk -f /dev/sdb                         # 파일시스템과 마운트 상태 확인
findmnt -S /dev/sdb1                      # 대상 파티션의 마운트 여부 확인
swapon --show                             # 활성 Swap 목록 확인
```

마운트 해제:

```bash
umount /dev/sdb1                          # 대상 파티션의 마운트 해제
```

Swap이면:

```bash
swapoff /dev/sdb1                         # 대상 Swap 파티션 비활성화
```

파티션 삭제:

```bash
fdisk /dev/sdb                             # 대화형 파티션 편집 시작
```

```text
p       현재 파티션 목록 확인
d       파티션 삭제
번호    삭제 대상 파티션 번호
p       삭제 결과 확인
w       변경 사항 저장
```

반영:

```bash
partprobe /dev/sdb                         # 변경된 파티션 테이블 반영
udevadm settle                             # udev 이벤트 처리 완료 대기
lsblk /dev/sdb                             # 파티션 삭제 결과 확인
```

> 파티션 삭제는 파일시스템과 데이터에 대한 정상적인 접근을 불가능하게 만들 수 있다. 대상 디스크와 파티션 번호를 다시 확인한다.

---

## 6. 🛠️ 파일시스템 생성

### 6-1. 포맷 전 확인

```bash
lsblk -f /dev/sdb                         # 대상 디스크의 파일시스템 구조 확인
findmnt -S /dev/sdb1                      # 대상 파티션의 마운트 여부 확인
wipefs -n /dev/sdb1                       # 기존 시그니처를 삭제하지 않고 조회
blkid /dev/sdb1                           # UUID·LABEL·파일시스템 타입 확인

pvs                                       # LVM Physical Volume 사용 여부 확인
lvs                                       # LVM Logical Volume 확인
cat /proc/mdstat                          # Software RAID 구성 상태 확인
swapon --show                             # Swap 사용 여부 확인
```

### 6-2. XFS

```bash
mkfs.xfs /dev/sdb1                        # 기본 설정으로 XFS 파일시스템 생성
mkfs.xfs -L DATA /dev/sdb1                # DATA 레이블로 XFS 생성
```

기존 시그니처를 강제로 덮어쓰기:

```bash
mkfs.xfs -f /dev/sdb1                     # 기존 시그니처를 무시하고 강제 생성
```

> `-f`는 기존 파일시스템을 파괴할 수 있다. 빈 실습용 장치가 확실할 때만 사용한다.

### 6-3. ext4

```bash
mkfs.ext4 /dev/sdb1                       # 기본 설정으로 ext4 파일시스템 생성
mkfs.ext4 -L BACKUP /dev/sdb1             # BACKUP 레이블로 ext4 생성
```

일반 형식:

```bash
mkfs -t ext4 /dev/sdb1                    # mkfs 공통 형식으로 ext4 생성
```

### 6-4. 결과 검증

```bash
lsblk -f /dev/sdb1                        # 생성된 파일시스템과 UUID 확인
blkid /dev/sdb1                           # 파일시스템 식별 정보 확인
file -s /dev/sdb1                         # 장치 내부의 실제 데이터 형식 확인
```

> `mkfs`는 파일시스템을 생성하는 명령이며 장치 전체를 보안 목적으로 완전 삭제하는 명령이 아니다.

---

## 7. 🛠️ Swap 구성

### 7-1. Swap 파티션

```bash
mkswap /dev/sdb2                          # 파티션에 Swap 시그니처 생성
swapon /dev/sdb2                          # Swap 파티션 활성화

swapon --show                             # 활성화된 Swap 목록 확인
free -h                                   # 메모리와 Swap 용량·사용량 확인
```

비활성화:

```bash
swapoff /dev/sdb2                         # Swap 파티션 비활성화
```

fstab:

```fstab
UUID=<swap-uuid>  none  swap  defaults  0 0  # 부팅 시 UUID 기반 Swap 활성화
```

### 7-2. Swap 파일

```bash
fallocate -l 4G /swapfile                 # 4GiB 크기의 Swap 파일 공간 할당
chmod 0600 /swapfile                      # root만 읽기·쓰기 가능하도록 제한
mkswap /swapfile                          # 파일에 Swap 시그니처 생성
swapon /swapfile                          # Swap 파일 활성화
```

검증:

```bash
swapon --show                             # Swap 파일 활성화 여부 확인
free -h                                   # 전체 Swap 용량과 사용량 확인
ls -lh /swapfile                          # Swap 파일 크기와 권한 확인
```

fstab:

```fstab
/swapfile  none  swap  defaults  0 0      # 부팅 시 Swap 파일 자동 활성화
```

> Copy-on-Write 파일시스템에서는 Swap 파일을 만들기 위한 별도 절차가 필요할 수 있다.

---

## 8. 🛠️ 마운트·해제

### 8-1. 마운트

```bash
mkdir -p /data                            # 마운트포인트 디렉터리 생성
ls -la /data                              # 기존 파일이 있는지 확인
```

장치명:

```bash
mount /dev/sdb1 /data                     # 장치명을 사용해 /data에 마운트
```

타입 명시:

```bash
mount -t xfs /dev/sdb1 /data              # XFS 타입을 명시해 마운트
```

UUID:

```bash
mount UUID="<uuid>" /data                 # 파일시스템 UUID로 마운트
```

LABEL:

```bash
mount LABEL=DATA /data                    # 파일시스템 LABEL로 마운트
```

읽기 전용:

```bash
mount -o ro /dev/sdb1 /data               # 읽기 전용으로 마운트
```

보안 옵션:

```bash
mount -o rw,nosuid,nodev,noexec /dev/sdb1 /data
# 읽기·쓰기를 허용하고 SUID·장치 파일·직접 실행 제한
```

> 기존 파일이 있는 디렉터리에 마운트하면 해당 파일은 마운트가 유지되는 동안 가려진다.

### 8-2. 마운트 확인

```bash
findmnt /data                              # 마운트포인트 기준으로 상태 확인
findmnt -S /dev/sdb1                       # 장치 기준으로 마운트 상태 확인
findmnt -T /data/file                      # 파일 경로가 속한 마운트 확인

df -Th /data                               # 파일시스템 종류와 사용량 확인
lsblk -f                                   # 장치·파일시스템·마운트 구조 확인
mountpoint /data                           # 실제 마운트포인트인지 판정
```

현재 옵션:

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS,TARGET /data
# 소스·파일시스템 종류·옵션·대상만 출력
```

읽기·쓰기 테스트:

```bash
touch /data/.mount-test                    # 테스트 파일을 생성해 쓰기 가능 여부 확인
ls -l /data/.mount-test                    # 테스트 파일 생성 결과 확인
rm /data/.mount-test                       # 테스트 파일 삭제
```

### 8-3. 마운트 해제

```bash
umount /data                               # 마운트포인트를 기준으로 해제
```

또는:

```bash
umount /dev/sdb1                           # 장치명을 기준으로 해제
```

검증:

```bash
findmnt /data                              # 출력이 없는지 확인
mountpoint /data                           # 마운트포인트 여부 재확인
```

---

## 9. 🛠️ Busy 오류 진단

먼저 마운트포인트 밖으로 이동:

```bash
cd /                                      # 현재 셸을 마운트포인트 밖으로 이동
```

하위 마운트 확인:

```bash
findmnt -R /data                          # /data 아래의 하위 마운트 재귀 확인
```

사용 프로세스:

```bash
fuser -vm /data                           # 경로를 사용하는 프로세스 확인
lsof /data                                # 마운트포인트를 참조하는 열린 파일 확인
```

서비스 정상 종료:

```bash
systemctl stop <서비스>                   # 관련 서비스를 정상적으로 중지
```

프로세스에 정상 종료 신호:

```bash
kill <PID>                                # 기본 TERM 신호로 정상 종료 요청
```

재시도:

```bash
umount /data                              # 프로세스 종료 후 마운트 해제 재시도
```

최후 수단:

```bash
umount -l /data                           # 경로를 즉시 분리하는 Lazy unmount
```

> `umount -l`과 `kill -9`은 일반적인 첫 번째 조치로 사용하지 않는다.

---

## 10. 🛠️ `/etc/fstab` 영구 마운트

### 10-1. UUID 확인

```bash
blkid /dev/sdb1                           # 등록할 파일시스템 UUID 확인
lsblk -f                                  # UUID·타입·마운트 상태 교차 확인
```

### 10-2. 백업

```bash
cp -a /etc/fstab \
  "/etc/fstab.bak.$(date +%F-%H%M%S)"
# 기존 속성을 보존하며 날짜·시간이 포함된 백업 생성
```

### 10-3. 등록 예시

XFS:

```fstab
UUID=<xfs-uuid>  /data  xfs  defaults  0 0
```

ext4:

```fstab
UUID=<ext4-uuid>  /backup  ext4  defaults  0 2
```

Swap:

```fstab
UUID=<swap-uuid>  none  swap  defaults  0 0
```

비필수 외장 볼륨:

```fstab
UUID=<uuid>  /archive  xfs  defaults,nofail,x-systemd.device-timeout=10s  0 0
```

네트워크 파일시스템:

```fstab
server:/export  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0
```

systemd Automount:

```fstab
UUID=<uuid>  /archive  xfs  nofail,x-systemd.automount,x-systemd.idle-timeout=5min  0 0
```

> `/home`, `/var`, 데이터베이스처럼 시스템과 서비스에 필수적인 볼륨에 `nofail`을 무조건 적용하지 않는다.

> `x-systemd.automount`, `x-systemd.idle-timeout` 등 `x-systemd.*` 옵션은 systemd 표준 마운트 옵션이다. 세부 동작은 `man systemd.mount`를 기준으로 확인한다.

### 10-4. 검증

```bash
findmnt --verify --verbose                 # fstab 문법과 항목 검증
systemctl daemon-reload                   # systemd 설정 다시 읽기
mount -a                                  # fstab의 미마운트 항목 일괄 마운트
```

실제 상태:

```bash
findmnt /data                             # /data 마운트 결과 확인
df -Th /data                              # 파일시스템 종류와 용량 확인
```

fstab 항목 자체 테스트:

```bash
umount /data                              # 기존 수동 마운트 해제
mount /data                               # fstab 항목을 사용해 마운트
findmnt /data                             # 최종 마운트 결과 확인
```

> `fstab` 변경 후 검증하지 않은 상태로 바로 재부팅하지 않는다.

---

## 11. 🛠️ 파일시스템 검사·확장

### 11-1. XFS

읽기 전용 검사:

```bash
umount /data                              # 검사 전 XFS 마운트 해제
xfs_repair -n /dev/sdb1                  # 변경 없이 XFS 손상 여부 검사
```

복구:

```bash
xfs_repair /dev/sdb1                     # 마운트 해제 상태에서 실제 복구 수행
```

확장:

```bash
mount /dev/sdb1 /data                    # XFS 파일시스템 마운트
xfs_growfs /data                         # 마운트포인트 기준으로 XFS 확장
```

검증:

```bash
xfs_info /data                           # XFS 구조와 크기 확인
df -Th /data                              # 확장 결과 확인
```

> XFS 확장 전에 파티션이나 LVM LV 같은 하위 블록 장치의 크기를 먼저 늘려야 한다. XFS는 확장할 수 있지만 축소할 수 없다.

### 11-2. ext4

읽기 전용 검사:

```bash
umount /backup                            # 검사 전 ext4 마운트 해제
e2fsck -fn /dev/sdb2                     # 수정하지 않고 강제 검사
```

실제 복구:

```bash
e2fsck -f /dev/sdb2                      # 마운트 해제 상태에서 실제 검사·복구
```

확장:

```bash
resize2fs /dev/sdb2                      # 하위 장치 크기까지 ext4 확장
```

> ext4 축소는 일반적으로 백업, 마운트 해제, 파일시스템 검사, 파일시스템 축소, 하위 장치 축소 순서로 수행한다.

---

## 12. 🔍 장애 진단 명령

```bash
lsblk -f                                  # 장치·파일시스템·마운트 정보 확인
blkid                                     # UUID·LABEL·파일시스템 타입 확인
findmnt                                   # 현재 마운트 트리 확인
df -Th                                    # 파일시스템 종류와 공간 사용량 확인
df -i                                     # inode 사용량 확인

wipefs -n /dev/sdb1                       # 기존 시그니처 확인
file -s /dev/sdb1                         # 장치 내부의 실제 데이터 형식 확인

findmnt --verify --verbose                # fstab 문법과 참조 항목 검증
journalctl -b                             # 현재 부팅의 전체 시스템 저널 확인
journalctl -k -b                          # 현재 부팅의 커널 로그 확인
dmesg --ctime | tail -n 100               # 최근 커널 메시지 확인
```

SMART:

```bash
smartctl -a /dev/sdb                      # 디스크 SMART 상태와 오류 정보 확인
```

> 가상 디스크·SAN·하드웨어 RAID 환경에서는 SMART 정보가 제공되지 않거나 물리 디스크 상태와 다르게 보일 수 있다.

---

## 13. 📌 표준 작업 순서

```text
1. lsblk로 디스크 식별
2. MODEL·SERIAL·WWN 확인
3. 기존 마운트·파일시스템·LVM·RAID·Swap 확인
4. GPT 파티션 생성
5. partprobe 및 lsblk 검증
6. mkfs로 파일시스템 생성
7. blkid로 UUID 확인
8. 전용 빈 디렉터리에 임시 mount
9. 읽기·쓰기 테스트
10. fstab 백업 및 UUID 등록
11. findmnt --verify --verbose
12. systemctl daemon-reload
13. fstab 항목으로 재마운트
14. findmnt와 df로 최종 확인
```

> 📌 **핵심 요약**
> - 파티션·포맷 전 대상 장치 재확인
> - 신규 시스템은 GPT 우선 검토
> - `partprobe`는 파티션 테이블 재인식용
> - `mkfs`는 파일시스템 생성 명령
> - XFS는 확장 가능·축소 불가
> - ext4 축소는 오프라인 작업
> - 영구 마운트는 UUID 사용 권장
> - fstab 변경 후 검증 없이 재부팅하지 않음
> - 관련: 8-1. 💽 디스크 타입 & 파티션 구조 · 8-2. 🗂️ 파일 시스템 & Format · 8-3. 🔗 마운트 & umount · 8-4. ⚓ Automount · 8-5. 📋 파티션·마운트 통합 정리 · 8-6. 🚨 파티션·마운트 트러블슈팅 치트시트
