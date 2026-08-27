# 🏗️ 종합실습 다중 클라이언트 NFS 구성

> **Tag:** #Linux #Lab #NFS #exports #Partition #fstab #MultiClient  
> **핵심 요약:** 추가 디스크에 여러 파티션을 만들어 각각 ext4로 포맷하고, 클라이언트별 전용 공유 디렉터리(`/NFS_SB`, `/NFS_LC`)로 마운트한 뒤 `/etc/exports`에서 클라이언트 IP별로 서로 다른 공유를 제공하는 실습이다. Server-B와 Client-L이 각각 지정된 용량만 사용하도록 구성하고, 양쪽 모두 fstab에 등록해 재부팅 후에도 마운트가 유지되는지 검증한다.

---

## 1. 🎯 실습 목표 (Scenario)

### 1-1. 구성도

```text
Server-A (192.168.10.100)  NFS Server
├─ /dev/sdb1 (4G, ext4) → /NFS_SB → Server-B 전용
└─ /dev/sdb2 (2G, ext4) → /NFS_LC → Client-L 전용

Server-B (192.168.10.200)  NFS Client → /NFS_SBC
Client-L (192.168.10.130)  NFS Client → /NFS_CLIENT
```

### 1-2. 요구사항

```text
[1] Server-A에 10G HDD 추가 후 4G·2G·2G·1G·1G 파티션 구성
[2] 4G(sdb1), 2G(sdb2)를 ext4로 포맷
[3] Server-B는 4G 공유(/NFS_SB) 사용
[4] Client-L은 2G 공유(/NFS_LC) 사용
[5] 양쪽 모두 재부팅 후에도 마운트 유지
```

---

## 2. 🛠️ 단계별 실습 - 서버 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### STEP 1. 디스크·파티션 확인

```bash
lsblk                                      # 추가 디스크 인식 확인
fdisk -l /dev/sdb                          # 파티션 테이블 확인
```

```text
sdb      8:16  0  100G  0 disk
├─sdb1   8:17  0    4G  0 part
├─sdb2   8:18  0    2G  0 part
├─sdb3   8:19  0    2G  0 part
├─sdb4   8:20  0     1K 0 part      ← 확장 파티션
├─sdb5   8:21  0    1G  0 part
└─sdb6   8:22  0    1G  0 part
```

> MBR에서는 주 파티션이 최대 4개이므로 5개 이상 만들 때 확장 파티션(1K로 표시)과 논리 파티션(sdb5~)이 생긴다.

---

### STEP 2. 파일시스템 생성

```bash
mkfs.ext4 /dev/sdb1                        # 4G 파티션 포맷
mkfs.ext4 /dev/sdb2                        # 2G 파티션 포맷
lsblk -f                                   # UUID·파일시스템 확인
```

---

### STEP 3. 패키지·방화벽 확인

```bash
rpm -qa | grep nfs                         # nfs-utils 확인
rpm -qa | grep rpc                         # rpcbind 확인
dnf install -y nfs-utils                   # 없으면 설치
```

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --permanent --add-port=2049/tcp
firewall-cmd --permanent --add-port=111/tcp
firewall-cmd --permanent --add-port=111/udp
firewall-cmd --reload
firewall-cmd --list-all                    # 정책 확인
```

---

### STEP 4. 공유 디렉터리 생성·마운트

```bash
mkdir /NFS_SB                              # Server-B 전용
mkdir /NFS_LC                              # Client-L 전용

mount /dev/sdb1 /NFS_SB
mount /dev/sdb2 /NFS_LC
df -h | egrep 'NFS_SB|NFS_LC'              # 용량 확인
```

서버 재부팅 대비 fstab 등록:

```bash
UUID_SB=$(blkid -s UUID -o value /dev/sdb1)
UUID_LC=$(blkid -s UUID -o value /dev/sdb2)
echo $UUID_SB ; echo $UUID_LC

cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"

cat <<EOF >> /etc/fstab
UUID=$UUID_SB  /NFS_SB  ext4  defaults  0 0
UUID=$UUID_LC  /NFS_LC  ext4  defaults  0 0
EOF

findmnt --verify --verbose
systemctl daemon-reload
```

---

### STEP 5. 퍼미션 설정

```bash
chmod 1777 /NFS_SB                         # 소유자만 삭제 가능
chmod 1777 /NFS_LC
ls -ld /NFS_SB /NFS_LC                     # drwxrwxrwt 확인
```

SELinux 사용 시:

```bash
setsebool -P nfs_export_all_rw on
```

---

### STEP 6. exports 작성

```bash
cp -a /etc/exports /etc/exports.bak
vi /etc/exports
```

```text
/NFS_LC  192.168.10.130(rw,no_root_squash,sync)
/NFS_SB  192.168.10.200(rw,no_root_squash,sync)
```

적용:

```bash
exportfs -ra                               # 재읽기 후 적용
exportfs -v                                # 결과 확인
```

```text
/NFS_LC  192.168.10.130(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
/NFS_SB  192.168.10.200(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

---

### STEP 7. 서비스 실행·공개 확인

```bash
systemctl enable --now rpcbind
systemctl enable --now nfs-server
systemctl status nfs-server

showmount -e                               # 로컬 확인
showmount -e 192.168.10.100                # IP로 확인
```

```text
Export list for Server-A:
/NFS_SB 192.168.10.200
/NFS_LC 192.168.10.130
```

---

## 3. 🛠️ 단계별 실습 - 클라이언트

### STEP 8. Server-B (192.168.10.200)

```bash
dnf install -y nfs-utils net-tools
ifconfig | grep 192                        # IP 확인

mkdir /NFS_SBC
mount -t nfs 192.168.10.100:/NFS_SB /NFS_SBC
df -h | grep NFS_SB                        # 4G 확인
mount | grep nfs                           # 옵션 확인
```

fstab 등록:

```bash
cat <<EOF >> /etc/fstab
192.168.10.100:/NFS_SB  /NFS_SBC  nfs  defaults,_netdev,nofail  0 0
EOF

findmnt --verify --verbose
systemctl daemon-reload
umount /NFS_SBC && mount -a
findmnt /NFS_SBC
```

---

### STEP 9. Client-L (192.168.10.130)

```bash
sudo dnf install -y nfs-utils
sudo mkdir /NFS_CLIENT
sudo mount -t nfs 192.168.10.100:/NFS_LC /NFS_CLIENT
df -h | grep NFS_LC                        # 2G 확인
```

읽기·쓰기 테스트:

```bash
sudo cp -r /etc/a* /NFS_CLIENT/
ls -l /NFS_CLIENT/
```

fstab 등록:

```bash
sudo cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"

cat <<EOF | sudo tee -a /etc/fstab
192.168.10.100:/NFS_LC  /NFS_CLIENT  nfs  defaults,_netdev,nofail  0 0
EOF

findmnt --verify --verbose
sudo systemctl daemon-reload
sudo umount /NFS_CLIENT && sudo mount -a
findmnt /NFS_CLIENT
```

---

## 4. 🔍 검증 (Verification & Troubleshooting)

### 4-1. 재부팅 검증

```bash
sudo reboot                                # 클라이언트 재부팅
```

부팅 후:

```bash
mount | grep NFS                           # 자동 마운트 확인
df -h | grep NFS                           # 용량 확인
findmnt /NFS_CLIENT                        # 상태 확인
```

fstab 등록 전에는 재부팅 시 마운트가 사라지고, 등록 후에는 자동으로 유지되는 것을 비교 확인한다.

### 4-2. 접근 격리 확인

Server-B가 Client-L 전용 공유에 접근을 시도하면 거부되어야 한다.

```bash
mount -t nfs 192.168.10.100:/NFS_LC /mnt   # Server-B에서 실행 → 실패 정상
```

```text
mount.nfs: access denied by server while mounting 192.168.10.100:/NFS_LC
```

exports에서 클라이언트 IP를 지정했기 때문에 다른 호스트는 접근할 수 없다.

### 4-3. 서버 측 접속 현황

```bash
showmount -a                               # 마운트 중인 클라이언트
ss -tnp | grep 2049                        # 연결 세션
nfsstat -s                                 # 서버 통계
```

---

## 5. ✅ 최종 체크리스트

```text
[ ] 추가 디스크 파티션 구성 확인 (lsblk)
[ ] sdb1·sdb2 ext4 포맷
[ ] nfs-utils·rpcbind 설치 확인
[ ] 방화벽 nfs·rpc-bind·mountd permanent 허용 + reload
[ ] /NFS_SB, /NFS_LC 생성 및 파티션 마운트
[ ] 서버 fstab에 UUID 등록
[ ] /etc/exports 작성 후 exportfs -ra
[ ] showmount -e 로 공개 확인
[ ] Server-B → /NFS_SBC 마운트 성공
[ ] Client-L → /NFS_CLIENT 마운트 성공
[ ] 양쪽 fstab 등록 후 재부팅 유지 확인
[ ] 타 클라이언트 접근 차단 확인
```

> 📌 **핵심 요약**
> - 클라이언트별로 파티션·공유 디렉터리를 분리하면 용량 격리가 쉽다
> - exports에서 IP를 지정하면 다른 호스트 접근이 자동 차단된다
> - 서버는 파티션 fstab, 클라이언트는 NFS fstab을 각각 등록해야 한다
> - 변경 후에는 `exportfs -ra` → `showmount -e` 순으로 검증
> - 재부팅 검증까지 마쳐야 실습이 완료된다
> - 관련: ⚙️ NFS 서버 구성 · 💻 NFS 클라이언트 마운트 & fstab · 📚 종합정리 Samba & NFS
