# 종합실습 DNS Master + Web + FTP 통합 구성

> **Tag:** #Linux #Lab #DNS #BIND #httpd #vsftpd #ZoneFile #PTR
> **핵심 요약:** Server-A를 Master Name Server 겸 웹 서버로, Server-B를 FTP 서버로 구성해 `www.soldesk.com`은 웹 서버로, `ftp.soldesk.com`은 FTP 서버로 연결되도록 하는 통합 실습이다. httpd·vsftpd 설치와 방화벽 개방, zone 파일을 이용한 정방향·역방향 DNS 구성, 클라이언트 DNS 지정과 도메인 기반 접속 검증까지 전체 흐름을 다룬다. 실무 구조를 따라 `named.conf`와 `named.rfc1912.zones`, zone 파일을 분리해 관리한다.

---

## 1. 실습 목표 (Scenario)

### 1-1. 구성도

| 구분 | IP 주소 | 역할 |
|---|---|---|
| Server-A | 192.168.10.100 | DNS Master · HTTP Web Server |
| Server-B | 192.168.10.200 | FTP Server |
| Client-L | 192.168.10.150 | DNS Client(테스트) |

```text
www.soldesk.com  →  192.168.10.100 (Web)
ftp.soldesk.com  →  192.168.10.200 (FTP)
ns.soldesk.com   →  192.168.10.100 (DNS)
```

---

## 2. 단계별 실습 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### STEP 1. Server-A를 HTTP 서버로 구성

httpd는 가장 널리 사용되는 아파치 웹서버 패키지이다.

```bash
rpm -qa | grep httpd                       # 설치 확인
dnf install -y httpd                       # 설치
systemctl start httpd                      # 시작
systemctl enable httpd                     # 자동 시작
systemctl status httpd                     # 상태 확인
```

방화벽 개방:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --reload
firewall-cmd --list-service
```

테스트 페이지 작성:

```bash
vi /var/www/html/index.html
```

```html
<style>
        h1 { text-align: center; }
</style>

<h1>Soldesk IT Academy</h1>
<p>안녕하세요 솔데스크 학원입니다.</p>
<hr>
<h3>개설 과정 안내</h3>
<ul>
        <li>AWS 데브옵스 과정</li>
        <li>네트워크 전문가 과정</li>
        <li>AI 데이터 분석 과정</li>
        <li>풀스택 개발 과정</li>
</ul>
```

확인:

```bash
curl 192.168.10.100                        # 서버 자체 확인
```

Server-A·Client-L 모두 브라우저에서 `http://192.168.10.100`으로 접속해 페이지를 확인한다.

---

### STEP 2. Server-B를 FTP 서버로 구성

호스트명 설정:

```bash
hostnamectl set-hostname Server-B
cat /etc/hostname
```

네트워크 도구·vsftpd 설치:

```bash
dnf install -y net-tools bind-utils traceroute
ip addr show                               # 주소 확인(권장)
ifconfig ens160                            # 주소 확인
```

```bash
dnf install -y vsftpd
systemctl start vsftpd
systemctl enable vsftpd
systemctl status vsftpd
```

방화벽 개방:

```bash
firewall-cmd --permanent --add-service=ftp
firewall-cmd --permanent --add-port=20/tcp
firewall-cmd --permanent --add-port=21/tcp
firewall-cmd --reload
firewall-cmd --list-all
```

FTP 기본 디렉터리 확인:

```bash
ls -l /var/ftp
# drwxr-xr-x 2 root root 6 pub ← 익명 사용자 허용 시 접근 가능한 공용 디렉터리
```

---

### STEP 3. FTP 배너 설정 및 계정 생성

```bash
vi /var/ftp/welcome.msg
```

```text
===========================================================
        Welcome to Soldesk FTP Server
===========================================================
이 서버는 솔데스크 아이티 아카데미에서 운영하는 FTP 서버입니다.
공개 디렉터리는 /pub 디렉터리만 사용 가능합니다.
업로드 문의는 관리자 승인 후 가능합니다.
문의 : admin@soldesk.co.kr
```

```bash
vi /etc/vsftpd/vsftpd.conf
```

```text
pam_service_name=vsftpd
userlist_enable=YES
banner_file=/var/ftp/welcome.msg
```

```bash
systemctl restart vsftpd
```

FTP 접속용 계정 생성:

```bash
useradd guest
passwd guest
```

---

### STEP 4. named.conf 기본 옵션 확인

```bash
vi /etc/named.conf
```

```text
options {
        listen-on port 53 { any; };        # 모든 인터페이스에서 수신
        listen-on-v6 port 53 { none; };    # IPv6 미사용
        directory       "/var/named";      # zone 파일 기본 경로
        allow-query     { any; };          # 모든 질의 처리
        recursion yes;                     # 재귀 질의 허용
        dnssec-validation no;              # 실습 환경에서 비활성화
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

```bash
named-checkconf                            # 문법 검사
```

방화벽 개방:

```bash
firewall-cmd --permanent --add-service=dns
firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --reload
```

---

### STEP 5. 정방향 Zone 등록

```bash
vi /etc/named.rfc1912.zones
```

```text
zone "soldesk.com" IN {
        type master;
        file "soldesk.com.dns";
};
```

zone 파일 작성:

```bash
cd /var/named
vi soldesk.com.dns
```

```text
$TTL 1D
@   IN  SOA  ns.soldesk.com.  admin.soldesk.com. (
                2026072101 ; serial
                1D         ; refresh
                1H         ; retry
                1W         ; expire
                3H )       ; minimum
    IN  NS   ns.soldesk.com.
    IN  A    192.168.10.100

ns  IN  A    192.168.10.100
www IN  A    192.168.10.100
ftp IN  A    192.168.10.200
```

검증:

```bash
named-checkzone soldesk.com /var/named/soldesk.com.dns
# zone soldesk.com/IN: loaded serial 2026072101
# OK
```

---

### STEP 6. 역방향 Zone 등록

```bash
vi /etc/named.rfc1912.zones
```

```text
zone "10.168.192.in-addr.arpa" IN {
        type master;
        file "soldesk.com.rev";
};
```

zone 파일 작성:

```bash
vi /var/named/soldesk.com.rev
```

```text
$TTL 1D
@   IN  SOA  ns.soldesk.com.  root.soldesk.com. (
                2026072101 ; serial
                1D         ; refresh
                1H         ; retry
                1W         ; expire
                3H )       ; minimum
    IN  NS   ns.soldesk.com.

100 IN  PTR  www.soldesk.com.
200 IN  PTR  ftp.soldesk.com.
```

검증:

```bash
named-checkzone 10.168.192.in-addr.arpa /var/named/soldesk.com.rev
tail -10 /etc/named.rfc1912.zones          # zone 선언 확인
```

---

### STEP 7. 권한 설정 및 서비스 기동

```bash
chown root:named /var/named/soldesk.com.dns
chown root:named /var/named/soldesk.com.rev
chmod 640 /var/named/soldesk.com.dns
chmod 640 /var/named/soldesk.com.rev
ls -l /var/named/

systemctl restart named
systemctl enable named
systemctl status named
```

---

### STEP 8. 클라이언트 DNS 지정

```bash
cat /etc/resolv.conf                       # 현재 DNS 확인
vi /etc/resolv.conf
```

```text
search localdomain
#nameserver 192.168.10.2 # 기존 외부 DNS 주석
nameserver 192.168.10.100                  # 구축한 DNS 지정
```

영구 적용(NetworkManager 사용 시):

```bash
nmcli con mod ens160 ipv4.dns 192.168.10.100
nmcli con mod ens160 ipv4.ignore-auto-dns yes
nmcli con up ens160
```

> `/etc/resolv.conf` 직접 편집은 NetworkManager가 덮어쓸 수 있다. 영구 적용은 `nmcli`를 사용한다.

---

## 3. 통합 검증 (Verification)

### 3-1. 정방향·역방향 조회

```bash
nslookup www.soldesk.com
nslookup ftp.soldesk.com
```

```text
> www.soldesk.com
Name:   www.soldesk.com
Address: 192.168.10.100

> ftp.soldesk.com
Name:   ftp.soldesk.com
Address: 192.168.10.200
```

```bash
nslookup 192.168.10.100
nslookup 192.168.10.200
```

```text
100.10.168.192.in-addr.arpa     name = www.soldesk.com.
200.10.168.192.in-addr.arpa     name = ftp.soldesk.com.
```

---

### 3-2. 웹 접속 검증

```bash
curl 192.168.10.100                        # IP 기반
curl http://www.soldesk.com                # 도메인 기반
```

Client-L 브라우저에서 `http://www.soldesk.com/`으로 접속하면 작성한 페이지가 표시된다.

---

### 3-3. FTP 접속 검증

Client-L에 FTP 클라이언트 설치:

```bash
dnf install -y ftp
```

도메인으로 접속:

```bash
ftp ftp.soldesk.com
```

```text
Connected to ftp.soldesk.com (192.168.10.200).
220-===========================================================
220-    Welcome to Soldesk FTP Server
220-===========================================================
Name (ftp.soldesk.com:guest): guest
230 Login successful.
ftp> pwd
257 "/home/guest" is the current directory
```

> 과거에는 브라우저에서 `ftp://` 접속이 가능했지만 현재는 대부분의 브라우저가 FTP를 지원하지 않는다. 명령행 클라이언트나 전용 도구를 사용한다.

---

## 4. 최종 체크리스트

```text
[ ] Server-A: httpd 설치·기동·자동시작
[ ] Server-A: 방화벽 http/80 개방
[ ] Server-A: index.html 작성 및 curl 확인
[ ] Server-B: vsftpd 설치·기동·자동시작
[ ] Server-B: 방화벽 ftp/20/21 개방
[ ] Server-B: 배너 설정 및 guest 계정 생성
[ ] Server-A: named 설치, 방화벽 dns/53 tcp·udp 개방
[ ] named.conf 옵션(listen-on any, allow-query any) 확인
[ ] named.rfc1912.zones에 정방향·역방향 zone 선언
[ ] soldesk.com.dns / soldesk.com.rev 작성
[ ] named-checkconf, named-checkzone 통과
[ ] zone 파일 권한 root:named 640
[ ] named 기동 및 자동시작
[ ] 클라이언트 DNS를 192.168.10.100으로 지정
[ ] nslookup 정방향·역방향 정상
[ ] curl http://www.soldesk.com 정상
[ ] ftp ftp.soldesk.com 정상
```

---

## 5. 트러블슈팅

#### 시나리오 1. www.soldesk.com 접속 시 실제 외부 홈페이지로 이동됨

- **원인:** Client-L의 `/etc/resolv.conf`가 아직 외부 DNS(192.168.10.2)를 가리키고 있음.
- **해결:** `nameserver`를 구축한 DNS 서버(192.168.10.100)로 변경한다.

#### 시나리오 2. ftp.soldesk.com 접속은 되는데 브라우저로는 안 됨

- **원인:** 대부분의 최신 브라우저가 `ftp://` 프로토콜 지원을 중단함.
- **해결:** 명령행 `ftp` 클라이언트나 전용 FTP 도구를 사용한다.

#### 시나리오 3. named 재시작 후 서비스가 죽음

- **원인:** zone 파일 문법 오류 또는 named.conf/named.rfc1912.zones 문법 오류.
- **해결:**

```bash
named-checkconf
named-checkzone soldesk.com /var/named/soldesk.com.dns
named-checkzone 10.168.192.in-addr.arpa /var/named/soldesk.com.rev
journalctl -u named -n 50
```

>  **핵심 요약**
> - Server-A는 DNS + Web, Server-B는 FTP로 역할 분리
> - zone 선언은 named.rfc1912.zones, 데이터는 /var/named
> - 정방향은 A, 역방향은 PTR 레코드
> - 서비스별 방화벽 개방(80, 20·21, 53 tcp·udp) 필수
> - 클라이언트 DNS 지정 후 도메인 기반 접속 검증
> - 관련:  DNS 개념 & Master Name Server·Zone 이론 ·  vsFTP 설치 & 접근 제어 (user_list·chroot) ·  트러블슈팅 치트시트 (SSH·vsFTP·SFTP·DHCP·DNS)
