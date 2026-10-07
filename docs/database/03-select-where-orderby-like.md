# DB - SELECT · WHERE · ORDER BY · LIKE 실습 (emp · dept · member · product_catalog)

> **Tag:** #SQL #SELECT #WHERE #ORDERBY #LIKE #CONCAT #DML #조건검색 #정렬 #패턴검색
> **핵심 요약:** SELECT는 DML의 핵심으로 **어떤 컬럼을 / 어떤 테이블에서 / 어떤 조건으로 / 어떤 순서로** 조회할지를 정의한다. WHERE로 조건을 걸고, ORDER BY로 정렬하며, LIKE로 패턴 검색을 수행한다. 산술 연산·별칭(AS)·CONCAT 문자열 연결도 SELECT 구문 안에서 처리된다.

---

## 1. 개요 (Overview)

SELECT 문은 `SELECT 컬럼 FROM 테이블 WHERE 조건 ORDER BY 정렬기준;` 순서로 작성하며, 내부 실행 순서는 `FROM → WHERE → SELECT → ORDER BY` 다. `SELECT *`는 모든 컬럼을 조회하고, `SELECT empno, ename, sal`처럼 특정 컬럼만 조회할 수도 있다. SELECT는 조회(읽기)만 수행할 뿐 테이블 안의 값을 변경하지 않으며, UPDATE·DELETE는 별도 명령이 필요하다. `AS` 별칭은 `sal AS '월급'`처럼 컬럼 이름을 출력 시에만 바꿔 보여주는 기능이다.

SELECT 절에서는 산술 연산도 가능하다. 숫자형 컬럼은 `+`, `-`, `*`, `/` 연산이 가능하며 결과는 쿼리 실행 시에만 계산되어 출력되고 원본 데이터는 바뀌지 않는다. 예를 들어 `sal * 12`는 연봉을 계산하지만 원본 sal 값은 그대로이며, `(sal * 12) * 0.7`은 세금 30%를 제외한 연봉 실수령액을 계산한다. 계산 결과 컬럼에 별칭(AS)을 지정하면 ORDER BY에서 그 별칭을 사용할 수 있다(예: `ORDER BY annual_salary ASC`).

WHERE 절의 조건 연산자로는 비교 연산자(`=`, `!=`, `<>`, `>`, `>=`, `<`, `<=`), 논리 연산자(`AND`, `OR`, `NOT`), 날짜 비교, NULL 확인(`IS NULL`, `IS NOT NULL`)을 사용할 수 있다. `!=`와 `<>`는 "같지 않음"이라는 동일한 의미이며, `AND`는 모든 조건을 만족해야 하고 `OR`는 하나라도 만족하면 된다는 차이가 있다. `NOT`은 조건 전체를 반전시키는데, 예를 들어 `WHERE NOT (sal >= 2000 AND sal < 3000)`은 월급 2천만원대를 제외한다. `AND`/`OR`의 우선순위는 괄호로 명확히 해야 하며(예: `(age >= 30 AND age < 35) OR status='INACTIVE'`), 날짜는 문자열로 비교한다(예: `WHERE hiredate > '1981-12-31'`).

ORDER BY 정렬은 `ORDER BY 컬럼 ASC`(오름차순, 기본값) 또는 `ORDER BY 컬럼 DESC`(내림차순)로 지정하며, ASC는 기본값이므로 생략 가능하다. 다중 정렬은 쉼표로 구분하는데, 앞 컬럼이 같을 때 뒤 컬럼 기준으로 정렬한다. 예를 들어 `ORDER BY deptno ASC, sal DESC`는 부서 번호 오름차순, 같은 부서 내에서는 급여 내림차순으로 정렬한다. ORDER BY는 일반적으로 SQL 문의 **마지막 부분**에 작성한다.

LIKE 패턴 검색에서 `%`는 0개 이상의 임의 문자를, `_`는 정확히 1개의 임의 문자를 의미한다. `LIKE 'S%'`는 S로 시작하는 값, `LIKE '%S'`는 S로 끝나는 값, `LIKE '%S%'`는 S가 포함된 값을 찾는다. `LIKE '_C%'`는 두 번째 문자가 C인 값(글자 수 무관)을, `LIKE '___'`는 정확히 세 글자인 값을 찾는다. `NOT LIKE`는 조건에 해당하지 않는 값을 찾는다. 대소문자를 구분하지 않으려면 `LOWER()` 함수로 변환 후 비교하며, 예를 들어 `LOWER(ename) LIKE 's%'`는 대소문자 구분 없이 s로 시작하는 값을 찾는다.

CONCAT 함수는 여러 개의 문자열이나 컬럼 값을 하나의 문자열로 이어 붙이는 함수다. MySQL/MariaDB에서는 `||` 연산자 대신 `CONCAT()`을 사용하며, 기본 형식은 `CONCAT(값1, 값2, 값3, ...)`이다. `CONCAT(ename, '사원')`은 `SMITH사원`, `ALLEN사원` 등을 만들고, `CONCAT(ename, '_USA')`는 `SMITH_USA`, `ALLEN_USA` 등을 만든다. WHERE 절과 함께 사용해 특정 조건으로 필터링한 후 CONCAT으로 출력할 수도 있다.

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** MariaDB/MySQL 계열 RDBMS.

### Step 1. SELECT 기본 패턴

```sql
-- 모든 컬럼 조회
SELECT * FROM emp;

-- 특정 컬럼만 조회
SELECT empno, ename, sal FROM emp;

-- 산술 연산 + 별칭(AS)
SELECT
    sal AS '월급',
    sal * 0.7 AS '실수령액',
    sal * 12 AS '연봉'
FROM emp;

-- 문자 연결 (CONCAT)
SELECT ename, CONCAT(ename, '_USA') FROM emp;
SELECT ename, CONCAT(ename, ' 사원') FROM emp;
```

### Step 2. WHERE 절 조건 패턴

```sql
-- 단일 조건
SELECT * FROM emp WHERE empno = 7782;
SELECT * FROM emp WHERE sal >= 3000;

-- 범위 조건 (AND)
SELECT * FROM emp WHERE sal >= 2000 AND sal < 3000;

-- 조건 반전 (NOT)
SELECT * FROM emp WHERE NOT (sal >= 2000 AND sal < 3000);

-- 다른 값 조건
SELECT * FROM emp WHERE job != 'SALESMAN';
SELECT * FROM emp WHERE job <> 'SALESMAN';

-- 날짜 비교
SELECT * FROM emp WHERE hiredate > '1981-12-31';

-- OR 조건
SELECT * FROM member WHERE status = 'ACTIVE' OR age >= 35;

-- AND + OR 혼합 (괄호로 우선순위 명확히)
SELECT * FROM member WHERE (age >= 30 AND age < 35) OR status = 'INACTIVE';
```

### Step 3. ORDER BY 정렬 패턴

```sql
-- 오름차순 (기본값, ASC 생략 가능)
SELECT * FROM emp ORDER BY sal ASC;
SELECT * FROM emp ORDER BY empno;

-- 내림차순
SELECT * FROM emp ORDER BY sal DESC;

-- 다중 정렬 (부서 오름차순, 같은 부서 내 급여 내림차순)
SELECT ename, deptno, sal
FROM emp
ORDER BY deptno ASC, sal DESC;

-- WHERE + ORDER BY 결합
SELECT empno, ename, sal
FROM emp
WHERE sal < 2000
ORDER BY sal DESC;

-- 별칭(AS)으로 ORDER BY 참조
SELECT
    ename, job,
    sal AS salary,
    sal * 12 AS annual_salary_total,
    (sal * 12) * 0.7 AS annual_salary
FROM emp
ORDER BY annual_salary ASC;
```

### Step 4. LIKE 패턴 검색

```sql
-- 기본 패턴
SELECT * FROM emp WHERE ename LIKE 'S%';     -- S로 시작
SELECT * FROM emp WHERE ename LIKE '%S';     -- S로 끝남
SELECT * FROM emp WHERE ename LIKE '%S%';    -- S 포함
SELECT * FROM emp WHERE ename LIKE '%SC%';   -- SC 포함
SELECT * FROM emp WHERE ename LIKE '%S%C%';  -- S, C 모두 포함

-- 자릿수 지정 (_)
SELECT * FROM emp WHERE ename LIKE '_C';     -- 2글자이고 두 번째가 C
SELECT * FROM emp WHERE ename LIKE '_C%';    -- 두 번째 문자가 C (글자수 무관)
SELECT * FROM emp WHERE ename LIKE '__C%';   -- 세 번째 문자가 C

-- 부정 LIKE
SELECT * FROM emp WHERE ename NOT LIKE 'S%';     -- S로 시작하지 않음
SELECT * FROM emp WHERE ename NOT LIKE '%S%';    -- S 미포함

-- OR + LIKE 결합
SELECT * FROM emp
WHERE ename LIKE 'A%' OR ename LIKE '%N';    -- A로 시작하거나 N으로 끝남

-- 대소문자 구분 없이 검색
SELECT * FROM emp WHERE LOWER(ename) LIKE 's%';
```

### Step 5. member 테이블 DML 실습 패턴

```sql
-- INSERT
INSERT INTO member (member_id, username, password, email)
VALUES (1, 'kim', 'passwd1234', 'kim@example.com');

-- UPDATE: 특정 회원 전화번호 수정
UPDATE member
SET smartPhone = '010-2222-3333'
WHERE member_id = 3;

-- UPDATE: 특정 조건으로 상태 일괄 변경
UPDATE member
SET status = 'YOUNG'
WHERE age < 35;

-- DELETE: 특정 회원 1명 삭제
DELETE FROM member WHERE member_id = 1;

-- DELETE: 조건부 일괄 삭제
DELETE FROM member WHERE status = 'INACTIVE';
```

### Step 6. emp 테이블 대표 조회 쿼리 모음

```sql
-- 월급이 2000이상 3000미만인 사원 정보
SELECT * FROM emp WHERE sal >= 2000 AND sal < 3000;

-- 부서 10번 직원, ename 뒤에 '사원' 추가
SELECT ename, CONCAT(ename, '사원'), deptno
FROM emp WHERE deptno = 10;

-- 부서 20이면서 급여 2500 초과
SELECT sal, CONCAT(ename, '사원')
FROM emp WHERE deptno = 20 AND sal > 2500;

-- 연봉 실수령액 낮은 순
SELECT ename, job,
       sal AS salary,
       sal * 12 AS annual_salary_total,
       (sal * 12) * 0.7 AS annual_salary
FROM emp ORDER BY annual_salary ASC;

-- 1981-12-31 이후 입사 사원
SELECT * FROM emp WHERE hiredate > '1981-12-31';
```

### Step 7. product_catalog 테이블 생성 및 데이터

LIKE 실습에 사용하는 상품 카탈로그 테이블이다. 아래 DDL + INSERT를 먼저 실행해야 EX1~EX18 실습 쿼리가 동작한다.

```sql
CREATE TABLE product_catalog (
    product_id    INT PRIMARY KEY,
    product_code  VARCHAR(20)  NOT NULL,
    product_name  VARCHAR(100) NOT NULL,
    brand         VARCHAR(50)  NOT NULL,
    category      VARCHAR(30)  NOT NULL,
    model_name    VARCHAR(50),
    color         VARCHAR(30),
    description   VARCHAR(200)
);
```

```sql
INSERT INTO product_catalog VALUES
(1, 'NB-SAM-001', 'Samsung Galaxy Book', 'Samsung',
 'Notebook', 'GalaxyBook4', 'Silver', 'Lightweight office notebook');
INSERT INTO product_catalog VALUES
(2, 'NB-LEN-002', 'Lenovo ThinkPad X1', 'Lenovo',
 'Notebook', 'ThinkPad_X1', 'Black', 'Business notebook with strong security');
INSERT INTO product_catalog VALUES
(3, 'NB-APP-003', 'Apple MacBook Air', 'Apple',
 'Notebook', 'MacBookAir_M3', 'Midnight', 'Slim notebook for students');
INSERT INTO product_catalog VALUES
(4, 'NB-ASU-004', 'ASUS ROG Gaming Laptop', 'ASUS',
 'Notebook', 'ROG_Strix_G16', 'Gray', 'High performance gaming notebook');
INSERT INTO product_catalog VALUES
(5, 'PH-SAM-101', 'Samsung Galaxy S25', 'Samsung',
 'SmartPhone', 'Galaxy_S25', 'Blue', 'Latest Galaxy smart phone');
INSERT INTO product_catalog VALUES
(6, 'PH-APP-102', 'Apple iPhone 17 Pro', 'Apple',
 'SmartPhone', 'iPhone17_Pro', 'Black', 'Premium smart phone with pro camera');
INSERT INTO product_catalog VALUES
(7, 'PH-GOO-103', 'Google Pixel Phone', 'Google',
 'SmartPhone', 'Pixel_10', 'White', 'AI camera smart phone');
INSERT INTO product_catalog VALUES
(8, 'PH-XIA-104', 'Xiaomi Redmi Note', 'Xiaomi',
 'SmartPhone', 'Redmi_Note_15', 'Green', 'Affordable smart phone');
INSERT INTO product_catalog VALUES
(9, 'MN-LG-201', 'LG UltraWide Monitor', 'LG',
 'Monitor', 'UltraWide_34', 'Black', 'Wide screen monitor for office');
INSERT INTO product_catalog VALUES
(10, 'MN-SAM-202', 'Samsung Odyssey Monitor', 'Samsung',
 'Monitor', 'Odyssey_G7', 'Black', 'Curved gaming monitor');
INSERT INTO product_catalog VALUES
(11, 'MN-DEL-203', 'Dell UltraSharp Display', 'Dell',
 'Monitor', 'UltraSharp_U27', 'Silver', 'Professional color display');
INSERT INTO product_catalog VALUES
(12, 'KB-LOG-301', 'Logitech Wireless Keyboard', 'Logitech',
 'Keyboard', 'MX_Keys', 'Graphite', 'Quiet wireless keyboard');
INSERT INTO product_catalog VALUES
(13, 'KB-RAZ-302', 'Razer Mechanical Keyboard', 'Razer',
 'Keyboard', 'BlackWidow_V4', 'Black', 'RGB mechanical gaming keyboard');
INSERT INTO product_catalog VALUES
(14, 'KB-APP-303', 'Apple Magic Keyboard', 'Apple',
 'Keyboard', 'Magic_Keyboard', 'White', 'Compact wireless keyboard');
INSERT INTO product_catalog VALUES
(15, 'MS-LOG-401', 'Logitech Silent Mouse', 'Logitech',
 'Mouse', 'Silent_M650', 'White', 'Silent wireless office mouse');
INSERT INTO product_catalog VALUES
(16, 'MS-RAZ-402', 'Razer Gaming Mouse', 'Razer',
 'Mouse', 'DeathAdder_V3', 'Black', 'Fast RGB gaming mouse');
INSERT INTO product_catalog VALUES
(17, 'HD-SEA-501', 'Seagate External Hard Drive', 'Seagate',
 'Storage', 'Backup_Plus_5TB', 'Black', 'Portable backup storage');
INSERT INTO product_catalog VALUES
(18, 'SD-SAM-502', 'Samsung Portable SSD', 'Samsung',
 'Storage', 'T7_Shield', 'Blue', 'Fast portable solid state drive');
INSERT INTO product_catalog VALUES
(19, 'NW-CIS-601', 'Cisco Wireless Router', 'Cisco',
 'Network', 'Cisco_Router_AX', 'Black', 'Secure wireless network router');
INSERT INTO product_catalog VALUES
(20, 'NW-TPL-602', 'TP-Link WiFi Router', 'TP-Link',
 'Network', 'Archer_AX80', 'Black', 'High speed home WiFi router');
INSERT INTO product_catalog VALUES
(21, 'AC-USB-701', 'USB-C Multi Hub', 'Baseus',
 'Accessory', 'USB_C_HUB', 'Gray', 'USB hub with HDMI and LAN ports');
INSERT INTO product_catalog VALUES
(22, 'AC-CAB-702', 'Premium HDMI Cable', 'Ugreen',
 'Accessory', 'HDMI_2_1', 'Black', 'Supports 8K 120Hz display');
INSERT INTO product_catalog VALUES
(23, 'SP-JBL-801', 'JBL Bluetooth Speaker', 'JBL',
 'Speaker', 'Flip_7', 'Red', 'Portable waterproof speaker');
INSERT INTO product_catalog VALUES
(24, 'SP-SON-802', 'Sony Smart Speaker', 'Sony',
 'Speaker', 'Smart_Speaker_X', 'White', 'Voice controlled home speaker');
INSERT INTO product_catalog VALUES
(25, 'EV-SAL-901', 'Summer 30% Sale Package', 'EventShop',
 'Event', 'SALE_30_PERCENT', 'Mixed', 'Special 30% discount product');
INSERT INTO product_catalog VALUES
(26, 'EV-SET-902', 'Office_Set Package', 'EventShop',
 'Event', 'OFFICE_SET', 'Mixed', 'Notebook mouse and keyboard package');
```

### Step 8. product_catalog LIKE 실습 쿼리 모음

```sql
-- Samsung으로 시작하는 상품
SELECT * FROM product_catalog WHERE product_name LIKE 'samsung%';

-- Keyboard로 끝나는 상품의 코드, 이름, 브랜드
SELECT product_code, product_name, brand
FROM product_catalog WHERE product_name LIKE '%keyboard';

-- Gaming이 포함된 상품의 이름, 카테고리, 설명
SELECT product_name, category, description
FROM product_catalog WHERE product_name LIKE '%gaming%';

-- 상품 설명에 wireless 포함
SELECT * FROM product_catalog WHERE description LIKE '%wireless%';

-- 상품 코드가 NB-로 시작
SELECT product_code, product_name, brand
FROM product_catalog WHERE product_code LIKE 'nb-%';

-- EX15) 상품 코드에서 세 번째 문자가 -인 상품을 조회
SELECT product_code, product_name
FROM product_catalog
WHERE product_code LIKE '__-%';

-- EX16) 브랜드명이 정확히 세 글자인 상품을 조회
SELECT product_name, brand
FROM product_catalog WHERE brand LIKE '___';

-- EX17) 브랜드명의 두 번째 문자가 p인 상품을 조회
SELECT product_name, brand
FROM product_catalog WHERE brand LIKE '_p%';

-- EX18) 다음 조건을 모두 만족하는 상품의 상품명, 브랜드, 카테고리, 설명을 조회
--   # 상품 설명에 office 또는 business가 포함
--   # 상품명이 Apple로 시작하지 않음
--   # 검색 결과를 상품명 오름차순으로 정렬
SELECT product_name, brand, category, description
FROM product_catalog
WHERE (description LIKE '%office%' OR description LIKE '%business%')
  AND product_name NOT LIKE 'Apple%'
ORDER BY product_name ASC;
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```sql
-- 전체 데이터 확인
SELECT * FROM emp;
SELECT * FROM member;
SELECT * FROM product_catalog;

-- 건수 확인
SELECT COUNT(*) FROM emp;

-- 중복 없는 값 확인
SELECT DISTINCT job FROM emp;
SELECT DISTINCT status FROM member;

-- NULL 값 포함 여부
SELECT * FROM emp WHERE comm IS NULL;
SELECT * FROM member WHERE last_login IS NULL;
```

### 3-2. 트러블슈팅 시나리오

#### 시나리오 1. WHERE 조건에서 AND/OR 우선순위 오류

- **증상:** `WHERE age >= 30 AND age < 35 OR status='INACTIVE'` 결과가 예상과 다름.
- **원인:** `AND`가 `OR`보다 우선순위가 높아 `(age >= 30 AND age < 35) OR status='INACTIVE'`로 해석됨. 의도와 다를 수 있음.
- **해결:** 괄호로 명확히 구분:
  ```sql
  WHERE (age >= 30 AND age < 35) OR status = 'INACTIVE'
  ```

#### 시나리오 2. LIKE 검색 결과가 없음 (대소문자 문제)

- **증상:** `WHERE product_name LIKE 'samsung%'` 실행 시 결과 없음. 실제 데이터는 `Samsung`으로 저장됨.
- **원인:** MariaDB는 기본 collation에 따라 대소문자를 구분하지 않는 경우가 많지만, 일부 설정에서는 구분.
- **해결:**
  ```sql
  WHERE LOWER(product_name) LIKE 'samsung%'
  -- 또는
  WHERE product_name LIKE 'Samsung%'   -- 대소문자 맞춰 검색
  ```

#### 시나리오 3. UPDATE 시 WHERE 절 누락으로 전체 데이터 수정

```sql
-- 실수
UPDATE member SET status = 'INACTIVE';   -- WHERE 없음 → 전체 회원 상태 일괄 변경

-- 실행 전 반드시 SELECT로 대상 확인
SELECT * FROM member WHERE 조건;
-- 확인 후 UPDATE
UPDATE member SET status = 'INACTIVE' WHERE 조건;
```

**예방:** 운영 DB에서는 `SET sql_safe_updates = 1;` 설정으로 WHERE 없는 UPDATE 차단.

#### 시나리오 4. ORDER BY 별칭 참조 오류

- **증상:** `ORDER BY '월급'` 실행 시 정렬이 되지 않거나 오류 발생.
- **원인:** 문자열로 감싸면 컬럼명이 아닌 리터럴로 인식됨.
- **해결:**
  ```sql
  -- 잘못된 방법
  ORDER BY '월급'

  -- 올바른 방법 (별칭은 따옴표 없이)
  ORDER BY 월급
  -- 또는 컬럼 번호 사용
  ORDER BY 1
  ```

---

>  **핵심 요약**
> - SELECT 절에서 산술 연산·AS 별칭·CONCAT 모두 사용 가능
> - WHERE: AND/OR 우선순위 주의 → 괄호로 명확히 구분
> - ORDER BY: ASC(오름차순, 기본) / DESC(내림차순), 다중 정렬 시 쉼표 구분
> - LIKE: `%`(0개 이상 임의 문자) / `_`(정확히 1자) / 대소문자 주의 → `LOWER()` 활용
> - UPDATE·DELETE는 **반드시 WHERE 절로 대상 한정** — 없으면 전체 변경
