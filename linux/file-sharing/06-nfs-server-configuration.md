# ⚙️ NFS 서버 구성 (/etc/exports·exportfs·방화벽)

> **Tag:** #Linux #NFS #exports #exportfs #showmount #firewalld #nfsserver  
> **핵심 요약:** NFS 서버는 `nfs-utils` 설치 후 `/etc/exports`에 공유 디렉터리와 허용 클라이언트·옵션을 정의하고, `exportfs -ra`로 반영한 뒤 `nfs-server`를 실행하면 구성된다. 방화벽은 `nfs`와 `rpc-bind` 서비스를 `--permanent`로 허용해야 재부팅·reload 후에도 유지된다. `--permanent` 없이 추가한 규칙은 `firewall-cmd --reload` 시점에 모두 사라지므로 반드시 영구 옵션을 사용한다.

---

## 1. 📖 개요 (Overview)

`/etc/exports`의 문법은 다음과 같다.

```text
형식: 공유디렉터리  클라이언트(옵션,옵션,...)
```

```text
/NFSS     192.168.10.130(rw,no_root_squash,sync)
/NFS_LC   192.168.10.130(rw,no_root_squash,sync)
/NFS_SB   192.168.10.200(rw,no_root_squash,sync)
/backup   192.168.10.0/24(ro,sync,no_subtree_check)
/pub      *(ro,sync)
```

클라이언트 지정 방식은 단일 IP(`192.168.10.130`), 네트워크 대역(`192.168.10.0/24`), 호스트명(`client-l.localdomain`), 와일드카드(`*.localdomain`, `*`)가 가능하다.

가장 흔한 실수는 클라이언트와 괄호 사이에 공백을 넣는 것이다. `192.168.10.130 (rw)`처럼 띄우면 "모든 호스트에 rw" + "해당 IP는 기본 옵션(ro)"으로 해석되어 의도와 완전히 달라진다.

exportfs 명령의 옵션은 다음과 같다.

| 옵션 | 의미 |
|---|---|
| `-a` | `/etc/exports`의 모든 항목을 export |
| `-r` | 설정 파일을 다시 읽어 재export |
| `-ra` | 재읽기 + 전체 적용 (변경 후 표준) |
| `-v` | 현재 export 상태 상세 출력 |
| `-u` | 특정 export 해제 |
| `-ua` | 전체 export 해제 |

```bash
exportfs -v                                # 현재 적용된 export 확인
exportfs -a                                # exports 내용 적용
exportfs -ra                               # 재읽기 후 적용(권장)
exportfs -u 192.168.10.130:/NFSS           # 특정 항목 해제
```

디렉터리가 없으면 다음과 같이 실패한다.

```text
exportfs: Failed to stat /NFSS: No such file or directory
```

Linux 방화벽(firewalld)에 `--permanent` 없이 추가한 규칙은 런타임 전용이라 `--reload`나 재부팅 시 초기화된다.

```bash
firewall-cmd --add-service=nfs             # 런타임 전용(사라짐)
firewall-cmd --permanent --add-service=nfs # 영구 저장(유지)
firewall-cmd --reload                      # 영구 설정을 런타임에 적용
```

```text
--permanent 없이 추가 → --reload 하면 사라짐
--permanent 로 추가   → --reload 해야 적용됨
```

이 두 가지 성질 때문에 "허용했는데 reload 후 목록에서 사라졌다"는 현상이 발생한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 패키지 설치·확인

```bash
dnf install -y nfs-utils                   # NFS 서버·클라이언트 도구
rpm -qa | grep nfs                         # 설치 확인
rpm -qa | grep rpcbind                     # rpcbind 확인
```

---

### Step 2. 방화벽 영구 허용

```bash
firewall-cmd --permanent --add-service=nfs         # NFS 서비스
firewall-cmd --permanent --add-service=rpc-bind    # RPC 포트 매퍼
firewall-cmd --permanent --add-service=mountd      # NFSv3 사용 시
firewall-cmd --permanent --add-port=2049/tcp       # NFS 포트
firewall-cmd --permanent --add-port=111/tcp        # rpcbind
firewall-cmd --permanent --add-port=111/udp
firewall-cmd --reload
```

확인:

```bash
firewall-cmd --list-port                   # 허용 포트
firewall-cmd --list-services               # 허용 서비스
firewall-cmd --list-all                    # 전체 정책
```

잘못 추가한 규칙 삭제:

```bash
firewall-cmd --permanent --remove-port=1049/tcp
firewall-cmd --reload
```

> **참고:** NFSv4만 사용한다면 `nfs` 서비스(2049)만으로 충분하다. `showmount`를 쓰려면 `rpc-bind`와 `mountd`도 열어야 한다.

---

### Step 3. 공유 디렉터리 생성

```bash
mkdir /NFSS                                # 단일 공유 예
mkdir /NFS_LC                              # Client-L 전용
mkdir /NFS_SB                              # Server-B 전용
ls -ld /NFSS /NFS_LC /NFS_SB               # 권한 확인
```

별도 파티션을 공유 지점으로 쓰는 경우:

```bash
mkfs.ext4 /dev/sdb1                        # 4G 파티션 포맷
mkfs.ext4 /dev/sdb2                        # 2G 파티션 포맷
mount /dev/sdb1 /NFS_SB
mount /dev/sdb2 /NFS_LC
df -h | egrep 'NFS_SB|NFS_LC'              # 마운트 확인
lsblk -f                                   # UUID 확인
```

서버 재부팅 후에도 유지하려면 UUID로 fstab에 등록한다.

```bash
UUID_SB=$(blkid -s UUID -o value /dev/sdb1)
UUID_LC=$(blkid -s UUID -o value /dev/sdb2)

cat <<EOF >> /etc/fstab
UUID=$UUID_SB  /NFS_SB  ext4  defaults  0 0
UUID=$UUID_LC  /NFS_LC  ext4  defaults  0 0
EOF

findmnt --verify --verbose
```

---

### Step 4. exports 작성·적용

```bash
ls -l /etc/exports                         # 기본은 빈 파일
cp -a /etc/exports /etc/exports.bak        # 백업
vi /etc/exports
```

```text
/NFS_LC  192.168.10.130(rw,no_root_squash,sync)
/NFS_SB  192.168.10.200(rw,no_root_squash,sync)
```

적용·확인:

```bash
exportfs -ra                               # 재읽기 후 적용
exportfs -v                                # 적용 결과 확인
```

출력 예:

```text
/NFS_LC  192.168.10.130(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
/NFS_SB  192.168.10.200(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

---

### Step 5. 서비스 실행·자동 시작

```bash
systemctl start nfs-server                 # NFS 서버 시작
systemctl enable nfs-server                # 부팅 시 자동 실행
systemctl status nfs-server                # 상태 확인

systemctl start rpcbind
systemctl enable rpcbind
```

설정 변경 후:

```bash
exportfs -ra                               # 대부분 재시작 없이 반영
systemctl restart nfs-server               # 필요 시 재시작
```

`nfs-server`는 `Active: active (exited)`로 표시되는 것이 정상이다. 실제 서비스는 커널 스레드(nfsd)가 담당하기 때문이다.

---

### Step 6. 공유 목록 확인

```bash
showmount -e                               # 로컬 서버의 export 목록
showmount -e 192.168.10.100                # 특정 서버의 export 목록
showmount -a                               # 현재 마운트한 클라이언트 목록
```

출력 예:

```text
Export list for Server-A:
/NFS_SB 192.168.10.200
/NFS_LC 192.168.10.130
```

---

### Step 7. 디렉터리 퍼미션 조정

exports의 `rw`만으로는 부족하고 실제 디렉터리 퍼미션도 맞아야 한다.

```bash
chmod 777 /NFS_LC                          # 실습용 전체 쓰기
chown nobody:nobody /NFS_LC                # all_squash 사용 시
chmod 1777 /NFS_LC                         # 소유자만 삭제 가능(권장)
ls -ld /NFS_LC
```

SELinux 사용 시:

```bash
setsebool -P nfs_export_all_rw on          # NFS 읽기·쓰기 공유 허용
getsebool -a | grep nfs                    # 관련 부울 확인
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 서버 점검 순서

```bash
systemctl status nfs-server rpcbind        # 서비스 상태
exportfs -v                                # export 적용 여부
showmount -e localhost                     # 목록 조회
ss -tulnp | grep 2049                      # 포트 리스닝
firewall-cmd --list-all                    # 방화벽 정책
journalctl -u nfs-server -n 50             # 로그
```

### 3-2. 대표 오류

| 증상 | 원인·조치 |
|---|---|
| `Failed to stat /NFSS` | 공유 디렉터리 미생성 → `mkdir` 후 `exportfs -ra` |
| `exportfs: /NFSS does not support NFS export` | 경로 오타 또는 지원하지 않는 파일시스템 |
| exports가 반영 안 됨 | `exportfs -ra` 누락 |
| `/etc/exports` 경로에 `/` 누락 | `NFSS`가 아니라 `/NFSS`로 절대 경로 작성 |
| reload 후 방화벽 규칙 소실 | `--permanent` 누락 |
| 클라이언트에서 목록 조회 실패 | `rpc-bind`·`mountd` 방화벽 허용 필요 |

### 3-3. 설정 변경 시 표준 절차

```text
vi /etc/exports → exportfs -ra → exportfs -v → showmount -e
```

> 📌 **핵심 요약**
> - `/etc/exports`는 "경로 클라이언트(옵션)" 형식, 괄호 앞 공백 금지
> - 변경 후에는 항상 `exportfs -ra`로 반영
> - 방화벽은 `--permanent` + `--reload` 조합이 필수
> - `nfs-server`의 `active (exited)`는 정상 상태
> - exports 옵션과 디렉터리 퍼미션이 모두 맞아야 쓰기 가능
> - 관련: 📡 NFS 개념 & RPC 동작 원리 · 💻 NFS 클라이언트 마운트 & fstab · 🏗️ 종합실습 다중 클라이언트 NFS 구성
