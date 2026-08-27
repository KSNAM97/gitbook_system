# ⚙️ mdadm 명령어 & RAID 관리

> **Tag:** #Linux #mdadm #RAID #Superblock #Assemble #md127 #Storage
> **핵심 요약:** `mdadm`(Multiple Device Admin)은 Linux에서 Software RAID를 생성·확인·복구·모니터링·해체하는 대표 도구이다. 생성은 `--create`, 상세 확인은 `--detail`, 슈퍼블록 확인은 `--examine`, 장애 주입·제거·추가는 `--fail`·`--remove`·`--add`로 수행한다. RAID 해체 시 슈퍼블록이 남으면 재부팅 후 `/dev/md127`로 자동 조립되는 문제가 생길 수 있으므로 `--zero-superblock`으로 초기화하고 `/etc/mdadm.conf`와 `dracut -fv`로 반영한다.

---

## 1. 📖 개요 (Overview)

`mdadm`은 Linux에서 RAID 디스크 어레이(RAID array)를 만들고 관리하는 명령입니다. RAID 0·1·5·6 등의 레벨을 지원하며, 여러 디스크를 묶어 `/dev/md0` 같은 하나의 가상 디스크처럼 사용할 수 있게 합니다. 주요 기능은 생성(`--create`), 상세 확인(`--detail`), 재조립(`--assemble`), 제거(`--remove`), 모니터링(`--monitor`) 등입니다. 하드웨어 RAID가 컨트롤러 장비로 RAID를 구성한다면, `mdadm` 기반 Software RAID는 **운영체제가 디스크를 묶어** RAID 기능을 제공합니다.

Software RAID를 만들 때 가장 많이 쓰는 명령이 `--create`입니다.

```text
형식: mdadm --create --verbose [RAID장치] --level=[레벨] --raid-devices=[디스크수] [디스크목록]
```

```bash
mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
```

`/dev/md0`은 새로 생성되는 RAID 장치, `--level=1`은 RAID 1(미러링), `--raid-devices=2`는 사용 디스크 개수, 뒤의 경로는 실제로 묶을 디스크입니다. 생성 후에는 `/dev/md0`을 `ext4`·`xfs` 등으로 포맷한 뒤 마운트해서 사용합니다. 파티션(`/dev/sdb1`)이 아니라 디스크 전체(`/dev/sdb`)를 그대로 멤버로 쓰는 것도 가능합니다.

RAID 정보 확인에서 `--detail`은 RAID 장치의 **전체 구성 정보**를, `--examine`은 **개별 디스크에 저장된 슈퍼블록**을 보여줍니다.

```bash
mdadm --detail /dev/md0       # RAID 레벨·상태·Sync·구성원 등 전체 정보
mdadm --examine /dev/sdb      # 해당 디스크의 RAID 슈퍼블록(어느 array 소속인지) 확인
```

`--detail` 출력에는 `Raid Level`, `Array Size`, `Raid Devices`, `Active Devices`, `Failed Devices`, `Spare Devices`, `State`, `UUID`, 구성원별 `State`(active sync / spare / faulty) 등이 담깁니다.

Superblock(슈퍼블록)은 파일시스템이나 RAID에 대한 핵심 정보를 저장한 작은 메타데이터 블록입니다. RAID 슈퍼블록에는 **RAID 레벨, RAID UUID, 디스크 순서, 구성 정보** 등이 담겨 재부팅 후에도 시스템이 RAID를 자동 조립할 수 있게 합니다. 반대로 RAID를 해체한 뒤에도 슈퍼블록이 남아 있으면 **예기치 않은 자동 조립 문제**(`md127` 등)가 발생할 수 있습니다.

`/dev/md127`로 자동 생성되는 이유는, 원래 `/dev/md0`으로 만들었더라도 디스크에 **이전 RAID 정보(슈퍼블록)** 가 남아 있으면 부팅 시 시스템이 원래 이름·설정 정보를 정확히 찾지 못해 임시 장치 번호인 **`/dev/md127`** 을 사용하기 때문입니다.

```bash
mdadm --stop /dev/md127                    # 자동 조립된 RAID 정지
mdadm --zero-superblock /dev/sdb1          # 슈퍼블록 초기화
mdadm --zero-superblock /dev/sdc1
mdadm --create /dev/md0 --level=linear --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm --detail --scan > /etc/mdadm.conf    # 현재 RAID 정보를 설정 파일에 저장
dracut -fv                                 # initramfs 갱신
```

`initramfs`는 부팅 초기에 사용하는 임시 루트 파일시스템으로 RAID·LVM·파일시스템 모듈 등을 포함합니다. `dracut`은 이 이미지를 생성·갱신하는 프로그램이며, `-f`는 강제 재생성, `-v`는 상세 출력입니다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. RAID 생성과 확인

```bash
mdadm --create /dev/md0 --level=5 --raid-devices=4 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1
                                            # RAID 5 생성
mdadm --detail --scan                      # 생성된 RAID 요약 확인
mdadm --detail /dev/md0                    # 상세 정보 확인
lsblk -f                                   # RAID 멤버·UUID 확인
```

> **주의:** RAID 생성 시 `write-intent bitmap` 활성화 여부를 묻는 경우 `y`를 선택하면 복구 속도 최적화에 도움이 됩니다.

### Step 2. Spare(예비) 디스크 포함 생성

```bash
mdadm --create /dev/md5 \
  --level=5 --raid-devices=4 \
  /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 \
  --spare-devices=1 /dev/sdf1
                                            # 활성 4개 + Spare 1개로 RAID 5 생성
```

Spare 디스크는 평소 대기하다가 활성 디스크 장애 시 자동으로 투입되어 복구(rebuild)를 시작합니다.

### Step 3. RAID 모니터링

```text
형식: mdadm --monitor --scan
```

디스크 장애 감지, RAID Sync 진행률 감시, 로그 기록(`/var/log/messages` 등)이 주요 기능이며, 상태 변화 시 이메일 발송·syslog 기록을 설정할 수 있습니다.

### Step 4. 장애 주입·제거·추가

```bash
mdadm --fail /dev/md5 /dev/sdc1            # 해당 디스크를 논리적 장애로 표시
mdadm --remove /dev/md5 /dev/sdc1          # RAID에서 디스크 제거
mdadm --add /dev/md5 /dev/sdc1             # 디스크를 RAID에 다시 추가
```

`--fail`은 장애 상황을 인위적으로 만들어 결함 허용을 테스트할 때, `--remove`는 실제 RAID에서 디스크를 분리할 때 사용합니다. 슈퍼블록이 남아 있으면 `--add`로 다시 추가할 수 있습니다.

### Step 5. RAID 중지·해체·슈퍼블록 초기화

```bash
umount /RAID5                              # 마운트되어 있으면 먼저 해제
mdadm --stop /dev/md5                      # RAID 동작 중지(구성 삭제 아님)
mdadm --zero-superblock /dev/sdb1          # 디스크 슈퍼블록 초기화
mdadm --zero-superblock --force /dev/sdb1  # 강제 초기화
wipefs -a /dev/sdb                         # 디스크 전체 시그니처 삭제
```

> **주의:** ⚠️ **파괴적 명령 주의:** `wipefs -a`와 `--zero-superblock`은 데이터 접근을 파괴할 수 있습니다. 실행 전 반드시 `lsblk`, `mdadm --detail`로 대상 디스크가 맞는지 확인합니다. `--stop`은 현재 조립된 RAID의 **동작만 중지**하며 구성 자체를 삭제하지 않습니다.

### Step 6. RAID 복구(assemble)

```bash
mdadm --assemble --scan                    # 슈퍼블록을 읽어 자동 재조립
mdadm --assemble /dev/md0 /dev/sdb1 /dev/sdc1
                                            # 수동으로 구성원을 명시해 조립
```

`/etc/mdadm.conf`에 RAID 설정이 등록되어 있으면 자동 조립이 더 안정적입니다.

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 확인 명령

```bash
cat /proc/mdstat                           # 현재 RAID 상태·진행률
mdadm --detail /dev/md0                    # 상세 상태
mdadm --examine /dev/sdb1                  # 디스크 슈퍼블록
lsblk -f                                   # RAID 멤버·파일시스템 확인
```

### 3-2. 재부팅 후 md127로 자동 조립됨

```bash
mdadm --stop /dev/md127                    # 잘못 조립된 RAID 정지
mdadm --zero-superblock /dev/sdb1 /dev/sdc1
mdadm --create /dev/md0 --level=linear --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm --detail --scan > /etc/mdadm.conf
dracut -fv
```

### 3-3. 상태(State) 값 해석

| State | 의미 |
|---|---|
| clean | 정상, 일관성 있음 |
| degraded | 디스크 부족(장애), 결함 허용 소진 위험 |
| recovering | Spare 등으로 복구(rebuild) 진행 중 |
| clean, degraded, recovering | 데이터 일관성은 있으나 복구 진행 중 |

### 3-4. 시나리오. `mount | grep md0`에 아무것도 안 뜬다

- **원인:** 재부팅 후 `/etc/fstab`에 자동 마운트 항목이 없으면 RAID가 조립되어도 **마운트는 자동으로 되지 않습니다.**
- **해결:**
  ```bash
  UUID1=$(blkid -s UUID -o value /dev/md0)   # RAID 위 파일시스템 UUID 추출
  echo $UUID1
  cat <<EOF >> /etc/fstab
  UUID=$UUID1  /linear  ext4  defaults  0 0
  EOF
  mount -a
  mount | grep md0
  ```

### 3-5. RAID 삭제 순서

```text
umount → mdadm --stop → mdadm --zero-superblock → wipefs -a → fstab 정리
```

> **참고:** RAID 관련 자동 마운트를 `/etc/fstab`에 등록한 채 디스크를 제거하면 재부팅 시 emergency mode로 진입할 수 있습니다. RAID를 해체할 때는 관련 fstab 항목도 함께 정리합니다.

> 📌 **핵심 요약**
> - `--create`로 생성, `--detail`·`--examine`으로 확인
> - 슈퍼블록은 RAID 구조를 기록한 메타데이터
> - 해체 후 슈퍼블록이 남으면 `md127` 자동 조립 문제 발생
> - `--zero-superblock` + `/etc/mdadm.conf` + `dracut -fv`로 방지
> - 파괴적 명령 전 대상 디스크 재확인
> - 관련: 🧩 RAID 개념 & Hardware vs Software RAID · 📊 RAID 레벨별 특징 (Linear·0·1·5·6) · 🏗️ 종합실습 RAID 5 + Spare 구성 & 장애 복구
