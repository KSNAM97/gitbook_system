# 🚨 파티션·마운트 트러블슈팅 치트시트

> **Tag:** #Linux #Partition #Mount #fstab #Troubleshooting #CheatSheet  
> **핵심 요약:** 저장장치 장애는 물리·가상 디스크 인식 → 파티션 → 파일시스템 → 마운트 → fstab 순서로 범위를 좁힌다. 진단 첫 단계는 `lsblk -f`, `blkid`, `findmnt`, `df -Th`이며 파괴적인 `mkfs`, `wipefs`, 파티션 삭제는 원인을 확인하기 전에 실행하지 않는다.

---

## 1. 🎯 핵심 진단 흐름

```text
lsblk
→ 디스크와 파티션 존재 확인

lsblk -f / blkid
→ 파일시스템과 UUID 확인

findmnt / df -Th
→ 실제 마운트 확인

findmnt --verify --verbose
→ fstab 검증

journalctl / dmesg
→ 커널·부팅 오류 확인
```

---

## 2. 🛠️ 증상별 즉시 대응표

### 디스크·파티션

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| 새 디스크가 안 보임 | VM 미연결·커널 미검색 | 하이퍼바이저 확인, SCSI 재검색 |
| 파티션 생성 후 장치 없음 | 커널 테이블 미반영 | `partprobe`, `udevadm settle` |
| 2TiB 이상 공간 사용 불가 | MBR 한계 | GPT로 재설계 |
| 파티션 삭제 후 장치 잔존 | 커널이 사용 중 | 마운트·프로세스 확인, 재부팅 검토 |

### 파일시스템

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| `existing filesystem` | 기존 시그니처 | `wipefs -n` 후 데이터 확인 |
| `device is busy` | 마운트·Swap·LVM 사용 | `findmnt`, `swapon`, `pvs` |
| UUID 변경 | 재포맷 | `blkid` 후 fstab 갱신 |
| bad superblock | 손상·장치 오선택 | 타입 확인 후 전용 검사 도구 |

### 마운트

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| `wrong fs type` | 미포맷·타입 오류·손상 | `blkid`, `file -s`, `dmesg` |
| `target is busy` | 열린 파일·현재 경로 | `cd /`, `fuser -vm` |
| 기존 파일이 사라짐 | 마운트로 가려짐 | `umount` 후 확인 |
| 읽기 전용 | `ro` 또는 파일시스템 오류 | `findmnt`, `dmesg` |
| 공간은 있는데 생성 실패 | inode 부족 | `df -i` |

### fstab·부팅

| 증상 | 주요 원인 | 조치 |
|---|---|---|
| 재부팅 후 미마운트 | fstab 누락·UUID 오류 | `blkid`, journal 확인 |
| emergency mode | 필수 fstab 항목 실패 | 콘솔에서 fstab 수정 |
| 장치 대기 지연 | 비필수 장치 미존재 | `nofail`, timeout 검토 |
| mount -a는 성공, 설정 의심 | 이미 수동 마운트됨 | 해제 후 `mount /경로` |

---

## 3. 🔍 대표 복구 시나리오

### 시나리오 1. 새 디스크가 없다

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL
ls /sys/class/scsi_host/
```

재검색:

```bash
for host in /sys/class/scsi_host/host*; do
  echo "- - -" > "$host/scan"
done

udevadm settle
lsblk
dmesg --ctime | tail -n 50
```

`partprobe`는 새 디스크 검색용이 아니다.

---

### 시나리오 2. 파티션은 만들었지만 `/dev/sdb1`이 없다

```bash
fdisk -l /dev/sdb
partprobe /dev/sdb
udevadm settle
lsblk /dev/sdb
```

사용 중인 디스크의 테이블을 변경했다면 재부팅이 필요할 수 있다.

---

### 시나리오 3. `mkfs`가 기존 파일시스템을 발견했다

```bash
wipefs -n /dev/sdb1
blkid /dev/sdb1
lsblk -f /dev/sdb1
```

데이터가 필요하면 중단한다.

정말 삭제할 장치가 맞을 때만:

```bash
wipefs -a /dev/sdb1
```

또는:

```bash
mkfs.xfs -f /dev/sdb1
```

---

### 시나리오 4. `umount: target is busy`

```bash
cd /
findmnt -R /data
fuser -vm /data
```

서비스 정상 종료:

```bash
systemctl stop <서비스>
```

재시도:

```bash
umount /data
```

`kill -9`, `umount -l`, `umount -f`는 첫 번째 조치로 사용하지 않는다.

---

### 시나리오 5. 마운트 후 파일이 사라졌다

```bash
findmnt /data
umount /data
ls -la /data
```

파일이 보이면 마운트에 의해 가려진 것이다. 새 파일시스템으로 이전하려면 임시 마운트와 `rsync` 절차를 사용한다.

---

### 시나리오 6. fstab 오타로 emergency mode

콘솔에서 로그인 후 루트가 읽기 전용이면:

```bash
mount -o remount,rw /
```

백업:

```bash
cp -a /etc/fstab /etc/fstab.recovery
```

수정:

```bash
vi /etc/fstab
```

검증:

```bash
findmnt --verify --verbose
systemctl daemon-reload
mount -a
```

오류가 없을 때:

```bash
reboot
```

---

### 시나리오 7. 재포맷 후 자동 마운트 실패

```bash
blkid /dev/sdb1
grep -n '/data' /etc/fstab
```

새 UUID로 수정한 뒤:

```bash
findmnt --verify --verbose
systemctl daemon-reload
mount /data
findmnt /data
```

---

### 시나리오 8. 파일시스템이 읽기 전용으로 바뀌었다

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /data
dmesg --ctime | tail -n 100
journalctl -k -b
```

I/O 오류가 있으면 디스크 상태와 스토리지 경로를 확인한다.

```bash
smartctl -a /dev/sdb
```

SMART가 제공되지 않는 가상·SAN 환경에서는 하이퍼바이저와 스토리지 로그를 확인한다.

---

## 4. 🔍 긴급 명령 모음

```bash
lsblk -f
blkid
findmnt
df -Th
df -i

findmnt --verify --verbose
journalctl -b
journalctl -k -b

fuser -vm /data
findmnt -R /data
```

> 📌 **핵심 요약**
> - 인식 → 파티션 → 파일시스템 → 마운트 → fstab 순서로 진단
> - 원인 확인 전 `mkfs`, `wipefs -a`, 파티션 삭제 금지
> - busy는 프로세스 정상 종료 후 해제
> - fstab은 재부팅 전 검증
> - 관련: 8-1. 💽 디스크 타입 & 파티션 구조 · 8-2. 🗂️ 파일 시스템 & Format · 8-3. 🔗 마운트 & umount · 8-4. ⚓ Automount · 8-5. 📋 파티션·마운트 통합 정리
