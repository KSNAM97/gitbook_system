# 📄 Zone 파일 레코드 옵션 (TTL·SOA·NS·serial·A·AAAA)

> **Tag:** #Linux #DNS #ZoneFile #TTL #SOA #NSRecord #ARecord #AAAA #serial
> **핵심 요약:** Zone 파일 첫 줄의 `$TTL`은 다른 서버가 이 도메인 정보를 조회했을 때 결과를 캐시에 저장할 시간을 정하며, SOA는 zone 파일의 시작을 선언하고 serial·refresh·retry·expire·minimum 다섯 값으로 Secondary와의 동기화 정책을 정의한다. NS는 네임서버를, A는 IPv4 주소를, AAAA는 IPv6 주소를 매핑하며, 도메인 이름 끝의 점(`.`) 유무가 절대 이름과 상대 이름을 가르는 가장 흔한 실수 지점이다.

---

## 1. 📖 개요 (Overview)

TTL은 zone 파일의 첫 번째 줄에 작성한다. 다른 서버가 이 도메인 정보를 조회했을 때, 그 결과를 자신의 캐시(Cache)에 얼마 동안 저장할지를 정하는 설정이다. 값을 따로 지정하지 않으면 86400초(24시간)가 기본값으로 적용된다.

단위는 W(주)·D(일)·H(시간)·M(분)을 붙여서 표현할 수 있다.

```text
$TTL 3H    # 3시간 동안 캐시 유지
$TTL 1D    # 1일 동안 캐시 유지
```

IP 변경 작업이 예정되어 있다면 미리 TTL을 짧게 낮춰두면 변경 반영이 빨라진다. 변경 완료 후 다시 늘린다.

SOA는 권한의 시작이며 zone 파일이 시작된다는 의미이다. `@` 기호는 현재 도메인명(예: soldesk.com)을 뜻한다. 첫 번째 항목은 주 DNS 서버(Primary Name Server) 이름이고, 두 번째 항목은 관리자 이메일 주소이며 `@` 대신 `.`으로 표시한다.

괄호 안의 다섯 값은 다음과 같은 역할을 한다.

| 항목 | 역할 | 단위·예시 |
|---|---|---|
| serial | zone 파일이 변경될 때마다 증가시켜야 하는 버전 번호. Secondary(보조) DNS 서버가 이 번호를 보고 Master의 데이터가 변경되었는지 판단 | 보통 날짜 형식(YYYYMMDDnn), 예: `2023100201` |
| refresh | Secondary가 주기적으로 Master에 데이터 변경 여부를 확인하러 가는 주기 | 보통 1일, 예: `1D` |
| retry | Slave가 Master 접근 실패 시 얼마 후 다시 시도할지 | 보통 1시간~1일, 예: `1H` |
| expire | Slave가 Master에 계속 접근 실패할 경우 데이터를 더 이상 신뢰하지 않고 삭제할 때까지의 시간 | 보통 1주일, 예: `1W` |
| minimum | DNS 응답에서 TTL이 명시되지 않은 레코드에 적용되는 최소 TTL 값(캐시가 유지되는 최소 시간) | 예: `3H` |

NS Record는 이 도메인을 관리하는 DNS 서버의 이름을 지정한다. 도메인 이름 끝에는 반드시 점(`.`)을 붙여야 한다.

```text
@   IN  NS  ns.soldesk.com.
```

위 예시는 soldesk.com 도메인의 네임서버가 ns.soldesk.com이라는 뜻이다.

A Record(Address Record)는 도메인 이름을 IPv4 주소로 변환하는 역할을 한다.

```text
형식 : 호스트명   IN   A   IPv4주소
www  IN  A  192.168.111.100
ftp  IN  A  192.168.111.200
```

AAAA Record는 도메인 이름을 IPv6 주소로 변환하는 역할을 한다. IPv4의 A Record와 같은 역할을 IPv6용으로 수행한다.

```text
www  IN  AAAA  2001:43A1:9900:D3::871C:671
```

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 레코드 조합 예시

```text
$TTL 1D
@       IN SOA  ns.soldesk.com. admin.soldesk.com. (
                                20260721        ; serial
                                1D              ; refresh
                                1H              ; retry
                                1W              ; expire
                                3H )            ; minimum
@       IN      NS      ns.soldesk.com.         ; 네임서버 지정
@       IN      A       192.168.10.100          ; 도메인 기본 IP

ns      IN      A       192.168.10.100
www     IN      A       192.168.10.100
ftp     IN      A       192.168.10.200
```

### 2-2. serial 갱신 습관

zone 파일을 수정할 때마다 serial 값을 반드시 증가시켜야 한다. 그렇지 않으면 Secondary 서버가 변경 사실을 인지하지 못한다.

```bash
vi /var/named/soldesk.com.db     # serial 값을 20260721 → 20260722로 증가
systemctl restart named
```

### 2-3. 문법 검증

```bash
named-checkzone soldesk.com /var/named/soldesk.com.db
```

```text
zone soldesk.com/IN: loaded serial 20260722
OK
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 검증 명령어

```bash
named-checkzone <도메인명> <zone파일경로>    # zone 파일 문법 검사
nslookup www.soldesk.com                     # A 레코드 확인
dig www.soldesk.com AAAA                     # AAAA 레코드 확인
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. zone 파일을 수정했는데 Secondary·클라이언트에 반영되지 않음

- **원인:** serial 값을 증가시키지 않아 변경 사실이 전파되지 않음.
- **해결:** serial을 증가시킨 뒤 `systemctl restart named` 또는 `rndc reload`로 재로딩한다.

#### 🚨 시나리오 2. 이름이 이상하게 중복되어 조회됨

- **원인:** NS·A 레코드 값 끝의 점(`.`)을 누락해 상대 이름으로 해석됨.
- **해결:** `ns.soldesk.com.`처럼 완전한 이름(FQDN)에는 반드시 마지막 점을 붙인다.

#### 🚨 시나리오 3. TTL을 너무 길게 설정해 IP 변경이 늦게 반영됨

- **원인:** `$TTL` 값이 크면 클라이언트·중간 DNS의 캐시가 오래 유지됨.
- **해결:** 변경 예정 전에는 TTL을 짧게(예: 300초) 낮추고, 변경 완료 후 다시 원래 값으로 늘린다.

> 📌 **핵심 요약**
> - `$TTL`은 첫 줄에 위치, 캐시 유지 시간(기본 86400초)
> - SOA의 serial·refresh·retry·expire·minimum이 Secondary 동기화 정책을 결정
> - serial은 수정할 때마다 반드시 증가
> - A는 IPv4, AAAA는 IPv6 매핑
> - 관련: 🧭 DNS 개념 & Master Name Server·Zone 이론 · 🏗️ 종합실습 DNS Master + Web + FTP 통합 구성
