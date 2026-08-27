# 📚 종합정리 Samba & NFS

> **Tag:** #Linux #Summary #Samba #NFS #SMB #CIFS #FileSharing #Review  
> **핵심 요약:** Samba와 NFS는 모두 네트워크로 서버의 디스크 공간을 공유하는 기술이지만, Samba는 SMB 프로토콜 기반의 계정 인증 방식이고 NFS는 RPC 기반의 IP·UID 신뢰 방식이다. Samba는 `smbd`·`nmbd` 데몬과 `smb.conf`, `smbpasswd`로 구성하고, NFS는 `nfs-server`·`rpcbind`와 `/etc/exports`, `exportfs`로 구성한다. 두 기술 모두 방화벽 `--permanent` 설정과 fstab 영구 마운트, 디렉터리 퍼미션 정합성이 성공의 핵심이다.

---

## 1. 🎯 전체 구조 한눈에 보기

### 1-1. 기술 계층 비교

```text
[Samba/SMB]
Windows/Linux Client
   ↓ SMB2/3 (TCP 445)
smbd (인증·파일 I/O)  +  nmbd (NetBIOS 이름, UDP 137·138)
   ↓
/etc/samba/smb.conf → 공유 섹션([Share])
   ↓
/SHARE (Linux 퍼미션 + SELinux samba_share_t)

[NFS]
Linux Client
   ↓ NFSv4 (TCP 2049) / NFSv3 (+ rpcbind 111)
nfsd (커널 스레드)  +  rpcbind  +  mountd
   ↓
/etc/exports → exportfs -ra
   ↓
/NFS_LC (Linux 퍼미션 + UID/GID 매핑)
```

---

### 1-2. 핵심 대조표

| 항목 | Samba(SMB) | NFS |
|---|---|---|
| 프로토콜 | SMB2/SMB3 | NFSv3 / NFSv4 |
| 주요 데몬 | `smbd`, `nmbd` | `nfsd`(커널), `rpcbind`, `mountd` |
| 서비스 유닛 | `smb`, `nmb` | `nfs-server`, `rpcbind` |
| 설정 파일 | `/etc/samba/smb.conf` | `/etc/exports` |
| 반영 명령 | `systemctl restart smb` | `exportfs -ra` |
| 문법 검증 | `testparm` | `exportfs -v` |
| 공유 확인 | `smbclient -L IP` | `showmount -e IP` |
| 계정 관리 | `smbpasswd -a`, `pdbedit -L` | 별도 없음(UID 기반) |
| 마운트 | `mount -t cifs` | `mount -t nfs` |
| 필수 패키지 | `samba`, `samba-client`, `cifs-utils` | `nfs-utils` |
| 방화벽 서비스 | `samba` | `nfs`, `rpc-bind`, `mountd` |
| 포트 | TCP 445·139 / UDP 137·138 | TCP 2049 (+111) |
| 권한 결정 | 계정 인증 + create/directory mask | Linux 퍼미션 + UID/GID |
| SELinux | `samba_share_t` | `nfs_export_all_rw` |

---

## 2. 🛠️ 구축 절차 요약

### 2-1. Samba 서버 구축 8단계

```text
1) dnf install -y samba samba-client samba-common cifs-utils
2) mkdir /SHARE ; groupadd SG ; useradd -G SG samba
3) chmod 1777 /SHARE ; chown root:SG /SHARE
4) passwd samba ; smbpasswd -a samba
5) vi /etc/samba/smb.conf → [Share] 섹션 작성
6) testparm 으로 문법 검증
7) firewall-cmd --permanent --add-service=samba ; --reload
8) semanage fcontext + restorecon → systemctl enable --now smb nmb
```

### 2-2. NFS 서버 구축 7단계

```text
1) dnf install -y nfs-utils
2) mkdir /NFS_LC ; chmod 1777 /NFS_LC
3) vi /etc/exports → "/NFS_LC 192.168.10.130(rw,sync,no_subtree_check)"
4) exportfs -ra ; exportfs -v
5) firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd} ; --reload
6) setsebool -P nfs_export_all_rw on
7) systemctl enable --now rpcbind nfs-server → showmount -e
```

### 2-3. 클라이언트 마운트 요약

```bash
# Samba(CIFS) 클라이언트
mkdir /smbClient
mount -t cifs //192.168.10.131/winShare /smbClient -o credentials=/root/.smbcred,iocharset=utf8

# NFS 클라이언트
mkdir /NFSC
mount -t nfs 192.168.10.100:/NFSS /NFSC
```

fstab 영구화:

```text
//192.168.10.131/winShare  /smbClient  cifs  credentials=/root/.smbcred,iocharset=utf8,_netdev  0 0
192.168.10.100:/NFSS       /NFSC       nfs   defaults,_netdev,nofail                           0 0
```

---

## 3. 🔍 개념 핵심 정리

### 3-1. 자주 헷갈리는 포인트

| 오해 | 정확한 내용 |
|---|---|
| "메모리를 공유한다" | 디스크 공간(스토리지)을 공유한다 |
| "Samba는 Linux↔Windows 전용" | Linux↔Linux, Windows↔Windows도 가능 |
| "Windows에 Samba 서버를 설치" | Windows는 공유 폴더 자체가 SMB 서버 |
| "exports에 rw면 무조건 쓰기 가능" | 디렉터리 Linux 퍼미션도 맞아야 함 |
| "방화벽 허용하면 끝" | `--permanent` + `--reload` 둘 다 필요 |
| "mount 하면 계속 유지" | 재부팅 시 해제, fstab 등록 필요 |
| "CIFS = 구식이라 못 씀" | 마운트 타입 이름일 뿐, 실제는 SMB3 협상 |
| "nfs-server가 exited면 실패" | 커널 스레드 방식이라 정상 상태 |

### 3-2. 권한 모델 차이

```text
[Samba]
Windows 계정 인증 → Linux 계정 매핑
→ valid users / force group 으로 접근 제어
→ create mask / directory mask 로 생성 권한 결정

[NFS]
클라이언트가 보낸 UID/GID를 그대로 신뢰(sec=sys)
→ root_squash 로 root 권한 축소
→ 서버·클라이언트 UID 일치가 중요
```

### 3-3. 삭제 권한과 Sticky Bit

```text
파일 삭제 권한 = 상위 디렉터리의 쓰기 권한
777 공유 → 누구나 남의 파일 삭제 가능
1777 (Sticky Bit) → 소유자·디렉터리 소유자·root만 삭제 가능
읽기·복사는 그대로 허용
```

---

## 4. ✅ 통합 체크리스트

```text
[공통]
[ ] 서버·클라이언트 IP와 통신(ping) 확인
[ ] 필요한 패키지 설치 확인
[ ] 방화벽 --permanent 허용 후 --reload
[ ] 공유 디렉터리 생성 및 퍼미션 설정
[ ] SELinux 컨텍스트·부울 확인
[ ] 서비스 enable --now 로 자동 시작
[ ] fstab 등록 후 findmnt --verify
[ ] 재부팅 후 마운트 유지 검증

[Samba 전용]
[ ] Linux 계정 + smbpasswd -a 등록
[ ] smb.conf 작성 후 testparm 통과
[ ] smbclient -L 로 공유 목록 확인
[ ] Windows 네트워크 드라이브 연결 확인
[ ] Sticky Bit로 삭제 권한 제어

[NFS 전용]
[ ] /etc/exports 절대 경로·괄호 공백 확인
[ ] exportfs -ra 실행 및 -v 검증
[ ] showmount -e 로 공개 확인
[ ] 허용되지 않은 IP 접근 차단 확인
[ ] UID/GID 매핑 정합성 확인
```

> 📌 **핵심 요약**
> - Samba는 계정 인증 기반, NFS는 IP·UID 신뢰 기반
> - 설정 반영은 Samba `restart smb`, NFS `exportfs -ra`
> - 확인은 Samba `smbclient -L`, NFS `showmount -e`
> - 방화벽 `--permanent`와 fstab 등록이 실습 성패를 가른다
> - 권한 문제는 퍼미션 → 설정 옵션 → UID/계정 → SELinux 순으로 점검
> - 관련: ⚡ Samba·NFS 퀵 레퍼런스 · 🚨 Samba·NFS 트러블슈팅 치트시트
