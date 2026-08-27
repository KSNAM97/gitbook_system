# 🗄️ DB - 데이터와 데이터베이스 기초 (MariaDB 설치 포함)

> **Tag:** #Database #MariaDB #MySQL #데이터 #정보 #DB설치 #보안설정
> **핵심 요약:** 데이터(Data)는 가공되지 않은 사실이고, 정보(Information)는 데이터를 처리해 의미를 부여한 결과물이다. 데이터베이스는 데이터를 **중복 없이, 정확하게, 효율적으로** 관리하는 시스템이며, MySQL/MariaDB는 그 구현체다. DB의 목적은 단순 저장이 아니라 **정보 가치 창출**이다.

---

## 1. 📖 개요 (Overview)

데이터(Data)는 가공되지 않은 사실(raw facts)로 그 자체로는 의미가 없는 순수한 기록이다. 예를 들어 온도 측정값 `29, 30, 31`, 시험 점수 `85`, 출근 시간 `08:57`, 회원 이름 `김철수` 등이 데이터에 해당한다. 정보(Information)는 이러한 데이터를 처리하여 의미와 가치를 부여한 결과물이며, 의사결정의 기반이 된다. 앞의 데이터를 가공하면 `오늘 평균 기온 30도`, `반 평균보다 높은 성적`, `지각 여부 판단` 같은 정보가 된다. **MySQL은 데이터를 저장하는 도구**이지 정보 자체를 만들지 않는다. 하지만 `SELECT`, 조건 검색, 집계 함수를 사용하는 이유는 **데이터를 정보로 변환**하기 위해서다. 즉 DB의 목적은 데이터 저장이 아니라 **정보 가치 창출**에 있다.

데이터베이스가 필요한 이유는 파일이나 엑셀로 데이터를 관리할 때 생기는 한계 때문이다. 첫째, 대량 데이터 문제로, 수만 개 상품을 파일로 관리하면 검색·수정이 어렵고 데이터 일관성이 무너지는데 DB는 구조적 저장과 빠른 검색, 일관성 유지를 제공한다. 둘째, 동시 접속 문제로, 100~10,000명이 동시에 주문/결제할 때 파일은 충돌이 발생하고 재고 차감 오류가 생기지만 DB는 **트랜잭션(Transaction)** 처리로 동시 작업에서도 데이터 오염 없이 처리한다. 셋째, 집계·분석 문제로, 오늘 매출·카테고리별 매출·반품률·재고 현황 등을 DB는 SQL 한 줄로 즉시 분석할 수 있으며, 그룹·정렬·집계 기능을 통해 시즌·지역별 마케팅 전략·재고 전략 수립도 가능하다. 넷째, 관계 연결 문제로, 고객 테이블 + 주문 테이블 + 상품 테이블을 **JOIN**으로 연결해 VIP 분석, 개인화 추천 등 고객 관계 분석이 가능하다. 정리하면 데이터베이스의 역할은 데이터를 정확하게 저장하고, 원하는 데이터를 빠르게 찾도록 구조화하며, 여러 사용자가 동시에 안전하게 사용하도록 관리하는 것이다.

기본 설치 상태의 MariaDB는 익명 사용자 접속 허용, root 원격 로그인 허용, 테스트 DB 존재 등 보안 취약점이 있으므로, `mysql_secure_installation` 스크립트를 실행해 이를 대화형으로 정리해야 한다. 이 스크립트에서 다루는 보안 설정 항목은 root 비밀번호 설정, unix_socket 인증 방식 선택, 익명 사용자 제거, root 원격 접속 차단, 테스트 DB 삭제, 권한 테이블 리로드 등이다. 외부 접속을 허용하려면 설정 파일(`/etc/my.cnf.d/mariadb-server.cnf`)에서 `bind-address=0.0.0.0`의 주석을 해제해야 하고, 방화벽에서 **3306/tcp 포트** 및 **mysql 서비스**를 모두 허용해야 외부에서 연결할 수 있다. Workbench(GUI 클라이언트)로 연결하기 전에는 bind-address 설정, 방화벽 설정, FLUSH PRIVILEGES 세 가지가 모두 완료되어 있어야 한다.

MariaDB 계정 생성과 권한 부여는 `CREATE USER '사용자'@'호스트' IDENTIFIED BY '비밀번호';` 로 계정을 만들고, `GRANT ALL PRIVILEGES ON *.* TO '사용자'@'호스트';` 로 권한을 부여한 뒤, `FLUSH PRIVILEGES;` 로 즉시 반영하는 순서로 이루어진다. 여기서 `'user1'@'%'`의 `%`는 모든 IP/호스트에서 접속을 허용한다는 의미이고, `'user1'@'localhost'`는 로컬에서만 접속을 허용한다. `ON *.*`에서 첫 번째 `*`는 모든 데이터베이스를, 두 번째 `*`는 해당 DB의 모든 테이블을 의미한다. `ALL PRIVILEGES`는 SELECT, INSERT, UPDATE, DELETE, CREATE, DROP 등 대부분의 권한을 포함하며, `FLUSH PRIVILEGES`는 사용자·권한 정보를 메모리에 다시 로드해 변경 사항을 즉시 반영한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** MariaDB/MySQL 계열 RDBMS.

### Step 1. MariaDB 설치 및 서비스 시작

```bash
# 설치
dnf install -y mariadb-server

# 설치 확인
rpm -qa | grep mariadb-server

# 서비스 시작 및 자동 시작 등록
systemctl start mariadb
systemctl enable mariadb

# 서비스 상태 확인
systemctl status mariadb
```

### Step 2. 보안 초기 설정 (`mysql_secure_installation`)

```bash
mysql_secure_installation
# 1. root 비밀번호 입력: (처음 설치 시 Enter)
# 2. unix_socket 인증: n
# 3. root 비밀번호 변경: y → 새 비밀번호 입력 (예: admin1234)
# 4. 익명 사용자 제거: n (테스트 환경)  / y (운영 환경)
# 5. root 원격 접속 차단: n (테스트)     / y (운영)
# 6. test DB 삭제: y
# 7. 권한 테이블 리로드: y
```

### Step 3. MariaDB 접속 및 계정·권한 설정

```sql
-- 접속
mysql -u root -p

-- 계정 생성 (모든 호스트 허용)
CREATE USER 'user1'@'%' IDENTIFIED BY '1234';

-- 권한 부여 (모든 DB, 모든 테이블)
GRANT ALL PRIVILEGES ON *.* TO 'user1'@'%';

-- 즉시 반영
FLUSH PRIVILEGES;

-- 접속 종료
exit
```

### Step 4. 방화벽 설정 (외부 접속 허용)

```bash
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload

# 확인
firewall-cmd --list-port      # 3306/tcp 출력 확인
firewall-cmd --list-service   # mysql 항목 확인
```

### Step 5. 외부 접속 허용 설정 (bind-address)

```bash
# 설정 파일 편집
vi /etc/my.cnf.d/mariadb-server.cnf

# 아래 줄의 주석(#) 제거
bind-address=0.0.0.0

# 서비스 재시작
systemctl restart mariadb
```

### Step 6. MySQL Workbench 설치

```
# 다운로드 경로
https://dev.mysql.com/downloads/workbench/
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```bash
# 설치 확인
rpm -qa | grep mariadb-server

# 서비스 상태
systemctl status mariadb

# 방화벽 포트/서비스 확인
firewall-cmd --list-port
firewall-cmd --list-service

# 접속 테스트
mysql -u root -p
mysql -u user1 -p -h <서버IP>
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. 외부에서 MariaDB 접속 불가

- **증상:** Workbench에서 연결 시 `Can't connect to MySQL server on '서버IP'` 오류.
- **원인:** bind-address가 `127.0.0.1`(로컬 전용)이거나 방화벽에서 3306 포트가 막혀 있음.
- **해결:**
  ```bash
  # 1) bind-address 확인 및 수정
  vi /etc/my.cnf.d/mariadb-server.cnf
  # bind-address=0.0.0.0 으로 변경 후 저장
  systemctl restart mariadb

  # 2) 방화벽 포트 허용
  firewall-cmd --permanent --add-port=3306/tcp
  firewall-cmd --permanent --add-service=mysql
  firewall-cmd --reload
  ```

#### 🚨 시나리오 2. 계정으로 접속 시 권한 없음 오류

- **증상:** `CREATE TABLE` 등 실행 시 `Access denied for user ...` 오류.
- **원인:** 계정 생성 후 `GRANT` 또는 `FLUSH PRIVILEGES` 를 실행하지 않음.
- **해결:**
  ```sql
  GRANT ALL PRIVILEGES ON *.* TO 'user1'@'%';
  FLUSH PRIVILEGES;
  ```

#### 🚨 시나리오 3. `systemctl enable mariadb` 이후에도 부팅 시 서비스 미시작

- **증상:** 서버 재부팅 후 MariaDB가 자동 시작되지 않음.
- **원인:** `enable` 명령은 실행했지만 이미 `disable` 상태가 고착된 경우.
- **해결:**
  ```bash
  systemctl enable --now mariadb   # enable + 즉시 시작
  systemctl is-enabled mariadb     # enabled 출력 확인
  ```

---

> 📌 **핵심 요약**
> - 데이터 → 가공 → 정보. DB의 목적은 **정보 가치 창출**
> - 파일 관리의 한계(동시성, 집계, 관계) → DB로 해결
> - MariaDB 설치 후 순서: `systemctl start/enable` → `mysql_secure_installation` → 계정·권한 설정 → 방화벽 설정 → `bind-address` 설정 → 재시작
> - 관련: 🔧 DB - SQL 문법 (DDL·DML·DCL) · 🔍 DB - SELECT·WHERE·ORDER BY·LIKE 실습 · 📋 emp·dept 테이블 정의 및 데이터
