# DNS 개념 & Master Name Server·Zone 이론

> **Tag:** #Linux #DNS #BIND #named #MasterNameServer #Zone #named.conf #ReverseDNS
> **핵심 요약:** Master Name Server는 특정 도메인에 대한 최종 권한(Authoritative)을 가지고 Zone File을 직접 관리하는 DNS 서버로, A·NS·MX·CNAME 등의 레코드를 보유하고 질의에 공식 응답을 제공한다. Linux에서 DNS 서버 소프트웨어는 BIND이고 실행되는 데몬은 `named`이며, 전체 동작은 `/etc/named.conf`에서, 실제 레코드는 `/var/named/` 아래 Zone File에서 관리한다. 정방향(도메인→IP)은 A 레코드, 역방향(IP→도메인)은 PTR 레코드가 담당하며, 역방향 DNS는 메일 서버 신뢰성 검증과 로그 분석에 특히 중요하다.

---

## 1. 개요 (Overview)

Master Name Server는 해당 도메인에 대한 권한을 가진(Authoritative) DNS 서버 중 하나이며, Zone File(영역 파일)을 직접 관리한다. 즉 `example.com` 도메인의 IP 주소 매핑 정보(A, MX, NS, CNAME 등)를 직접 보유하며, DNS 질의가 들어오면 해당 도메인에 대한 공식적인 응답 정보를 제공한다. 다른 서버(Secondary Name Server)는 이 Master 서버로부터 데이터를 복제(Zone Transfer)받는다.

주요 특징은 다음과 같다.

- 권한: 해당 도메인에 대한 최종 권한(Authoritative)을 가짐
- 데이터 저장 위치: `/var/named/example.com.zone` 같은 zone 파일에 직접 저장
- 데이터 변경: 관리자가 직접 수정 가능
- 동기화: Secondary 서버에 데이터를 전송
- 역할: 도메인에 대한 공식 응답을 제공하는 원본 DNS

동작 과정은 다음과 같다.

```text
1) 내부 사용자가 www.soldesk.com에 접속
2) 로컬 DNS가 Master Name Server에 질의
3) Master 서버는 zone 파일을 참조해 IP 응답
4) 로컬 DNS는 해당 결과를 캐싱 후 사용자에게 반환
5) Secondary 서버가 있을 경우, Master 서버로부터 zone 데이터를 복제해 유지
```

Master·Secondary·Caching 서버의 차이는 다음과 같다.

| 구분 | Master Name Server | Secondary Name Server | Caching Name Server |
|---|---|---|---|
| 역할 | 원본 DNS 데이터 관리 | Master에서 데이터 복제 | 외부 질의 결과를 캐시 |
| zone 파일 보유 | 있음(직접 수정 가능) | 있음(복제본) | 없음 |
| 데이터 변경 | 수동 또는 자동으로 수정 | Master로부터만 복사 | 없음 |
| 용도 | 내부 도메인 운영 | 부하 분산, 백업 | 클라이언트 성능 향상 |

현재 BIND에서는 Master/Slave 대신 Primary/Secondary라는 용어도 함께 사용한다.

BIND, named, Zone, Zone File의 관계는 다음과 같다.

| 구분 | 이름 | 설명 |
|---|---|---|
| 프로그램 이름 | BIND | DNS 서버 소프트웨어의 이름(패키지 이름) |
| 데몬(서비스 이름) | named | 실제로 실행되는 DNS 서버 프로세스 |
| 데이터 디렉터리 | `/var/named/` | named가 사용하는 zone 파일 저장 위치 |
| 기본 설정 파일 | `/etc/named.conf` | 서버 전체 동작 정의 |

`named.conf`는 BIND의 서비스 프로세스인 named가 시작될 때 읽어 들이며, 어떤 도메인을 관리할지, Zone File이 어디에 있는지, 어떤 IP 주소에서 DNS 요청을 받을지 등을 설정한다. DNS 질의를 허용할 대상, 재귀 질의 허용 여부, 캐싱 및 보안 정책도 여기서 정의한다.

Domain(도메인)은 인터넷에서 쓰는 이름 전체 구조를 의미하고, Zone(존)은 그 도메인 중에서 해당 DNS 서버가 직접 관리하는 부분만 잘라낸 것이다. 도메인 전체 중 우리 DNS 서버가 맡은 부분이 Zone이고, 그 관리 데이터가 들어 있는 파일이 Zone 파일이다.

```text
soldesk.com Zone을 등록하면 다음 이름들을 관리할 수 있다.
  www.soldesk.com
  mail.soldesk.com
  ftp.soldesk.com
  ns.soldesk.com
```

DNS 서버 설정은 두 군데에서 이루어진다.

- `/etc/named.conf`: 어떤 도메인을 관리할지 선언하는 곳(zone 파일 위치도 여기서 지정)
- `/var/named/`: 실제 zone 파일이 저장되는 디렉터리

zone 파일을 `/var/named/`에 두는 이유는 `named.conf`의 `directory` 항목이 기본 경로를 `/var/named`로 지정하고 있기 때문이다.

named.conf의 주요 옵션은 다음과 같다.

```text
listen-on port 53 { any; };        # 127.0.0.1=자신만 처리 | any=모든 DNS 요청 처리
listen-on-v6 port 53 { none; };    # IPv6 사용 안 함
directory       "/var/named";      # zone 파일 기본 경로
allow-query     { any; };          # localhost=자신만 | any=모든 요청 처리
recursion yes;                     # 재귀 질의 허용 여부
dnssec-validation no;              # 실습용 로컬 네임서버는 DNSSEC 검증 불필요
```

`recursion`은 서버 성격에 따라 결정한다. Authoritative(권한) 전용 서버라면 재귀를 끄고, 내부 클라이언트용 재귀(캐싱) 서버라면 켜되 접근 대상을 반드시 제한해야 한다.

공인 IP를 가진 서버에서 `recursion yes`와 `allow-query { any; }`를 함께 사용하면 DNS 증폭(Amplification) 공격에 악용될 수 있다. 운영 환경에서는 재귀 질의 허용 대상을 내부 대역으로 제한한다.

Zone 파일의 SOA 구조와 각 값의 의미는 다음과 같다. Zone 파일 예시(`soldesk.com.db`):

```text
$TTL 1D
@       IN      NS      ns.soldesk.com.
@       IN      A       192.168.10.100

ns      IN      A       192.168.10.100
www     IN      A       192.168.10.100
ftp     IN      A       192.168.10.200
```

SOA(Start Of Authority)는 권한의 시작이며 zone 파일이 시작된다는 의미이다. `@` 기호는 현재 도메인명(zone 선언에 쓴 이름)을 뜻한다. 첫 번째 항목은 주 DNS 서버(Primary Name Server) 이름, 두 번째 항목은 관리자 이메일 주소이며 `@` 대신 `.`으로 표기한다.

```text
@   IN SOA  ns.soldesk.com.  admin.soldesk.com. (
                2026072101 ; serial
                1D         ; refresh
                1H         ; retry
                1W         ; expire
                3H )       ; minimum
```

> **참고:** 관리자 이메일을 표기할 때 `admin.soldesk.com.`처럼 마지막 점을 포함하는 것이 원칙이다. 도메인이 아닌 다른 조직의 이메일 표기(예: 실습 중 실수로 입력한 `admin.naver.com`)를 그대로 두면 SOA 레코드의 의미가 왜곡되므로, 반드시 자신이 관리하는 도메인의 관리자 주소로 통일해야 한다.

| 값 | 역할 |
|---|---|
| serial | zone 파일 버전 번호. 변경할 때마다 증가시켜야 Secondary가 갱신을 인지 |
| refresh | Secondary가 Master에 변경 여부를 확인하러 가는 주기 |
| retry | Master 접근 실패 시 재시도 간격 |
| expire | 계속 실패할 경우 데이터를 신뢰하지 않고 폐기하기까지의 시간 |
| minimum | TTL이 명시되지 않은 레코드에 적용되는 최소 캐시 시간 |

named-checkzone 명령은 특정 도메인(zone)에 대한 zone 데이터 파일의 문법과 구조가 올바른지 검사하는 도구다. 즉 `named.conf` 설정이 아니라, 실제 zone 파일 내부의 A·NS·SOA 등의 구문이 규칙에 맞게 작성되어 있는지를 확인한다.

```bash
named-checkzone soldesk.com /var/named/soldesk.com.db
# zone soldesk.com/IN: loaded serial 20260721
# OK
```

정방향 DNS는 도메인을 IP로 변환하고, 역방향 DNS는 반대로 IP 주소를 도메인 이름으로 변환하며 PTR(Pointer) 레코드가 담당한다.

```text
정방향 : www.soldesk.com  --->  192.168.10.100
역방향 : 192.168.10.100   --->  www.soldesk.com
```

역방향 DNS는 특히 메일 서버의 신뢰성을 확인할 때 중요하게 사용된다. SMTP(메일 전송 표준 프로토콜) 서버는 메일을 전송한 서버가 정상적으로 등록된 서버인지 확인하기 위해 두 가지를 검사한다.

```text
1) 메일을 전송한 서버의 IP 주소에 PTR 레코드가 등록되어 있는가?
2) PTR로 확인된 도메인 이름을 정방향 조회했을 때 원래 IP 주소가 다시 조회되는가?
```

이처럼 역방향 조회 결과를 다시 정방향으로 조회해 원래 IP와 일치하는지 확인하는 방식을 FCrDNS(Forward-confirmed Reverse DNS)라고 한다. 역방향 DNS가 없으면 정체불명의 서버로 판단되어 스팸 점수가 올라가고, 메일이 스팸함으로 이동하거나 수신 거부될 수 있다.

로그 분석에도 도움이 된다. 로그에는 대부분 IP만 기록되는데, 역방향 DNS가 있으면 이름이 함께 보여 누가 접속했고 어떤 장비인지 한눈에 알 수 있다.

```text
Failed password for 192.168.10.200
→ 역방향 조회 시 192.168.10.200 --> ftp.soldesk.com 으로 확인 가능
```

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 기본 named.conf 확인

```bash
vi /etc/named.conf
```

```text
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { none; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        allow-query     { any; };
        recursion yes;
        dnssec-validation no;
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

정방향 zone 선언 추가:

```text
zone "soldesk.com" IN {
        type master;
        file "soldesk.com.db";
};
```

```bash
named-checkconf                # named.conf 문법 검사
```

---

### Step 2. Zone 파일 준비 및 작성

```bash
ls -l /var/named
```

```text
-rw-r----- 1 root named 2112  named.ca
-rw-r----- 1 root named  152  named.empty
-rw-r----- 1 root named  152  named.localhost
-rw-r----- 1 root named  168  named.loopback
```

예시 파일을 복사해 시작한다.

```bash
cp /var/named/named.empty /var/named/soldesk.com.db
cat /var/named/soldesk.com.db
```

```text
$TTL 3H
@       IN SOA  @ rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      @
        A       127.0.0.1
        AAAA    ::1
```

실제 도메인에 맞게 작성:

```bash
vi /var/named/soldesk.com.db
```

```text
$TTL 1D
@       IN SOA  ns.soldesk.com. admin.soldesk.com. (
                                20260721        ; serial
                                1D              ; refresh
                                1H              ; retry
                                1W              ; expire
                                3H )            ; minimum
@       IN      NS      ns.soldesk.com.         ; soldesk.com의 DNS 서버
@       IN      A       192.168.10.100          ; soldesk.com의 기본 IP

ns      IN      A       192.168.10.100
www     IN      A       192.168.10.100
ftp     IN      A       192.168.10.200
```

---

### Step 3. 문법 검증 및 권한 설정

```bash
named-checkzone soldesk.com /var/named/soldesk.com.db
# zone soldesk.com/IN: loaded serial 20260721
# OK
```

```bash
chown root:named /var/named/soldesk.com.db
chmod 640 /var/named/soldesk.com.db
ls -l /var/named/
```

> **참고:** named 데몬은 `named` 사용자로 실행되므로 그룹 읽기 권한이 필요하다. other에 권한을 주지 않는 것이 보안상 바람직하다.

---

### Step 4. 역방향 Zone 등록

```bash
vi /etc/named.conf
```

```text
zone "soldesk.com" IN {
        type master;
        file "soldesk.com.db";
};

zone "10.168.192.in-addr.arpa" IN {    # 192.168.10.0/24 역방향
        type master;
        file "soldesk.com.rev";
};
```

```bash
cp /var/named/named.empty /var/named/soldesk.com.rev
vi /var/named/soldesk.com.rev
```

```text
$TTL 1D
@       IN SOA  ns.soldesk.com. root.soldesk.com. (
                                2026072101      ; serial
                                1D              ; refresh
                                1H              ; retry
                                1W              ; expire
                                3H )            ; minimum
@       IN      NS      ns.soldesk.com.

100     IN      PTR     www.soldesk.com.        ; 192.168.10.100
200     IN      PTR     ftp.soldesk.com.        ; 192.168.10.200
```

```bash
named-checkzone 10.168.192.in-addr.arpa /var/named/soldesk.com.rev
chown root:named /var/named/soldesk.com.rev
chmod 640 /var/named/soldesk.com.rev
```

---

### Step 5. 서비스 반영 및 클라이언트 DNS 지정

```bash
systemctl restart named
systemctl enable named
systemctl status named
```

```bash
cat /etc/resolv.conf
vi /etc/resolv.conf
```

```text
search localdomain
#nameserver 192.168.10.2 # 기존(외부) DNS 주석 처리
nameserver 192.168.10.100      # 구축한 DNS 서버 지정
```

> **참고:** `/etc/resolv.conf`는 NetworkManager가 재작성할 수 있다. 영구 적용하려면 `nmcli con mod ens160 ipv4.dns 192.168.10.100` 후 연결을 재적용하는 방식을 사용한다.

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 조회 테스트

```bash
nslookup www.soldesk.com
nslookup 192.168.10.100 127.0.0.1        # 서버 자체에서 역방향 검증
```

```text
> **참고:** www.soldesk.com
Name:   www.soldesk.com
Address: 192.168.10.100

> **참고:** 192.168.10.100 127.0.0.1
100.10.168.192.in-addr.arpa     name = www.soldesk.com.
```

공인 DNS로 역방향 원리 확인:

```bash
nslookup dns.google
nslookup 8.8.8.8
# 8.8.8.8.in-addr.arpa name = dns.google.
```

### 3-2. 실제 사이트로 연결될 때

내부 DNS를 만들었는데 실제 외부 홈페이지로 접속된다면 클라이언트가 아직 외부 DNS를 바라보고 있는 것이다.

```bash
cat /etc/resolv.conf         # nameserver 확인
```

### 3-3. 서비스가 기동되지 않을 때

```bash
named-checkconf
named-checkzone soldesk.com /var/named/soldesk.com.db
journalctl -u named -n 50
ls -l /var/named/            # 파일 권한(root:named 640) 확인
```

### 3-4. 트러블슈팅 시나리오

#### 시나리오 1. zone 로딩은 되는데 www.soldesk.com이 외부 사이트로 연결됨

- **원인:** `/etc/resolv.conf`가 여전히 외부 DNS(192.168.10.2 등)를 가리키고 있음.
- **해결:** `nameserver`를 구축한 DNS 서버(192.168.10.100)로 변경하고 재확인한다.

#### 시나리오 2. 역방향 조회가 NXDOMAIN으로 실패

- **원인 후보:** zone 이름의 옥텟 순서 오류, PTR 값 끝의 점(`.`) 누락.
- **해결:** zone 이름을 네트워크 대역을 뒤집은 형태(`10.168.192.in-addr.arpa`)로 정확히 선언했는지, PTR 값 끝에 점이 있는지 확인한다.

#### 시나리오 3. zone 파일 권한 오류로 named 기동 실패

- **원인:** zone 파일 소유자·권한이 `root:named 640`이 아님.
- **해결:** `chown root:named`, `chmod 640` 재적용 후 재시작.

>  **핵심 요약**
> - Master는 zone 파일을 직접 관리하는 권한 서버, Secondary는 복제본, Caching은 zone 없이 캐시만 보유
> - BIND는 소프트웨어, named는 데몬, `/var/named`는 데이터 경로
> - SOA의 serial은 수정할 때마다 반드시 증가시켜야 함
> - 정방향은 A, 역방향은 PTR 레코드, 메일 신뢰성(FCrDNS)에 필수
> - zone 파일 작성 후 `named-checkzone`, 권한은 `root:named 640`
> - 관련:  DHCP 개념 & 서버 구성 ·  종합실습 DNS Master + Web + FTP 통합 구성 ·  Zone 파일 레코드 옵션 (TTL·SOA·A·AAAA)
