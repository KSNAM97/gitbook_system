# ⚓ Automount

> **Tag:** #Linux #fstab #Automount #UUID #Mount #Boot #systemd  
> **핵심 요약:** `/etc/fstab`은 부팅 시 또는 `mount -a` 실행 시 파일시스템을 정해진 위치에 마운트하기 위한 정적 설정 파일이다. 일반적으로 UUID를 사용하고, 편집 후 `findmnt --verify --verbose`, `systemctl daemon-reload`, `mount -a` 및 실제 마운트 상태를 검증해야 한다. `/etc/fstab`의 부팅 마운트와 접근 시점에 연결하는 `autofs`는 서로 다른 개념이다.

---

## 1. 📖 개요 (Overview)

`/etc/fstab`은 다음 정보를 저장한다.

```text
어떤 파일시스템을
어느 디렉터리에
어떤 타입과 옵션으로
언제·어떻게 마운트할 것인가
```

기본 6개 필드:

```text
장치  마운트포인트  파일시스템  옵션  dump  fsck-pass
```

예:

```fstab
UUID=<uuid>  /data  xfs  defaults  0 0
```

UUID를 사용하는 이유는, `/dev/sdb1` 같은 커널 장치명은 환경 변화에 따라 바뀔 가능성이 있기 때문이다.

안정적인 식별 방법:

- `UUID=`
- `LABEL=`
- `/dev/disk/by-id/`
- LVM의 `/dev/mapper/...`

일반적인 파일시스템에는 UUID가 편리하다.

```bash
blkid /dev/sdb1
lsblk -f
```

> **참고:** `/dev/sdX` 사용이 항상 금지되는 것은 아니지만 이동식 장치나 여러 디스크가 있는 환경에서는 UUID가 더 안정적이다.

fstab의 "Automount"와 autofs는 같지 않다.

```text
/etc/fstab
→ 부팅 과정 또는 mount -a 실행 시 정적 마운트

autofs
→ 특정 경로에 접근할 때 요청 기반으로 마운트
→ 일정 시간 미사용 시 자동 해제 가능
```

이 문서의 중심은 `/etc/fstab` 영구 마운트이다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. UUID와 파일시스템 타입 확인

```bash
lsblk -f
blkid /dev/sdb1
```

예:

```text
/dev/sdb1: UUID="..." TYPE="xfs"
```

마운트포인트 생성:

```bash
mkdir -p /data
```

기존 내용 확인:

```bash
ls -la /data
```

수동 테스트:

```bash
mount /dev/sdb1 /data
findmnt /data
touch /data/.test
rm /data/.test
umount /data
```

---

### Step 2. fstab 백업

```bash
cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"
```

편집:

```bash
sudoedit /etc/fstab
# 또는 vi /etc/fstab, vim /etc/fstab
```

---

### Step 3. fstab 기본 예시

XFS 데이터 볼륨:

```fstab
UUID=<xfs-uuid>  /data  xfs  defaults  0 0
```

ext4 데이터 볼륨:

```fstab
UUID=<ext4-uuid>  /backup  ext4  defaults  0 2
```

Swap:

```fstab
UUID=<swap-uuid>  none  swap  defaults  0 0
```

비필수 외장 디스크:

```fstab
UUID=<uuid>  /archive  xfs  defaults,nofail,x-systemd.device-timeout=10s  0 0
```

네트워크 파일시스템 예:

```fstab
server:/export  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0
```

> **주의:** `/home`, `/var`, 데이터베이스 볼륨처럼 서비스에 필수적인 파일시스템에 `nofail`을 무조건 적용하면 마운트 실패 상태로 시스템과 서비스가 기동해 더 큰 데이터 장애를 만들 수 있다. 필수 여부에 따라 선택한다.

---

### Step 4. fstab 6개 필드

| 필드 | 의미 | 예 |
|---|---|---|
| 1 | 장치 식별자 | `UUID=...` |
| 2 | 마운트포인트 | `/data` |
| 3 | 파일시스템 타입 | `xfs`, `ext4`, `swap` |
| 4 | 마운트 옵션 | `defaults`, `nofail` |
| 5 | dump 사용 | 일반적으로 `0` |
| 6 | fsck 순서 | 루트 `1`, ext 계열 일반 `2`, XFS·swap `0` |

일반 옵션:

| 옵션 | 의미 |
|---|---|
| `defaults` | 일반 기본 옵션 묶음 |
| `ro` | 읽기 전용 |
| `rw` | 읽기·쓰기 |
| `noexec` | 직접 실행 제한 |
| `nosuid` | Set-UID·Set-GID 효과 제한 |
| `nodev` | 장치 파일 해석 제한 |
| `nofail` | 마운트 실패를 부팅 필수 실패로 취급하지 않음 |
| `_netdev` | 네트워크 장치 의존 마운트 |
| `x-systemd.device-timeout=` | 장치 대기 시간 |
| `x-systemd.automount` | systemd automount unit 생성 |

정확한 기본 옵션은 다음으로 확인한다.

```bash
man mount
man fstab
man systemd.mount
```

---

### Step 5. 변경 후 검증

문법·구조 검증:

```bash
findmnt --verify --verbose
```

systemd unit 재생성 반영:

```bash
systemctl daemon-reload
```

fstab 항목 마운트:

```bash
mount -a
```

상태 확인:

```bash
findmnt /data
df -Th /data
```

> **참고:** `mount -a`가 아무 출력 없이 종료되어도 원하는 장치가 올바른 위치와 옵션으로 마운트되었는지 `findmnt`로 확인한다.

이미 수동으로 마운트된 상태라면 fstab 항목 자체를 충분히 시험하지 못할 수 있다.

```bash
umount /data
mount /data
findmnt /data
```

`mount /data`는 `/etc/fstab` 항목을 찾아 마운트한다.

---

### Step 6. systemd automount 옵션

접근할 때 마운트하도록 구성:

```fstab
UUID=<uuid>  /archive  xfs  nofail,x-systemd.automount,x-systemd.idle-timeout=5min  0 0
```

반영:

```bash
systemctl daemon-reload
```

automount unit 확인:

```bash
systemctl list-units --type=automount
```

접근:

```bash
ls /archive
```

상태 확인:

```bash
findmnt /archive
```

> **참고:** `x-systemd.automount`는 `/etc/fstab`에서 systemd automount unit을 생성하는 방식이며 전통적인 `autofs` 서비스와 구현이 다르다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
lsblk -f
blkid
findmnt --verify --verbose
systemctl daemon-reload
mount -a
findmnt /data
df -Th /data
```

Swap:

```bash
swapon --show
free -h
```

### 3-2. 재부팅 후 마운트되지 않는다

```bash
grep -n '/data' /etc/fstab
blkid
findmnt --verify --verbose
journalctl -b -u local-fs.target
```

확인:

- UUID 일치
- 파일시스템 타입 일치
- 마운트포인트 존재
- 옵션 철자
- 장치 준비 시간
- 파일시스템 손상

### 3-3. emergency mode

환경에 따라 콘솔 또는 구조 모드에서 복구한다.

루트가 읽기 전용이면:

```bash
mount -o remount,rw /
```

백업과 현재 파일 확인:

```bash
cp -a /etc/fstab /etc/fstab.recovery
vi /etc/fstab
```

문제 항목을 수정하거나 임시 주석 처리한다.

```bash
findmnt --verify --verbose
mount -a
```

오류가 해결된 후:

```bash
systemctl daemon-reload
reboot
```

> **주의:** 루트 파일시스템 구조, initramfs, 암호화, systemd emergency 환경에 따라 복구 명령이 달라질 수 있다.

### 3-4. 재포맷 후 UUID가 달라졌다

```bash
blkid /dev/sdb1
grep -n '/data' /etc/fstab
```

새 UUID로 수정:

```bash
sudoedit /etc/fstab
findmnt --verify --verbose
systemctl daemon-reload
mount -a
```

> 📌 **핵심 요약**
> - fstab은 정적 영구 마운트 설정
> - UUID 사용 권장
> - 변경 전 백업
> - 변경 후 `findmnt --verify --verbose`
> - systemd 환경에서는 `daemon-reload`
> - `mount -a` 후 실제 위치·장치·옵션 확인
> - 관련: 8-1. 💽 디스크 타입 & 파티션 구조 · 8-2. 🗂️ 파일 시스템 & Format · 8-3. 🔗 마운트 & umount
