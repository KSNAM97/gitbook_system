# 종합정리 네트워크 서비스 (SSH·SCP·SFTP·vsFTP·DHCP·DNS)

> **Tag:** #Linux #Summary #SSH #SCP #SFTP #vsFTP #DHCP #DNS #Ports
> **핵심 요약:** 11-1~11-8 문서에서 다룬 원격 접속(SSH), 파일 전송(SCP·SFTP·vsFTP), 주소 자동 할당(DHCP), 이름 해석(DNS)을 하나의 지도로 묶는다. SSH는 원격 접속과 SCP·SFTP의 기반이 되고, vsFTP는 평문 전송의 대안으로 SFTP·SCP가 쓰인다. DHCP는 IP를 자동 할당하고 DNS는 이름을 IP로 해석하며, 모든 서비스는 설치 → 설정 → 방화벽 개방 → 데몬 기동·재시작 → 검증이라는 동일한 흐름을 따른다.

---

## 1. 서비스 전체 지도

### 1-1. 포트 요약

| 서비스 | 프로토콜·포트 | 데몬 | 설정 파일 |
|---|---|---|---|
| SSH·SFTP·SCP | TCP 22 | sshd | `/etc/ssh/sshd_config` |
| Telnet | TCP 23 | telnet.socket | 사용 지양 |
| FTP(제어) | TCP 21 | vsftpd | `/etc/vsftpd/vsftpd.conf` |
| FTP(데이터) | TCP 20 | vsftpd | 동일 |
| HTTP | TCP 80 | httpd | `/etc/httpd/conf/httpd.conf` |
| DNS | TCP·UDP 53 | named | `/etc/named.conf` |
| DHCP(서버) | UDP 67 | dhcpd | `/etc/dhcp/dhcpd.conf` |
| DHCP(클라이언트) | UDP 68 | NetworkManager | nmconnection |

---

### 1-2. 파일 전송 방식 비교

| 구분 | FTP(vsftpd) | SFTP | SCP |
|---|---|---|---|
| 기반 | 자체 프로토콜 | SSH | SSH |
| 포트 | 20, 21 | 22 | 22 |
| 암호화 | 없음 | 있음 | 있음 |
| 별도 데몬 | 필요(vsftpd) | 불필요 | 불필요 |
| 사용 형태 | 대화형 | 대화형 | 단발 명령 |
| 스크립트 적합성 | 보통 | 보통 | 높음 |

```text
보안이 필요한 전송     → SFTP 또는 SCP
자동화 스크립트         → SCP 또는 rsync
익명 공개 배포          → vsFTP(익명, 읽기 전용)
대용량·증분·재개        → rsync
```

---

### 1-3. 공통 구축 흐름

모든 서비스는 다음 5단계를 따른다.

```text
1) 패키지 설치        dnf install -y 패키지
2) 설정 파일 편집     vi /etc/서비스/설정파일
3) 방화벽 개방        firewall-cmd --permanent --add-service=...
4) 데몬 기동·재시작   systemctl start|restart|enable 서비스
5) 검증               상태·포트·로그·클라이언트 테스트
```

> 5단계 중 하나라도 빠지면 서비스가 동작하지 않는다. 특히 방화벽 개방과 데몬 재시작이 가장 자주 누락된다.

---

## 2. 서비스별 핵심 정리

### 2-1. SSH (11-1)

```bash
vi /etc/ssh/sshd_config
sshd -t                                    # 재시작 전 문법 검사(필수)
systemctl restart sshd
```

핵심 옵션: `PermitRootLogin no`(root 직접 로그인 차단), `Port`(포트 변경 시 방화벽 동시 처리), `Banner`(접속 경고문), `MaxAuthTries`·`LoginGraceTime`(인증 시도·대기 제한), `AllowUsers`·`AllowGroups`(화이트리스트, 자기 계정 누락 주의).

---

### 2-2. SCP·SFTP (11-2, 11-4)

SSH만 동작하면 추가 설치 없이 사용할 수 있다.

```bash
scp 로컬파일 계정@호스트:/원격경로          # 업로드
scp 계정@호스트:/원격파일 로컬경로          # 다운로드
scp -r ...                                 # 디렉터리
sftp 계정@호스트                           # 대화형 접속
```

업로드는 로컬(SCP) 또는 SSH 로그인 시점(SFTP)의 현재 디렉터리 기준, 다운로드는 원격 경로를 직접 지정할 수 있다는 규칙을 기억한다.

---

### 2-3. vsFTP (11-3)

핵심 설정 항목은 익명 접속(`anonymous_enable`), 로컬 계정(`local_enable`), 쓰기 권한(`write_enable`), 전송 로그(`xferlog_enable`), 접근 제어(`userlist_enable`·`userlist_deny`), 계정별 설정(`user_config_dir`), 격리(`chroot_local_user`·`allow_writeable_chroot`)이다.

```bash
dnf install -y vsftpd
vi /etc/vsftpd/vsftpd.conf
firewall-cmd --permanent --add-service=ftp
firewall-cmd --permanent --add-port=20/tcp --add-port=21/tcp
firewall-cmd --reload
systemctl enable --now vsftpd
```

접근 제어 요약:

```text
userlist_deny=YES  → user_list는 차단 목록(기본)
userlist_deny=NO   → user_list는 허용 목록
chroot_local_user  → 홈 디렉터리를 / 로 인식시켜 격리
```

---

### 2-4. DHCP (11-5)

```bash
dnf install -y dhcp-server
vi /etc/dhcp/dhcpd.conf
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --permanent --add-port=67/udp --add-port=68/udp
firewall-cmd --reload
systemctl enable --now dhcpd
cat /var/lib/dhcpd/dhcpd.leases
```

핵심 개념은 DORA(Discover·Offer·Request·ACK), 임대 시간(default/max-lease-time), range 설계, 한 네트워크 내 DHCP 서버 중복 금지이다.

---

### 2-5. DNS·BIND (11-6, 11-7, 11-8)

```bash
dnf install -y bind bind-utils
vi /etc/named.conf                         # 또는 named.rfc1912.zones
vi /var/named/도메인.db                     # zone 파일
named-checkconf
named-checkzone 도메인 /var/named/도메인.db
chown root:named /var/named/도메인.db
chmod 640 /var/named/도메인.db
firewall-cmd --permanent --add-service=dns
firewall-cmd --permanent --add-port=53/tcp --add-port=53/udp
firewall-cmd --reload
systemctl enable --now named
```

zone 파일 필수 요소는 `$TTL`, SOA(serial·refresh·retry·expire·minimum), NS, A, PTR이며 도메인 값 끝의 점(`.`)을 반드시 확인한다.

```text
A     이름 → IPv4
AAAA  이름 → IPv6
NS    도메인의 네임서버
PTR   IP → 이름(역방향)
```

---

## 3. 통합 검증 루틴

### 3-1. 서비스 상태 4종 확인

```bash
systemctl status 서비스                     # 데몬 동작
ss -tlnp | grep 포트                        # 리스닝
firewall-cmd --list-all                    # 방화벽
journalctl -u 서비스 -n 50                  # 로그
```

### 3-2. 네트워크 연결 확인

```bash
ping -c 3 대상IP
ss -tlnp
telnet 대상IP 포트                          # 포트 연결 확인(진단용)
```

### 3-3. 이름 해석 확인

```bash
cat /etc/resolv.conf                        # 사용 중인 DNS
nslookup 도메인                             # 정방향
nslookup IP                                 # 역방향
```

### 3-4. 서비스별 최종 테스트

```bash
curl http://www.soldesk.com                 # 웹
ftp ftp.soldesk.com                         # FTP
sftp guest@192.168.10.100                   # SFTP
scp /tmp/a.txt guest@192.168.10.100:/tmp    # SCP
ssh guest@192.168.10.100                    # SSH
```

---

## 4. 보안 관점 정리

| 서비스 | 위험 요소 | 대응 |
|---|---|---|
| FTP | 평문 전송, 익명 업로드 | SFTP 전환, 익명 차단, chroot |
| Telnet | 평문 전송 | SSH로 대체, 서비스 비활성화 |
| SSH | 무차별 대입, root 로그인 | root 차단, 키 인증, 시도 제한, IP 제한 |
| DNS | 재귀 개방으로 증폭 공격 | `allow-recursion` 내부 제한 |
| DHCP | Rogue DHCP(중복 서버) | 서버 단일화, 스위치 DHCP Snooping |

> 공통 원칙은 최소 권한, 최소 노출, 로그 확보이다. 서비스는 필요한 것만 켜고, 접근은 필요한 대상에게만 열며, 접속·전송 기록은 반드시 남긴다.

>  **핵심 요약**
> - 파일 전송은 vsFTP(평문) vs SFTP·SCP(SSH 암호화)로 구분
> - 모든 서비스는 설치 → 설정 → 방화벽 → 재시작 → 검증 흐름
> - DNS는 zone 선언과 zone 파일 두 곳을 함께 관리
> - SSH 설정 변경은 `sshd -t` 검증 후 재시작
> - 포트·서비스 변경 시 방화벽 개방을 항상 함께 처리
> - 관련:  트러블슈팅 치트시트 ·  퀵 레퍼런스
