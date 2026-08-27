# 🔧 DB - SQL 문법 (DDL · DML · DCL)

> **Tag:** #SQL #DDL #DML #DCL #CREATE #ALTER #DROP #INSERT #SELECT #UPDATE #DELETE #GRANT #REVOKE #자료형 #제약조건
> **핵심 요약:** SQL은 크게 구조를 정의하는 **DDL**, 데이터를 다루는 **DML**, 권한을 제어하는 **DCL** 세 종류로 나뉜다. DDL은 즉시 반영되어 롤백이 어렵고, DML은 COMMIT/ROLLBACK으로 되돌릴 수 있다. 테이블 설계 시 자료형과 제약조건을 올바르게 지정하는 것이 DB 품질의 핵심이다.

---

## 1. 📖 개요 (Overview)

DDL(Data Definition Language)은 테이블·DB 등 **구조(그릇)**를 만드는 언어, DML(Data Manipulation Language)은 그 안의 **데이터(내용)**를 다루는 언어, DCL(Data Control Language)은 **접근 권한**을 관리하는 언어다. DDL은 실행 즉시 적용되고 ROLLBACK이 불가능하며 COMMIT 없이 바로 반영되어 개발·설계 단계에서 주로 사용된다. DML은 데이터 내용을 변경하며 COMMIT/ROLLBACK으로 복구가 가능해 실무에서 가장 많이 사용되는 CRUD 작업이다. DCL은 데이터 자체가 아니라 데이터 **사용 권한**을 제어하며 보안·사용자 관리·데이터 안정성 유지가 목적이다. DDL이 그릇을 만드는 언어라면, DML은 그 그릇 안의 음식을 다루는 언어라고 비유할 수 있다.

DDL의 대표 명령어는 다섯 가지로, `CREATE`(생성), `ALTER`(구조 변경), `DROP`(완전 삭제), `TRUNCATE`(데이터만 삭제), `RENAME`(이름 변경)이 있다. `DROP`과 `TRUNCATE`의 차이는, DROP은 구조와 데이터를 모두 삭제하며 되돌릴 수 없는 반면, TRUNCATE는 구조는 유지하고 데이터만 삭제한다는 점이다(DELETE보다 빠르지만 되돌릴 수 없고 AUTO_INCREMENT도 초기화된다). 실무에서는 **백업 없이 DROP/TRUNCATE 사용을 금지**해야 한다.

테이블 자료형은 저장할 데이터 성격에 맞게 선택해야 저장 공간을 아끼고 데이터 무결성을 보장할 수 있다. 금액은 `DECIMAL`, 일반 문자는 `VARCHAR`, 날짜+시간은 `DATETIME`, 로그 기록에는 `TIMESTAMP`가 적합하다. `CHAR`와 `VARCHAR`의 차이는, CHAR는 고정 길이로 항상 n바이트를 채우며 국가코드·주민번호 앞자리 등에 쓰이고, VARCHAR는 가변 길이로 실무에서 가장 많이 사용된다는 점이다. `FLOAT`와 `DECIMAL`의 차이는, FLOAT는 근사값으로 정밀도가 낮고, DECIMAL은 정확한 소수로 돈·가격 표현에 필수라는 점이다. `DATETIME`과 `TIMESTAMP`의 차이는, DATETIME은 서버 시간대와 무관하고, TIMESTAMP는 서버 시간대의 영향을 받아 로그 기록용으로 적합하다는 점이다.

제약조건(Constraints)은 테이블에 잘못된 데이터가 입력되는 것을 **미리 방지**하는 장치다. `PRIMARY KEY`, `UNIQUE`, `NOT NULL`, `DEFAULT`, `CHECK`를 컬럼에 설정해 데이터 무결성을 보장한다. `PRIMARY KEY`는 중복 불가와 NULL 불가를 동시에 만족하며 테이블에 하나만 설정 가능하고 `AUTO_INCREMENT`와 자주 함께 사용된다. `UNIQUE`는 중복 불가지만 NULL은 허용하며 이메일, 사용자 이름 등 여러 컬럼에 적용할 수 있다. `NOT NULL`은 NULL 입력을 금지하는 필수 입력 항목(비밀번호, 이름, 가격 등)에 사용한다. `DEFAULT`는 값을 입력하지 않았을 때 자동으로 들어가는 기본값이다. `CHECK`는 특정 조건을 만족하는 값만 허용하는데, MariaDB 10.x 일부 버전에서는 강제되지 않을 수 있음에 주의해야 한다.

DCL은 DB 보안·사용자 관리·데이터 안정성 유지를 위해 **접근 권한을 부여(`GRANT`)하거나 회수(`REVOKE`)**하는 언어다. 보안(Security) 측면에서는 DB에 중요 정보가 많기 때문에 권한을 세분화해 관리해야 하고, 사용자 관리 측면에서는 개발자/관리자/외부 사용자마다 접근 권한이 달라야 한다. 또한 불필요한 권한을 제거함으로써 실수로 삭제·수정하는 사고를 방지할 수 있다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. 데이터베이스 생성·삭제

```sql
-- 데이터베이스 생성
CREATE DATABASE database이름;

-- 데이터베이스 삭제
DROP DATABASE database이름;

-- 사용할 DB 선택
USE database이름;
```

### 2-2. 테이블 생성 (CREATE TABLE)

```sql
-- 기본 형식
CREATE TABLE 테이블명 (
    컬럼명  자료형  제약조건,
    ...
    PRIMARY KEY (컬럼명)
);

-- 실전 예제 (제약조건 포함)
CREATE TABLE test (
    id    VARCHAR(16)  PRIMARY KEY,
    pw    VARCHAR(20)  NOT NULL,
    age   INT          DEFAULT 18  CHECK (age >= 18 AND age <= 65),
    birth DATE
);

-- test 테이블에 데이터 삽입
INSERT INTO test VALUES('hong', '1234', 20, '2026-07-31');

-- 회원 테이블 (다양한 제약조건 포함)
CREATE TABLE member (
    member_id   INT(10)      PRIMARY KEY,
    username    VARCHAR(50)  NOT NULL,
    password    VARCHAR(100) NOT NULL,
    email       VARCHAR(100),
    phone       VARCHAR(20),
    birth_date  DATE,
    join_date   DATETIME     DEFAULT NOW(),
    status      CHAR(1)      DEFAULT 'A'
);
```

### 2-3. 테이블 삭제·데이터 전체 삭제

```sql
-- 테이블 완전 삭제 (구조 + 데이터 모두)
DROP TABLE 테이블명;

-- 데이터만 전체 삭제 (구조 유지, AUTO_INCREMENT 초기화)
TRUNCATE TABLE 테이블명;
```

### 2-4. 테이블 구조 수정 (ALTER TABLE)

```sql
-- 1) 컬럼 추가 (ADD)
ALTER TABLE member ADD address VARCHAR(100);

-- 여러 컬럼 동시 추가
ALTER TABLE member
ADD phone VARCHAR(20),
ADD regdate DATETIME;

-- 2) 컬럼 삭제 (DROP COLUMN)
ALTER TABLE member DROP COLUMN address;

-- 3) 자료형 수정 (MODIFY)
ALTER TABLE member MODIFY name VARCHAR(100);    -- 길이 변경
ALTER TABLE member MODIFY age BIGINT;           -- 자료형 변경

-- 4) 컬럼 이름 변경 + 자료형 재정의 (CHANGE)
ALTER TABLE member CHANGE age user_age INT;

-- 5) 테이블 이름 변경 (RENAME)
ALTER TABLE member RENAME TO customer;
```

### 2-4-1. ALTER TABLE member 실습 (EX1~EX7)

아래 실습은 위 `member` 테이블 생성 직후부터 순서대로 진행된다. 각 단계별로 `DESC member;`로 구조 변화를 확인한다.

```sql
-- EX1) phone 컬럼 이름을 smartPhone으로 변경
ALTER TABLE member CHANGE phone smartPhone VARCHAR(20);
```

| Field | Type | Null | Key | Default | Extra |
|---|---|---|---|---|---|
| member_id | int(10) | NO | PRI | NULL | |
| username | varchar(50) | NO | | NULL | |
| password | varchar(100) | NO | | NULL | |
| email | varchar(100) | YES | | NULL | |
| **smartPhone** | varchar(20) | YES | | NULL | |
| birth_date | date | YES | | NULL | |
| join_date | datetime | YES | | current_timestamp() | |
| status | char(1) | YES | | A | |

```sql
-- EX2) status 컬럼 타입을 CHAR(1) → VARCHAR(10)으로 변경
ALTER TABLE member MODIFY status VARCHAR(10);
```

| Field | Type | Null | Key | Default |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |
| **status** | **varchar(10)** | YES | | A |

```sql
-- EX3) email 컬럼에 NOT NULL 제약 조건 추가
ALTER TABLE member MODIFY email VARCHAR(100) NOT NULL;
```

| Field | Type | Null | Key | Default |
|---|---|---|---|---|
| **email** | varchar(100) | **NO** | | NULL |

```sql
-- EX4) 마지막 로그인 시간 저장용 last_login 컬럼 추가
ALTER TABLE member ADD last_login DATETIME;
```

| Field | Type | Null | Key | Default |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |
| **last_login** | **datetime** | YES | | NULL |

```sql
-- EX5) birth_date 컬럼 삭제
ALTER TABLE member DROP COLUMN birth_date;
```

> `birth_date` 컬럼이 목록에서 제거됨. 데이터도 함께 삭제되므로 주의.

```sql
-- EX6) 나이 저장용 age 컬럼 추가 (3자리 정수)
ALTER TABLE member ADD age INT(3);
```

| Field | Type | Null | Key | Default |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |
| **age** | **int(3)** | YES | | NULL |

```sql
-- EX7) age 컬럼의 기본값을 18로 변경
ALTER TABLE member MODIFY age INT(3) DEFAULT 18;
```

| Field | Type | Null | Key | Default |
|---|---|---|---|---|
| **age** | int(3) | YES | | **18** |

> 최종 `member` 구조: `member_id`, `username`, `password`, `email`(NOT NULL), `smartPhone`, `join_date`, `status`(VARCHAR 10), `last_login`, `age`(DEFAULT 18)

### 2-4-2. member 테이블 데이터 삽입 실습

```sql
-- EX8) 새 회원 1명 추가 (기본 형태)
INSERT INTO member (member_id, username, password, email)
VALUES (1, 'kim', 'passwd1234', 'kim@example.com');

-- EX9) 전체 회원 조회
SELECT * FROM member;
-- EX10) username과 email만 조회
SELECT username, email FROM member;
```

아래는 ALTER TABLE 실습 이후 바뀐 구조(`smartPhone`, `last_login`, `age` 포함)에 맞춰 20명 데이터를 삽입한다.

```sql
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(1,  'user01', 'pass1234',  'user01@test.com', '010-1001-1001', 'ACTIVE',   NOW(), 20);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(2,  'user02', 'pass1234',  'user02@test.com', '010-1002-1002', 'ACTIVE',   NOW(), 21);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(3,  'user03', 'pass1234',  'user03@test.com', '010-1003-1003', 'ACTIVE',   NOW(), 22);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(4,  'user04', 'pass1234',  'user04@test.com', '010-1004-1004', 'ACTIVE',   NOW(), 23);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(5,  'user05', 'pass1234',  'user05@test.com', '010-1005-1005', 'ACTIVE',   NOW(), 24);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(6,  'user06', 'pass1234',  'user06@test.com', '010-1006-1006', 'ACTIVE',   NOW(), 25);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(7,  'user07', 'pass1234',  'user07@test.com', '010-1007-1007', 'ACTIVE',   NOW(), 26);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(8,  'user08', 'pass1234',  'user08@test.com', '010-1008-1008', 'ACTIVE',   NOW(), 27);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(9,  'user09', 'pass1234',  'user09@test.com', '010-1009-1009', 'ACTIVE',   NOW(), 28);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(10, 'user10', 'pass1234',  'user10@test.com', '010-1010-1010', 'ACTIVE',   NOW(), 29);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(11, 'user11', 'pass5678',  'user11@test.com', '010-1011-1011', 'ACTIVE',   NOW(), 30);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(12, 'user12', 'pass5678',  'user12@test.com', '010-1012-1012', 'ACTIVE',   NOW(), 31);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(13, 'user13', 'pass5678',  'user13@test.com', '010-1013-1013', 'INACTIVE', NULL,  32);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(14, 'user14', 'pass5678',  'user14@test.com', '010-1014-1014', 'ACTIVE',   NOW(), 33);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(15, 'user15', 'pass5678',  'user15@test.com', '010-1015-1015', 'INACTIVE', NULL,  34);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(16, 'user16', 'linux1234', 'user16@test.com', '010-1016-1016', 'ACTIVE',   NOW(), 35);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(17, 'user17', 'linux1234', 'user17@test.com', '010-1017-1017', 'ACTIVE',   NOW(), 36);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(18, 'user18', 'linux1234', 'user18@test.com', '010-1018-1018', 'INACTIVE', NULL,  37);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(19, 'user19', 'maria1234', 'user19@test.com', '010-1019-1019', 'ACTIVE',   NOW(), 38);
INSERT INTO member (member_id, username, password, email, smartPhone, status, last_login, age) VALUES
(20, 'user20', 'maria1234', 'user20@test.com', '010-1020-1020', 'ACTIVE',   NOW(), 39);
```

### 2-4-3. member 테이블 SELECT · UPDATE · DELETE 실습 (EX11~EX24)

```sql
-- EX11) member_id가 2인 회원 조회 (PK 사용)
SELECT * FROM member WHERE member_id = 2;

-- EX12) status가 'ACTIVE'인 회원만 조회
SELECT * FROM member WHERE status = 'ACTIVE';

-- EX13) age가 35 이상인 회원 조회
SELECT * FROM member WHERE age >= 35;

-- EX14) age가 30 미만인 회원 조회
SELECT * FROM member WHERE age < 30;

-- EX15) status = 'ACTIVE' 이고 age >= 35 (AND 조건)
SELECT * FROM member WHERE status = 'ACTIVE' AND age >= 35;

-- EX16) status = 'ACTIVE' 이거나 age >= 35 (OR 조건)
SELECT * FROM member WHERE status = 'ACTIVE' OR age >= 35;

-- EX17) age 30~34 이거나 status = 'INACTIVE' (AND+OR 혼합)
SELECT * FROM member
WHERE (age >= 30 AND age < 35) OR status = 'INACTIVE';

-- EX18) member_id=3인 회원의 전화번호 수정
UPDATE member
SET smartPhone = '010-2222-3333'
WHERE member_id = 3;
SELECT * FROM member WHERE member_id = 3;

-- EX19) username='user10'인 회원 상태를 'INACTIVE'로 변경
UPDATE member
SET status = 'INACTIVE'
WHERE username = 'user10';
SELECT * FROM member WHERE username = 'user10';

-- EX20) age가 35 미만인 회원의 status를 'YOUNG'으로 일괄 변경
UPDATE member
SET status = 'YOUNG'
WHERE age < 35;
SELECT * FROM member;

-- EX21) member_id=1인 회원 삭제
DELETE FROM member WHERE member_id = 1;
SELECT * FROM member WHERE member_id = 1;  -- 결과 없음

-- EX22) status='INACTIVE'인 회원 모두 삭제
DELETE FROM member WHERE status = 'INACTIVE';
SELECT * FROM member;

-- EX23) age=30인 회원 조회
SELECT * FROM member WHERE age = 30;

-- EX24) age가 30이 아닌 회원 조회 (!=, <> 동일)
SELECT * FROM member WHERE age != 30;
-- 또는
SELECT * FROM member WHERE age <> 30;
```

### 2-5. 자료형 빠른 참조표

| 분류 | 자료형 | 설명 | 사용 예 |
|---|---|---|---|
| 숫자 | `INT` | 정수 (약 ±21억) | 나이, 수량, 주문번호 |
| 숫자 | `BIGINT` | 더 큰 정수 (±9경) | 매우 큰 PK, 로그ID |
| 숫자 | `FLOAT` | 소수(근사값) | 온도, 정밀도 불필요한 실수 |
| 숫자 | `DOUBLE` | 높은 정밀도 소수 | 과학 계산 |
| 숫자 | `DECIMAL(p,s)` | 정확한 소수 | 돈·가격 (필수) |
| 문자 | `CHAR(n)` | 고정 길이 | 국가코드, 주민번호 앞자리 |
| 문자 | `VARCHAR(n)` | 가변 길이 (실무 최다 사용) | 이름, 이메일, 주소 |
| 문자 | `TEXT` | 긴 문자열 | 게시판 본문 |
| 문자 | `LONGTEXT` | 매우 긴 텍스트 | 로그, 대용량 문서 |
| 날짜 | `DATE` | 날짜 (YYYY-MM-DD) | 생년월일 |
| 날짜 | `TIME` | 시간 (HH:MM:SS) | 시각 기록 |
| 날짜 | `DATETIME` | 날짜+시간 (실무 최다) | 가입일시, 주문시각 |
| 날짜 | `TIMESTAMP` | UNIX 시간 기반 | 로그 기록 |
| 논리 | `BOOLEAN` | TRUE/FALSE | 활성화 여부 |

### 2-6. 제약조건 빠른 참조

```sql
-- PRIMARY KEY (중복 불가, NULL 불가, 테이블당 1개)
id INT PRIMARY KEY
id INT AUTO_INCREMENT PRIMARY KEY

-- UNIQUE (중복 불가, NULL 허용, 여러 개 가능)
email VARCHAR(100) UNIQUE

-- NOT NULL (NULL 입력 금지)
pw VARCHAR(20) NOT NULL

-- DEFAULT (기본값 설정)
age INT DEFAULT 18
status VARCHAR(10) DEFAULT 'active'
reg_date DATETIME DEFAULT CURRENT_TIMESTAMP

-- CHECK (조건 범위 제한)
age INT CHECK (age >= 18 AND age <= 65)
score INT CHECK (score BETWEEN 0 AND 100)
```

### 2-7. DCL - 권한 부여·회수

```sql
-- 권한 부여
GRANT ALL PRIVILEGES ON *.* TO 'user1'@'%';
GRANT SELECT, INSERT ON mydb.* TO 'user2'@'localhost';

-- 권한 회수
REVOKE ALL PRIVILEGES ON *.* FROM 'user1'@'%';

-- 권한 즉시 반영
FLUSH PRIVILEGES;
```

### 2-8. 테이블 구조 확인

```sql
-- 테이블 구조 확인 (컬럼, 타입, NULL 여부, 기본값 등)
DESC 테이블명;
DESCRIBE 테이블명;

-- 생성된 테이블 목록
SHOW TABLES;

-- 데이터베이스 목록
SHOW DATABASES;
```

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```sql
SHOW DATABASES;                    -- DB 목록 확인
USE mydb;                          -- DB 선택
SHOW TABLES;                       -- 테이블 목록
DESC member;                       -- 테이블 구조 확인
SELECT * FROM member LIMIT 5;      -- 데이터 샘플 확인
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. `DROP TABLE` 실행 후 데이터가 사라짐

- **증상:** 실수로 `DROP TABLE member;` 실행 → 테이블과 데이터 모두 삭제됨.
- **원인:** DDL은 즉시 반영되며 ROLLBACK이 불가능.
- **해결/예방:**
  - 실무에서는 DROP 전 반드시 **백업** 실행.
  - 데이터만 삭제하려면 `DELETE FROM member;` 또는 `TRUNCATE TABLE member;` 사용.

#### 🚨 시나리오 2. `ALTER TABLE MODIFY` 후 기존 데이터 손실

- **증상:** 컬럼 자료형 변경 시 기존 값이 잘려나가거나 오류 발생.
- **원인:** 기존 데이터가 새 자료형과 호환되지 않음 (예: VARCHAR → INT, 값이 문자열인 경우).
- **해결:**
  - 변경 전 `SELECT DISTINCT 컬럼명 FROM 테이블명;` 으로 실제 값 확인.
  - 새 컬럼 추가 → 데이터 이전 → 기존 컬럼 삭제 순으로 안전하게 진행.

#### 🚨 시나리오 3. `CHECK` 제약조건이 동작하지 않음 (MariaDB)

- **증상:** `age INT CHECK (age >= 18)` 설정 후 나이 5를 입력해도 오류 없이 저장됨.
- **원인:** MariaDB 10.x 일부 버전에서 CHECK 구문은 파싱되지만 강제 적용되지 않음.
- **해결:**
  - MariaDB 10.3.10+ 에서는 `CONSTRAINT` 이름 명시 시 적용되는 경우 있음.
  - 애플리케이션 레이어에서 별도 유효성 검사 로직 추가 권장.

#### 🚨 시나리오 4. `AUTO_INCREMENT` 값이 TRUNCATE 후에도 초기화되지 않음

- **증상:** `DELETE FROM member;` 후 새로 데이터를 넣으면 id가 1부터 시작하지 않음.
- **원인:** `DELETE`는 AUTO_INCREMENT 값을 초기화하지 않음.
- **해결:**
  ```sql
  TRUNCATE TABLE member;              -- AUTO_INCREMENT도 1로 초기화
  -- 또는
  ALTER TABLE member AUTO_INCREMENT = 1;
  ```

---

> 📌 **핵심 요약**
> - DDL(구조) · DML(데이터) · DCL(권한) 세 종류 구분
> - DDL은 즉시 반영·롤백 불가 → **실행 전 반드시 확인**
> - 자료형: 돈 → `DECIMAL`, 일반 문자 → `VARCHAR`, 날짜시간 → `DATETIME`
> - 제약조건: `PRIMARY KEY` > `NOT NULL` > `UNIQUE` > `DEFAULT` > `CHECK` 순으로 자주 사용
> - 관련: 🗄️ DB - 데이터와 데이터베이스 기초 · 🔍 DB - SELECT·WHERE·ORDER BY·LIKE 실습 · 📋 emp·dept 테이블 정의 및 데이터
