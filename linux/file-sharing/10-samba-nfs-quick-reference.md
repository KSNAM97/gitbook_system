# ⚡ Samba·NFS 퀵 레퍼런스

> **Tag:** #Linux #QuickReference #Samba #NFS #Cheatsheet #Command  
> **핵심 요약:** Samba와 NFS 구축·점검에 필요한 명령을 한 페이지로 모은 참조 문서이다. 설치, 설정 파일, 서비스 제어, 방화벽, 마운트, 확인 명령을 목적별로 정리했으며 각 명령의 핵심 옵션과 대표 출력 형태를 함께 담았다. 실습 중 명령이 기억나지 않을 때 이 문서만 열어 바로 찾을 수 있도록 구성했다.

---

## 1. 📦 설치

```bash
dnf install -y samba samba-client samba-common samba-winbind   # Samba 서버·클라이언트
dnf install -y cifs-utils                                      # mount -t cifs 필수
dnf install -y nfs-utils                                       # NFS 서버·클라이언트
dnf install -y policycoreutils-python-utils                    # semanage 명령
dnf install -y net-tools                                       # ifconfig 등
```

설치 확인:

```bash
rpm -qa | grep samba
rpm -qa | grep nfs
rpm -qa | grep rpcbind
rpm -qa | grep cifs-utils
```

---

## 2. 🐧 Samba 명령

### 2-1. 계정 관리

```bash
useradd -G SG samba                        # Linux 계정 생성(보조 그룹)
passwd samba                               # Linux 비밀번호
smbpasswd -a samba                         # Samba DB 등록 + 비밀번호
smbpasswd -x samba                         # Samba DB에서 삭제
smbpasswd -d samba                         # 계정 비활성화
smbpasswd -e samba                         # 계정 활성화
pdbedit -L                                 # Samba 사용자 목록
pdbedit -Lv samba                          # 상세 정보
```

### 2-2. 설정·서비스

```bash
vi /etc/samba/smb.conf                     # 메인 설정 파일
testparm                                   # 문법 검증
testparm -s                                # 요약 출력

systemctl enable --now smb                 # SMB 데몬
systemctl enable --now nmb                 # NMB 데몬
systemctl restart smb nmb                  # 설정 반영
systemctl status smb
journalctl -u smb -n 50                    # 로그
```

### 2-3. 공유 섹션 템플릿

```text
[Share]
        path = /SHARE
        writable = yes
        browseable = yes
        guest ok = no
        valid users = @SG
        force group = SG
        create mask = 0666
        directory mask = 0777
```

### 2-4. 클라이언트·확인

```bash
smbclient -L 192.168.10.131                # 공유 목록
smbclient -L 192.168.10.131 -U root        # 계정 지정
smbclient //192.168.10.131/winShare -U root  # 대화형 접속(ls/get/put)
smbstatus                                  # 접속 세션·잠금
ss -tulnp | egrep '445|139|137|138'        # 포트 확인
```

### 2-5. CIFS 마운트

```bash
mount -t cifs //192.168.10.131/winShare /smbClient \
  -o username=root,password=1234,iocharset=utf8

mount -t cifs //192.168.10.131/winShare /smbClient \
  -o credentials=/root/.smbcred,iocharset=utf8,vers=3.0

mount | grep cifs                          # 협상 버전 확인
umount /smbClient
```

credentials 파일:

```bash
printf 'username=root\npassword=1234\n' > /root/.smbcred
chmod 600 /root/.smbcred
```

### 2-6. Windows 측 명령

```text
net user root 1234 /add                    # 계정 생성(관리자 권한)
net user                                   # 계정 목록
net share                                  # 공유 목록
net use Z: \\192.168.10.100\SHARE /user:samba 1234   # 드라이브 연결
net use Z: /delete                         # 연결 해제
net use * /delete                          # 자격 증명 캐시 초기화
ipconfig                                   # IP 확인
```

---

## 3. 📡 NFS 명령

### 3-1. exports 설정

```bash
vi /etc/exports
```

```text
/NFS_LC  192.168.10.130(rw,no_root_squash,sync)
/NFS_SB  192.168.10.200(rw,sync,no_subtree_check)
/backup  192.168.10.0/24(ro,sync)
/pub     *(ro,sync,all_squash)
```

### 3-2. exportfs·서비스

```bash
exportfs -ra                               # 재읽기 + 적용(표준)
exportfs -a                                # 전체 export
exportfs -v                                # 현재 상태 확인
exportfs -u IP:/경로                       # 특정 해제
exportfs -ua                               # 전체 해제

systemctl enable --now rpcbind
systemctl enable --now nfs-server
systemctl restart nfs-server
systemctl status nfs-server
journalctl -u nfs-server -n 50
```

### 3-3. 확인 명령

```bash
showmount -e                               # 로컬 export 목록
showmount -e 192.168.10.100                # 원격 export 목록
showmount -a                               # 접속 중인 클라이언트
rpcinfo -p                                 # RPC 서비스·포트
rpcinfo -p 192.168.10.100                  # 원격 RPC 조회
nfsstat -s                                 # 서버 통계
nfsstat -c                                 # 클라이언트 통계
nfsstat -m                                 # 마운트 상세
ss -tulnp | grep 2049                      # 포트 확인
```

### 3-4. NFS 마운트

```bash
mkdir /NFSC
mount -t nfs 192.168.10.100:/NFSS /NFSC
mount -t nfs -o rw,hard,timeo=600,vers=4.2 192.168.10.100:/NFSS /NFSC
mount -t nfs -o ro 192.168.10.100:/NFSS /NFSC

mount | grep nfs
df -h | grep NFS
findmnt /NFSC
umount /NFSC
umount -l /NFSC                            # lazy 해제
fuser -mv /NFSC                            # 사용 중 프로세스
```

---

## 4. 🔥 방화벽 (firewalld)

```bash
# Samba
firewall-cmd --permanent --add-service=samba
firewall-cmd --permanent --add-port=445/tcp
firewall-cmd --permanent --add-port=139/tcp
firewall-cmd --permanent --add-port=137/udp
firewall-cmd --permanent --add-port=138/udp

# NFS
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --permanent --add-port=2049/tcp
firewall-cmd --permanent --add-port=111/tcp
firewall-cmd --permanent --add-port=111/udp

firewall-cmd --reload                      # 적용(필수)
```

확인·삭제:

```bash
firewall-cmd --list-all                    # 전체 정책
firewall-cmd --list-port                   # 포트 목록
firewall-cmd --list-services               # 서비스 목록
firewall-cmd --permanent --remove-port=1049/tcp
firewall-cmd --permanent --remove-service=samba
firewall-cmd --reload
```

```text
--permanent 없이 추가 → reload/재부팅 시 소멸
--permanent 로 추가   → reload 해야 런타임 반영
```

---

## 5. 🔐 SELinux

```bash
getenforce                                 # 현재 모드
sestatus                                   # 상세 상태

# Samba
semanage fcontext -a -t samba_share_t "/SHARE(/.*)?"
restorecon -Rv /SHARE
ls -Zd /SHARE
setsebool -P samba_export_all_rw on

# NFS
setsebool -P nfs_export_all_rw on
setsebool -P nfs_export_all_ro on
getsebool -a | grep nfs

ausearch -m avc -ts recent                 # 거부 로그 확인
```

---

## 6. 💾 fstab 항목 모음

```text
# CIFS (Samba)
//192.168.10.131/winShare  /smbClient  cifs  username=root,password=1234,iocharset=utf8,_netdev  0 0
//192.168.10.131/winShare  /smbClient  cifs  credentials=/root/.smbcred,iocharset=utf8,_netdev   0 0

# NFS
192.168.10.100:/NFSS    /NFSC        nfs  defaults,_netdev          0 0
192.168.10.100:/NFS_LC  /NFS_CLIENT  nfs  defaults,_netdev,nofail   0 0

# 로컬 파티션(NFS 서버 측 공유 지점)
UUID=<uuid>  /NFS_SB  ext4  defaults  0 0
```

검증 절차:

```bash
cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"
findmnt --verify --verbose
systemctl daemon-reload
mount -a
findmnt /NFSC
```

---

## 7. 🔢 포트 요약

| 서비스 | 포트 | 용도 |
|---|---|---|
| SMB (Direct) | TCP 445 | 최신 Windows 기본 파일 공유 |
| SMB (NetBIOS) | TCP 139 | 구버전 호환 |
| NetBIOS Name | UDP 137 | 이름 서비스(nmbd) |
| NetBIOS Datagram | UDP 138 | 데이터그램 서비스(nmbd) |
| NFS | TCP 2049 | nfsd 데이터 통신 |
| rpcbind | TCP/UDP 111 | RPC 포트 매퍼 |
| mountd | 동적(고정 가능) | NFSv3 마운트 요청 |

---

## 8. 🔤 권한·옵션 요약

```text
[Samba smb.conf]
writable=yes / browseable=yes / guest ok=no
valid users=@그룹 / force group=그룹
create mask=0666 / directory mask=0777

[NFS exports]
ro | rw
sync | async
root_squash | no_root_squash | all_squash
no_subtree_check
anonuid= / anongid=

[퍼미션]
chmod 777  → 누구나 삭제 가능
chmod 1777 → 소유자만 삭제(Sticky Bit, drwxrwxrwt)
chmod 2770 → Set-GID, 그룹 소유권 상속
```

> 📌 **핵심 요약**
> - Samba 반영은 `testparm` → `systemctl restart smb nmb`
> - NFS 반영은 `exportfs -ra` → `exportfs -v` → `showmount -e`
> - 방화벽은 항상 `--permanent` + `--reload`
> - CIFS 마운트에는 `cifs-utils`, 한글에는 `iocharset=utf8`
> - fstab 수정 후 `findmnt --verify`로 검증
> - 관련: 📚 종합정리 Samba & NFS · 🚨 Samba·NFS 트러블슈팅 치트시트
