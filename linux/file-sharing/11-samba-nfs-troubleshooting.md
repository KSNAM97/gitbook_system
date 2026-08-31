# Samba·NFS 트러블슈팅 치트시트

> **Tag:** #Linux #Troubleshooting #Samba #NFS #CIFS #Error #Checklist  
> **핵심 요약:** Samba·NFS 장애는 대부분 네트워크, 방화벽, 서비스 상태, 설정 파일 문법, 디렉터리 퍼미션, UID/계정 매핑, SELinux 중 하나에서 발생한다. 이 문서는 증상별 원인과 조치를 표로 정리하고, 어떤 순서로 좁혀 나가야 하는지 진단 플로우를 제공한다. 오류 메시지를 그대로 검색할 수 있도록 실제 출력 문구를 함께 수록했다.

---

## 1. 공통 진단 플로우

### 1-1. 7계층 점검 순서

```text
1) 네트워크      ping / ip addr
2) 방화벽        firewall-cmd --list-all (서버·클라이언트 모두)
3) 서비스        systemctl status smb nmb / nfs-server rpcbind
4) 설정 문법     testparm / exportfs -v
5) 공유 공개     smbclient -L IP / showmount -e IP
6) 퍼미션        ls -ld 공유디렉터리, 소유자·그룹·모드
7) 인증/매핑     pdbedit -L / id 사용자 / UID 일치
8) SELinux       getenforce, ls -Zd, ausearch -m avc -ts recent
```

### 1-2. 만능 점검 스크립트

```bash
echo "== 서비스 ==" ; systemctl is-active smb nmb nfs-server rpcbind
echo "== 방화벽 ==" ; firewall-cmd --list-all
echo "== 포트 ==" ; ss -tulnp | egrep '445|139|2049|111'
echo "== Samba ==" ; testparm -s 2>/dev/null | head -20
echo "== NFS ==" ; exportfs -v
echo "== SELinux ==" ; getenforce
```

---

## 2. Samba 오류 대응표

### 2-1. 접속·인증 오류

| 오류 메시지 | 원인 | 조치 |
|---|---|---|
| `NT_STATUS_LOGON_FAILURE` | Samba DB에 계정 없음, 비밀번호 불일치 | `smbpasswd -a 계정`, `pdbedit -L` 확인 |
| `NT_STATUS_ACCESS_DENIED` | `valid users` 불일치, 퍼미션 부족 | `id 계정`으로 그룹 확인, `ls -ld` 점검 |
| `NT_STATUS_BAD_NETWORK_NAME` | 공유 이름 오타, 섹션 미정의 | `smbclient -L`로 실제 공유명 확인 |
| `NT_STATUS_CONNECTION_REFUSED` | smbd 미실행 | `systemctl status smb` → `restart` |
| `NT_STATUS_IO_TIMEOUT` | 방화벽 445 차단 | `firewall-cmd --permanent --add-service=samba` |
| `Connection to X failed (Error NT_STATUS_UNSUCCESSFUL)` | 프로토콜 협상 실패 | `-m SMB3` 또는 `vers=3.0` 지정 |

### 2-2. 마운트 오류 (Linux → Windows/Samba)

| 오류 메시지 | 원인 | 조치 |
|---|---|---|
| `mount: unknown filesystem type 'cifs'` | `cifs-utils` 미설치 | `dnf install -y cifs-utils` |
| `mount error(13): Permission denied` | 계정·비밀번호 오류, 공유 권한 부족 | 계정 재확인, Windows 공유 권한 점검 |
| `mount error(112): Host is down` | SMB 버전 협상 실패 | `-o vers=3.0` 또는 `vers=2.1` |
| `mount error(2): No such file or directory` | 공유명·경로 오타 | `//IP/공유명` 형식 재확인 |
| `mount error(115): Operation now in progress` | 네트워크 도달 불가, 방화벽 | `ping`, 445 포트 확인 |
| `mount error(22): Invalid argument` | 옵션 오타 | `-o` 옵션 문자열 재검토 |

### 2-3. 동작 이상

| 증상 | 원인 | 조치 |
|---|---|---|
| 공유는 보이는데 쓰기 불가 | `writable = no`, 디렉터리 권한, SELinux | `testparm`, `ls -ld`, `restorecon -Rv` |
| 탐색기 목록에 안 보임 | `browseable = no`, nmbd 미실행 | 설정 수정 + `systemctl start nmb` |
| 설정을 바꿔도 그대로 | 데몬 미재시작 | `systemctl restart smb nmb` |
| Windows가 예전 계정으로 붙음 | 자격 증명 캐시 | `net use * /delete` 후 재연결 |
| 새 파일 권한이 이상함 | mask 설정 | `create mask`, `directory mask` 조정 |
| 파일 그룹이 제각각 | `force group` 누락 | `force group = SG` 추가 |
| 남의 파일이 삭제됨 | Sticky Bit 없음 | `chmod 1777 /SHARE` |
| 한글 파일명 깨짐 | 문자셋 | `iocharset=utf8` 옵션 |

### 2-4. Samba 진단 명령

```bash
testparm                                   # 설정 문법
smbclient -L localhost -U 계정              # 로컬 인증 테스트
smbstatus                                  # 세션·잠금
pdbedit -L                                 # Samba 계정 목록
journalctl -u smb -n 100                   # 상세 로그
ls -Zd /SHARE                              # SELinux 컨텍스트
ausearch -m avc -ts recent                 # 접근 거부 로그
```

---

## 3. NFS 오류 대응표

### 3-1. 서버 측 오류

| 오류 메시지 | 원인 | 조치 |
|---|---|---|
| `exportfs: Failed to stat /NFSS: No such file or directory` | 공유 디렉터리 미생성 | `mkdir /NFSS` → `exportfs -ra` |
| `exportfs: /NFSS does not support NFS export` | 경로 오류·미지원 FS | 경로·파일시스템 확인 |
| `exportfs: No options for /NFSS ...: suggest ... to avoid warning` | 옵션 미지정 | 클라이언트 뒤 `(rw,sync)` 명시 |
| `exportfs: /etc/exports:1: syntax error` | 문법 오류 | 절대 경로·괄호 공백 확인 |
| `showmount`가 아무것도 출력 안 함 | `exportfs -ra` 미실행 | 반영 명령 실행 |
| 서비스가 `active (exited)` | 정상 | 커널 스레드 방식이므로 문제 아님 |

### 3-2. 클라이언트 마운트 오류

| 오류 메시지 | 원인 | 조치 |
|---|---|---|
| `mount point /NFSC does not exist` | 마운트포인트 없음 | `mkdir /NFSC` |
| `access denied by server while mounting` | exports에 클라이언트 IP 없음 | 서버 `exportfs -v`로 허용 IP 확인 |
| `Connection timed out` | 방화벽 2049 차단, 서버 다운 | `firewall-cmd --list-all`, `ping` |
| `clnt_create: RPC: Program not registered` | rpcbind·nfs-server 미실행 | `systemctl enable --now rpcbind nfs-server` |
| `requested NFS version or transport protocol is not supported` | 버전 불일치 | `-o vers=3` 또는 `vers=4.2` 지정 |
| `Stale file handle` | 서버에서 export 재생성됨 | `umount -l` 후 재마운트 |
| `Permission denied`(쓰기) | `ro` 옵션, 퍼미션, UID 불일치 | `rw` 확인, `chmod`, UID 통일 |
| `device is busy`(umount) | 사용 중 프로세스 | `fuser -mv /NFSC` → 종료 후 `umount -l` |

### 3-3. 권한·소유자 문제

| 증상 | 원인 | 조치 |
|---|---|---|
| 파일 소유자가 `nobody`/`4294967294` | idmap 불일치, squash 옵션 | UID 통일 또는 `all_squash,anonuid=` 설정 |
| root로 만들었는데 권한 없음 | `root_squash`(기본) | 필요 시 `no_root_squash`(보안 주의) |
| 같은 이름 계정인데 접근 거부 | 서버·클라이언트 UID 다름 | `id 계정`으로 UID 비교 후 통일 |
| 쓰기는 되는데 삭제가 됨 | Sticky Bit 없음 | `chmod 1777` |

### 3-4. NFS 진단 명령

```bash
showmount -e 192.168.10.100                # export 공개 확인
exportfs -v                                # 서버 적용 상태
rpcinfo -p 192.168.10.100                  # RPC 등록 서비스
mount -v -t nfs 192.168.10.100:/NFSS /NFSC # 상세 로그로 마운트
nfsstat -m                                 # 마운트 상세
dmesg | tail -30                           # 커널 오류
journalctl -u nfs-server -n 100            # 서버 로그
```

---

## 4. 방화벽·fstab 관련 함정

### 4-1. 방화벽

```text
증상: 허용했는데 --reload 후 목록에서 사라짐
원인: --permanent 누락 (런타임 규칙은 reload 시 초기화)
조치: firewall-cmd --permanent --add-service=nfs ; firewall-cmd --reload

증상: --permanent로 추가했는데 즉시 적용 안 됨
원인: reload 미실행
조치: firewall-cmd --reload
```

### 4-2. fstab

| 증상 | 원인 | 조치 |
|---|---|---|
| 재부팅 후 마운트 사라짐 | fstab 미등록 | 항목 추가 후 `mount -a` |
| 부팅이 멈추거나 지연 | 네트워크 준비 전 마운트 시도 | `_netdev`, `nofail` 추가 |
| emergency mode 진입 | fstab 항목 오타·장치 없음 | 콘솔에서 fstab 수정 후 재부팅 |
| `mount -a`가 조용히 실패 | 문법 오류 | `findmnt --verify --verbose` |

복구 요령:

```bash
cp -a /etc/fstab "/etc/fstab.bak.$(date +%F-%H%M%S)"   # 수정 전 항상 백업
findmnt --verify --verbose                             # 저장 직후 검증
systemctl daemon-reload
mount -a
```

> fstab을 수정한 뒤 재부팅하기 전에 반드시 `mount -a`와 `findmnt --verify`로 검증한다. 검증 없이 재부팅하면 emergency mode에 빠질 수 있다.

---

## 5. SELinux 관련

```bash
getenforce                                 # Enforcing/Permissive/Disabled
ausearch -m avc -ts recent                 # 최근 거부 로그
sealert -a /var/log/audit/audit.log        # 해석(setroubleshoot 설치 시)
```

| 증상 | 조치 |
|---|---|
| Samba 공유 접근 거부(설정은 정상) | `semanage fcontext -a -t samba_share_t "/SHARE(/.*)?"` → `restorecon -Rv /SHARE` |
| 홈 디렉터리 공유 실패 | `setsebool -P samba_enable_home_dirs on` |
| NFS 쓰기 거부 | `setsebool -P nfs_export_all_rw on` |
| 원인 파악용 임시 확인 | `setenforce 0`으로 테스트 후 반드시 `setenforce 1` 복귀 |

> `setenforce 0`은 원인 확인용 임시 조치일 뿐이다. SELinux를 끈 채 운영하지 말고, 원인을 확인한 뒤 올바른 컨텍스트·부울로 해결한다.

---

## 6. 증상별 30초 진단

```text
[접속 자체가 안 됨]
ping → 방화벽 → 서비스 상태 → 포트 리스닝

[목록은 보이는데 마운트 실패]
Samba: 계정/비밀번호 → valid users
NFS:   exports IP → exportfs -v

[마운트는 되는데 쓰기 실패]
설정 옵션(rw/writable) → 디렉터리 퍼미션 → UID/그룹 → SELinux

[재부팅하면 사라짐]
fstab 등록 여부 → _netdev/nofail → findmnt --verify

[설정을 바꿨는데 반영 안 됨]
Samba: systemctl restart smb nmb
NFS:   exportfs -ra
방화벽: firewall-cmd --reload
```

---

## 7. 장애 대응 최종 체크리스트

```text
[ ] 서버·클라이언트 IP 확인 및 ping 성공
[ ] 서버 방화벽에 해당 서비스가 --permanent로 등록됨
[ ] firewall-cmd --reload 실행함
[ ] 관련 서비스가 enabled + active 상태
[ ] 설정 파일 문법 검증 통과(testparm / exportfs -v)
[ ] 공유가 외부에 공개됨(smbclient -L / showmount -e)
[ ] 공유 디렉터리 퍼미션·소유권 적절
[ ] 계정 인증(Samba) 또는 UID 매핑(NFS) 정합
[ ] SELinux 컨텍스트·부울 설정 확인
[ ] fstab 등록 후 findmnt --verify 통과
[ ] 재부팅 후 정상 동작 확인
```

>  **핵심 요약**
> - 진단은 네트워크 → 방화벽 → 서비스 → 설정 → 퍼미션 → 인증 → SELinux 순
> - Samba 인증 오류는 `smbpasswd -a`와 `valid users`부터 확인
> - NFS 접근 거부는 exports의 클라이언트 IP가 1순위 원인
> - `--permanent` 누락과 `exportfs -ra` 누락이 가장 흔한 실수
> - fstab 수정 후에는 반드시 검증 후 재부팅
> - 관련:  종합정리 Samba & NFS ·  Samba·NFS 퀵 레퍼런스 ·  Linux Samba 서버 구축 ·  NFS 서버 구성
