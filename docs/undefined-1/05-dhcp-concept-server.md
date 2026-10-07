# DHCP 개념 & 서버 구성

> **Tag:** #Linux #DHCP #DORA #dhcpd #UDP67 #UDP68 #lease #NetworkManager
> **핵심 요약:** DHCP(Dynamic Host Configuration Protocol)는 클라이언트에게 IP 주소·서브넷마스크·게이트웨이·DNS 서버 주소·임대 시간 등 네트워크 통신에 필요한 정보를 자동으로 할당하는 프로토콜로 UDP 67(서버)·68(클라이언트) 포트를 사용한다. 동작 과정은 Discover → Offer → Request → ACK의 DORA 4단계이며, Rocky Linux에서는 `dhcp-server` 패키지를 설치하고 `/etc/dhcp/dhcpd.conf`에 subnet과 range를 정의한 뒤 `dhcpd` 서비스를 기동한다. 한 네트워크에 DHCP 서버가 둘 이상 동작하면 클라이언트가 어느 서버에서 주소를 받을지 예측할 수 없으므로 반드시 하나만 남겨야 한다.

---

## 1. 개요 (Overview)

PC나 서버 같은 통신 장비가 인터넷망을 통해 통신하려면 IP 주소, 서브넷마스크, 게이트웨이 IP 주소, DNS 서버 주소 등이 필요하다. DHCP는 클라이언트에게 네트워크 통신에 필요한 정보를 자동으로 할당하는 프로토콜이다.

DHCP 서버가 클라이언트에게 제공하는 주요 정보는 다음과 같다.

- IP 주소
- Subnet Mask
- Default Gateway
- DNS Server 주소
- 임대 시간(Lease Time)

DHCP는 UDP 67번과 UDP 68번을 사용한다.

```text
UDP 67 : DHCP Server가 요청을 수신
UDP 68 : DHCP Client가 응답을 수신
```

DHCP의 기본 동작 과정은 다음 네 단계의 앞 글자를 따 DORA라고 부른다.

```text
Discover : Client가 DHCP Server를 찾기 위해 Broadcast 전송
Offer    : DHCP Server가 IP 주소와 네트워크 정보를 제안
Request  : Client가 제안받은 주소의 사용을 요청
ACK      : DHCP Server가 주소 사용을 최종 승인
```

서비스 로그에서 실제 DORA 과정을 확인할 수 있다.

```text
DHCPDISCOVER from 00:0c:29:a1:c1:54 via ens160
DHCPOFFER on 192.168.10.150 to 00:0c:29:a1:c1:54
DHCPREQUEST for 192.168.10.150 (192.168.10.100)
DHCPACK on 192.168.10.150 to 00:0c:29:a1:c1:54
```

한 네트워크에 여러 DHCP 서버가 동시에 동작하면 클라이언트가 어느 서버에서 주소를 받을지 예측할 수 없다. 실습 환경에서 VMware의 내장 DHCP와 직접 구축한 DHCP 서버가 함께 동작하면 IP가 뒤섞여 테스트가 불가능해진다.

VMware Workstation에서는 다음 경로로 내장 DHCP를 중지한다.

```text
Edit → Virtual Network Editor → Change Settings
     → VMnet8 선택
          → "Use local DHCP service to distribute IP address to VMs" 체크 해제
               → Apply
```

VMware DHCP를 중지해도 VMnet8의 NAT 기능과 게이트웨이 주소는 계속 사용할 수 있다. DHCP 서버를 아직 구성하지 않은 상태에서는 클라이언트가 IPv4 주소를 받지 못하는 것이 정상이다.

Rocky Linux 9는 NetworkManager가 기본적인 DHCP 클라이언트 기능을 가지고 있기 때문에 별도의 dhcp-client 패키지를 설치하지 않아도 된다.

```bash
cat /etc/NetworkManager/system-connections/ens160.nmconnection
```

```text
[ipv4]
method=auto                # DHCP로 IP 주소 할당
```

수동 설정인 경우:

```text
[ipv4]
method=manual               # 수동으로 IP 주소 할당
address1=192.168.10.100/24,192.168.10.2
dns=192.168.10.2
```

Windows에서는 `ipconfig /all`로 확인한다.

```text
DHCP 사용 . . . . . . . . . : 예
IPv4 주소 . . . . . . . . . : 192.168.10.131(기본 설정)
임대 시작 날짜. . . . . . . : 2026년 7월 20일 월요일 오후 12:48:29
임대 만료 날짜. . . . . . . : 2026년 7월 20일 월요일 오후 1:18:30
기본 게이트웨이 . . . . . . : 192.168.10.2
DHCP 서버 . . . . . . . . . : 192.168.10.254
```

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 기존 DHCP 임대 주소 제거

```bash
nmcli connection down ens160        # ens160 down 전환(SSH 접속 중이면 세션 즉시 끊김 주의)
nmcli connection up ens160
ifconfig ens160                     # IPv4 주소가 확인되지 않음(정상)
```

Windows에서는:

```text
C:\Users\aaa> ipconfig /release     # 기존 DHCP 임대 주소 삭제
C:\Users\aaa> ipconfig /renew       # 새 주소 요청(DHCP 서버가 없으면 받을 수 없음)
```

---

### Step 2. 패키지 설치 및 방화벽 개방

```bash
dnf install -y dhcp-server
rpm -qa | grep dhcp
# dhcp-common-4.4.2-20.b1.el9.rocky.0.1.noarch
# dhcp-server-4.4.2-20.b1.el9.rocky.0.1.x86_64

ls -l /etc/dhcp
cat /etc/dhcp/dhcpd.conf             # 설치 직후에는 주석만 존재
```

```bash
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --permanent --add-port=67/udp
firewall-cmd --permanent --add-port=68/udp
firewall-cmd --reload
firewall-cmd --list-port
firewall-cmd --list-service
```

---

### Step 3. dhcpd.conf 작성

```bash
vi /etc/dhcp/dhcpd.conf
```

```text
authoritative;                                      # 이 서버가 해당 네트워크의 공식 DHCP
ddns-update-style none;                              # 동적 DNS 갱신 사용 안 함

subnet 192.168.10.0 netmask 255.255.255.0 {          # 서비스할 네트워크 대역

        option routers 192.168.10.2;                 # Client에게 할당할 Default Gateway
        option domain-name-servers 168.126.63.1, 8.8.8.8;   # Client에게 할당할 DNS 서버
        option subnet-mask 255.255.255.0;            # Client에게 할당할 Subnet Mask
        option broadcast-address 192.168.10.255;     # Client에게 할당할 Broadcast 주소

        range 192.168.10.150 192.168.10.200;         # 동적으로 할당할 IP 주소 범위

        default-lease-time 86400;                    # 기본 임대 시간: 86400초 = 1일
        max-lease-time 864000;                       # 최대 임대 시간: 864000초 = 10일
}
```

> **참고:** `range`에는 서버·게이트웨이·프린터 등 고정 IP 장비의 주소가 포함되지 않도록 설계한다. 고정 IP 대역과 동적 대역을 분리하는 것이 관리에 유리하다.

---

### Step 4. 서비스 기동

```bash
systemctl status dhcpd               # 기동 전 상태(inactive)
systemctl start dhcpd
systemctl enable dhcpd
systemctl status dhcpd
```

정상 기동 시 로그에 DORA 과정이 표시된다.

```text
Listening on LPF/ens160/00:0c:29:66:2f:9c
Sending on   LPF/ens160/00:0c:29:66:2f:9c
Server starting service.
DHCPDISCOVER from 00:0c:29:a1:c1:54 via ens160
DHCPOFFER on 192.168.10.150 to 00:0c:29:a1:c1:54
DHCPREQUEST for 192.168.10.150 (192.168.10.100)
DHCPACK on 192.168.10.150 to 00:0c:29:a1:c1:54
```

---

### Step 5. 클라이언트 확인

Linux 클라이언트는 IP 주소가 없으면 SSH 접속이 불가능하므로 콘솔로 접속해야 한다.

```bash
ifconfig ens160
# inet 192.168.10.150 netmask 255.255.255.0 broadcast 192.168.10.255
```

```bash
rpm -qa | grep dhcp     # dhcp-client 패키지가 없어도 정상(NetworkManager가 처리)
```

Windows 클라이언트:

```text
C:\Users\aaa> ipconfig /renew
IPv4 주소 . . . . . . . . . : 192.168.10.151
서브넷 마스크 . . . . . . . : 255.255.255.0
기본 게이트웨이 . . . . . . : 192.168.10.2
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 할당 내역 확인

```bash
cat /var/lib/dhcpd/dhcpd.leases
```

```text
lease 192.168.10.150 {
  starts 1 2026/07/20 05:51:34;
  ends 2 2026/07/21 05:51:34;
  binding state active;
  hardware ethernet 00:0c:29:a1:c1:54;
  client-hostname "Client-L";
}
lease 192.168.10.151 {
  starts 1 2026/07/20 05:54:26;
  ends 2 2026/07/21 05:54:26;
  binding state active;
  hardware ethernet 00:0c:29:e6:35:23;
  set vendor-class-identifier = "MSFT 5.0";
  client-hostname "DESKTOP-FOUO854";
}
```

### 3-2. IP를 받지 못할 때 점검 순서

```text
1) dhcpd 서비스가 동작 중인가         → systemctl status dhcpd
2) 방화벽 67/udp가 열려 있는가        → firewall-cmd --list-port
3) subnet 대역과 서버 IP가 일치하는가 → dhcpd.conf 확인
4) range가 고갈되지 않았는가          → dhcpd.leases 확인
5) 다른 DHCP 서버가 있는가            → VMware DHCP 중지 확인
6) 클라이언트가 자동 할당 설정인가    → method=auto / DHCP 사용 예
```

서비스 기동 실패 시 설정 문법을 먼저 확인한다.

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf     # 설정 문법 검사
journalctl -u dhcpd -n 50             # 상세 오류 확인
```

### 3-3. 트러블슈팅 시나리오

#### 시나리오 1. VMware DHCP와 직접 구축한 DHCP가 충돌

- **원인:** 한 네트워크에 두 개의 DHCP 서버가 동시에 동작.
- **해결:** VMware Workstation의 Virtual Network Editor에서 VMnet8의 내장 DHCP 체크를 해제한다. NAT·게이트웨이 기능은 그대로 유지된다.

#### 시나리오 2. Client-L이 콘솔에서만 접속 가능

- **원인:** IP를 아직 받지 못한 상태라 SSH 접속이 불가능함.
- **해결:** DHCP 서버 기동 후 클라이언트를 재부팅하거나 `nmcli connection up`으로 재요청하면 IP를 받아 SSH 접속이 가능해진다.

#### 시나리오 3. dhcpd 서비스가 기동되지 않음

- **원인:** dhcpd.conf 문법 오류 또는 subnet 대역과 서버 실제 IP 불일치.
- **해결:**

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
journalctl -u dhcpd -n 50
```

>  **핵심 요약**
> - DHCP는 IP·마스크·게이트웨이·DNS·임대시간을 자동 할당
> - UDP 67(서버)·68(클라이언트) 사용
> - 동작 과정은 Discover → Offer → Request → ACK
> - 설정은 `/etc/dhcp/dhcpd.conf`의 subnet·range
> - 한 네트워크에 DHCP 서버는 하나만 유지(VMware 내장 DHCP 중지 필수)
> - 관련:  DNS 개념 & Master Name Server·Zone 이론 ·  트러블슈팅 치트시트 (SSH·vsFTP·SFTP·DHCP·DNS)
