# ⚡ 퀵 레퍼런스 (SSH·SCP·SFTP·vsFTP·DHCP·DNS)

> **Tag:** #Linux #QuickReference #Cheatsheet #SSH #SCP #SFTP #vsFTP #DHCP #DNS
> **핵심 요약:** SSH·SCP·SFTP·vsFTP·DHCP·DNS 운영에 자주 쓰는 명령과 설정 항목을 한눈에 찾을 수 있도록 정리한 참조 문서다. 서비스 관리, 방화벽, 파일 전송, DNS 조회, 로그 확인 명령을 목적별로 묶었으며 옵션 혼동이 잦은 항목(포트 지정 대소문자, userlist_deny 반전 등)을 별도로 표시했다.

---

## 1. 🔧 서비스 관리

### 1-1. systemctl

```bash
systemctl start 서비스                      # 시작
systemctl stop 서비스                       # 중지
systemctl restart 서비스                    # 재시작(설정 반영)
systemctl reload 서비스                     # 무중단 설정 재적용(지원 시)
systemctl enable 서비스                     # 부팅 자동 시작
systemctl disable 서비스                    # 자동 시작 해제
systemctl enable --now 서비스               # 시작 + 자동시작 한 번에
systemctl status 서비스                     # 상태 확인
```

### 1-2. 패키지

```bash
rpm -qa | grep 키워드                       # 설치 확인
dnf install -y 패키지                       # 설치
dnf info 패키지                             # 정보 확인
```

---

## 2. 🔥 방화벽 (firewalld)

```bash
firewall-cmd --permanent --add-service=이름  # 서비스 허용
firewall-cmd --permanent --add-port=80/tcp   # 포트 허용
firewall-cmd --reload                        # 반영
firewall-cmd --list-all                      # 전체 확인
firewall-cmd --list-service                  # 서비스만
firewall-cmd --list-port                     # 포트만
```

서비스별 필요 개방:

```text
SSH   : ssh, 22/tcp (변경 시 해당 포트)
FTP   : ftp, 20/tcp, 21/tcp
DNS   : dns, 53/tcp, 53/udp
DHCP  : dhcp, 67/udp, 68/udp
HTTP  : http, 80/tcp
```

---

## 3. 🖥️ SSH

### 3-1. 접속

```bash
ssh 호스트                                  # 현재 계정
ssh -l 계정 호스트                          # 계정 지정
ssh 계정@호스트                             # 계정 지정
ssh -p 2002 계정@호스트                     # 포트(소문자 p)
```

### 3-2. sshd_config 주요 항목

```text
Port 22
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
Banner /etc/ssh/ssh-banner
AllowUsers guest1 guest2
AllowUsers @192.168.112.
AllowGroups sshGroup
Subsystem sftp /usr/libexec/openssh/sftp-server
```

### 3-3. 검증

```bash
sshd -t                                     # 문법 검사
sshd -T | grep -i 항목                      # 최종 적용값
ss -tlnp | grep sshd
tail -f /var/log/secure
```

---

## 4. 📤 SCP

```bash
scp 계정@호스트:/원격/파일 /로컬/경로        # 다운로드
scp /로컬/파일 계정@호스트:/원격/경로        # 업로드
scp -r 소스 대상                             # 디렉터리
scp -P 2002 소스 대상                        # 포트 지정(대문자 P)
scp -p 소스 대상                             # 시간·권한 유지
scp 계정@호스트:/경로/{a,b,c} /로컬/          # 중괄호 확장 다중 파일
```

Windows:

```text
scp C:\data\a.txt guest@192.168.10.100:/temp
scp -r guest@192.168.10.100:/home/guest/* C:\data\"Data Folder"
```

대안(대용량·증분):

```bash
rsync -avz --progress /src/ 계정@호스트:/dst/
```

---

## 5. 🔒 SFTP

```bash
sftp 계정@호스트                            # 접속
sftp -P 2002 계정@호스트                    # 포트 지정(대문자 P)
```

세션 명령:

```text
pwd / lpwd          원격 / 로컬 현재 위치
ls / lls            원격 / 로컬 목록
cd / lcd            원격 / 로컬 이동
put 파일            업로드
get 파일             다운로드
get /원격/파일 ./로컬/  경로 지정 다운로드
exit / quit / bye    종료
```

---

## 6. 📁 vsFTP

### 6-1. 설정 항목

```text
anonymous_enable=NO           # 익명 차단(권장)
local_enable=YES              # 로컬 계정 허용
write_enable=YES              # 업로드·삭제 허용
local_umask=022               # 업로드 파일 권한
xferlog_enable=YES            # 전송 로그
connect_from_port_20=YES      # Active 데이터 포트
banner_file=/var/ftp/welcome.msg
userlist_enable=YES
userlist_deny=YES             # YES=차단목록 / NO=허용목록
user_config_dir=/etc/vsftpd/userconfig
chroot_local_user=YES
allow_writeable_chroot=YES
```

### 6-2. 클라이언트 명령

```text
ftp 호스트          접속
pwd                 원격 현재 위치
cd 경로             원격 이동
put 파일            업로드
get 파일            다운로드
bye / quit          종료
```

### 6-3. 주요 경로

```text
/etc/vsftpd/vsftpd.conf     메인 설정
/etc/vsftpd/ftpusers        항상 차단(PAM)
/etc/vsftpd/user_list       허용·차단 목록
/etc/vsftpd/userconfig/     계정별 설정
/var/ftp/pub                익명 공개 디렉터리
/var/log/xferlog            전송 로그
```

---

## 7. 🌐 DHCP

### 7-1. dhcpd.conf

```text
authoritative;
ddns-update-style none;

subnet 192.168.10.0 netmask 255.255.255.0 {
        option routers 192.168.10.2;
        option domain-name-servers 168.126.63.1, 8.8.8.8;
        option subnet-mask 255.255.255.0;
        option broadcast-address 192.168.10.255;
        range 192.168.10.150 192.168.10.200;
        default-lease-time 86400;
        max-lease-time 864000;
}
```

### 7-2. 명령

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf          # 문법 검사
cat /var/lib/dhcpd/dhcpd.leases            # 임대 내역
journalctl -u dhcpd -f                     # 로그
```

클라이언트:

```bash
nmcli connection show
nmcli con mod ens160 ipv4.method auto
nmcli con up ens160
```

```text
ipconfig /all                              # Windows 상세
ipconfig /release                          # 반납
ipconfig /renew                            # 재요청
```

---

## 8. 🧭 DNS (BIND)

### 8-1. 설정 파일

```text
/etc/named.conf                 전체 옵션·zone 선언
/etc/named.rfc1912.zones        zone 선언(분리 구조)
/var/named/도메인.db             정방향 zone
/var/named/도메인.rev            역방향 zone
/etc/resolv.conf                클라이언트 DNS 지정
```

### 8-2. zone 선언

```text
zone "soldesk.com" IN {
        type master;
        file "soldesk.com.db";
};

zone "10.168.192.in-addr.arpa" IN {
        type master;
        file "soldesk.com.rev";
};
```

### 8-3. zone 파일 골격

```text
$TTL 1D
@   IN SOA ns.soldesk.com. admin.soldesk.com. (
            20260721 ; serial
            1D ; refresh
            1H ; retry
            1W ; expire
            3H ) ; minimum
@   IN NS  ns.soldesk.com.
ns  IN A   192.168.10.100
www IN A   192.168.10.100
ftp IN A   192.168.10.200
100 IN PTR www.soldesk.com.      ; 역방향 파일에서 사용
```

### 8-4. 검증·조회

```bash
named-checkconf                             # 설정 문법
named-checkzone 도메인 파일경로              # zone 문법
rndc reload                                 # 무중단 재로딩

nslookup 도메인
nslookup IP
nslookup 도메인 127.0.0.1                    # 특정 서버 지정
```

### 8-5. 권한

```bash
chown root:named /var/named/파일
chmod 640 /var/named/파일
```

---

## 9. ⚠️ 혼동하기 쉬운 항목

| 항목 | 정리 |
|---|---|
| 포트 옵션 | `ssh -p`(소문자), `scp -P`·`sftp -P`(대문자) |
| userlist_deny | YES=차단목록, NO=허용목록 |
| SFTP 업로드 위치 | 로그인 시점 홈 디렉터리 기준(파일 없으면 `stat` 오류) |
| SFTP 다운로드 위치 | 원격 경로 지정 가능, 접속 위치와 무관 |
| zone 이름 | 역방향은 옥텟을 뒤집음 |
| 도메인 끝 점 | zone 파일에서 누락 시 zone명 자동 추가 |
| serial | 수정할 때마다 반드시 증가 |
| scp 디렉터리 | `-r` 없으면 실패 |
| 설정 반영 | 데몬 restart 또는 reload 필요 |
| DHCP 서버 중복 | 한 네트워크에 하나만 유지 |

> 📌 **핵심 요약**
> - 서비스 관리는 `systemctl enable --now`로 시작+자동시작 동시 처리
> - 방화벽은 서비스명과 포트를 함께 열고 `--reload` 필수
> - 포트 지정은 ssh 소문자 p, scp·sftp 대문자 P
> - DNS는 `named-checkconf`·`named-checkzone`으로 사전 검증
> - SSH 설정은 `sshd -t` 후 재시작
> - 관련: 📚 종합정리 네트워크 서비스 (SSH·SCP·SFTP·vsFTP·DHCP·DNS) · 🚨 트러블슈팅 치트시트 (SSH·vsFTP·SFTP·DHCP·DNS)
