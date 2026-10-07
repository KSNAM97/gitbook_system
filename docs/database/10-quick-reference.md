# DB - SQL 퀵 레퍼런스

> **Tag:** #Database #MariaDB #SQL #QuickReference #CheatSheet
> **핵심 요약:** MariaDB 설치·접속·계정 관리부터 DDL·DML·DCL·자료형·제약조건(PK·UNIQUE·FK)·SELECT·WHERE·ORDER BY·LIKE·CONCAT·INNER JOIN·GROUP BY·HAVING·집계함수까지 빠르게 조회하는 암기 카드. 이해가 아니라 **"조회·복붙"** 이 목적이다.

---

## 1. MariaDB 설치 · 접속 · 계정 (Configuration)

```bash
# 설치 및 서비스
dnf install -y mariadb-server
systemctl start mariadb
systemctl enable mariadb
systemctl status mariadb
systemctl restart mariadb

# 보안 초기 설정
mysql_secure_installation

# 접속
mysql -u root -p
mysql -u user1 -p -h <서버IP>
exit                          # 접속 종료

# 방화벽
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload
firewall-cmd --list-port
firewall-cmd --list-service

# 외부 접속 허용 (설정 파일)
# /etc/my.cnf.d/mariadb-server.cnf
# bind-address=0.0.0.0
```

```sql
-- 계정 생성 + 권한 + 반영
CREATE USER 'user1'@'%' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON *.* TO 'user1'@'%';
FLUSH PRIVILEGES;

-- 계정 확인
SELECT User, Host FROM mysql.user;
SHOW GRANTS FOR 'user1'@'%';
```

---

## 2. DDL 문법 (Configuration)

### 1. 데이터베이스

```sql
CREATE DATABASE mydb;
DROP DATABASE mydb;
USE mydb;
SHOW DATABASES;
SELECT DATABASE();
```

### 2. 테이블 생성 (CREATE TABLE)

```sql
-- 기본 형식
CREATE TABLE 테이블명 (
    컬럼명  자료형  제약조건,
    ...
    CONSTRAINT pk_이름 PRIMARY KEY (컬럼명),
    CONSTRAINT fk_이름 FOREIGN KEY (컬럼명) REFERENCES 부모테이블(컬럼)
);

-- 예제: dept
CREATE TABLE dept (
    deptno  INT(2)       NOT NULL,
    dname   VARCHAR(14),
    loc     VARCHAR(13),
    CONSTRAINT pk_dept PRIMARY KEY (deptno)
);

-- 예제: emp (FK 포함)
CREATE TABLE emp (
    empno    INT(4)        NOT NULL,
    ename    VARCHAR(10),
    job      VARCHAR(9),
    mgr      INT(4),
    hiredate DATE,
    sal      DECIMAL(7,2),
    comm     DECIMAL(7,2),
    deptno   INT(2),
    CONSTRAINT pk_emp     PRIMARY KEY (empno),
    CONSTRAINT fk_deptno  FOREIGN KEY (deptno) REFERENCES dept(deptno)
);
```

### 3. 테이블 삭제 · 데이터 초기화

```sql
DROP TABLE 테이블명;              -- 구조 + 데이터 완전 삭제 (롤백 불가)
TRUNCATE TABLE 테이블명;          -- 구조 유지, 데이터 + AUTO_INCREMENT 초기화
SHOW TABLES;
DESC 테이블명;
SHOW CREATE TABLE 테이블명;       -- FK 포함 전체 DDL 확인
```

### 4. 테이블 구조 수정 (ALTER TABLE)

```sql
-- 컬럼 추가
ALTER TABLE 테이블명 ADD 컬럼명 자료형;
ALTER TABLE member ADD phone VARCHAR(20), ADD regdate DATETIME;

-- 컬럼 삭제
ALTER TABLE member DROP COLUMN address;

-- 자료형 수정
ALTER TABLE member MODIFY name VARCHAR(100);

-- 컬럼 이름 + 자료형 변경
ALTER TABLE member CHANGE age user_age INT;

-- 테이블 이름 변경
ALTER TABLE member RENAME TO customer;
```

---

## 3. 자료형 빠른 참조 (Configuration)

### 1. 숫자형

```sql
INT           -- 정수 (±21억). 나이, 수량, 번호
BIGINT        -- 큰 정수 (±9경). 대용량 PK, 로그 ID
FLOAT         -- 소수(근사값). 정밀도 불필요한 경우
DOUBLE        -- 높은 정밀도 소수
DECIMAL(p,s)  -- 정확한 소수. 돈·가격 필수 (p=전체자리, s=소수점)
              -- 예: DECIMAL(7,2) → 12345.67
```

### 2. 문자형

```sql
CHAR(n)       -- 고정 길이 (항상 n바이트). 국가코드, 고정 코드값
VARCHAR(n)    -- 가변 길이 (실무 최다 사용). 이름, 이메일, 주소
TEXT          -- 긴 텍스트. 게시판 본문
LONGTEXT      -- 매우 긴 텍스트. 로그, 대용량 문서
```

### 3. 날짜/시간형

```sql
DATE          -- 날짜만 (YYYY-MM-DD). 생년월일
TIME          -- 시간만 (HH:MM:SS)
DATETIME      -- 날짜+시간 (실무 최다). 가입일시, 주문시각
TIMESTAMP     -- UNIX 시간 기반. 서버 시간대 영향받음. 로그 기록용
BOOLEAN       -- TRUE/FALSE
```

---

## 4. 제약조건 빠른 참조 (Configuration)

```sql
PRIMARY KEY                          -- 중복 불가 + NULL 불가, 테이블당 1개
id INT AUTO_INCREMENT PRIMARY KEY    -- 자동 증가 + PK

UNIQUE                               -- 중복 불가, NULL 허용, 여러 개 가능
email VARCHAR(100) UNIQUE

NOT NULL                             -- NULL 입력 금지
pw VARCHAR(20) NOT NULL

DEFAULT                              -- 기본값
age INT DEFAULT 18
status VARCHAR(10) DEFAULT 'active'
join_date DATETIME DEFAULT NOW()
join_date DATETIME DEFAULT CURRENT_TIMESTAMP

CHECK                                -- 범위 제한 (MariaDB 일부 버전 비강제)
age INT CHECK (age >= 18 AND age <= 65)

FOREIGN KEY                          -- 외래키 (참조 무결성)
FOREIGN KEY (deptno) REFERENCES dept(deptno)

-- ON DELETE 옵션
FOREIGN KEY (deptno) REFERENCES dept(deptno)             -- 기본: 자식 있으면 부모 삭제 차단
FOREIGN KEY (deptno) REFERENCES dept(deptno) ON DELETE CASCADE    -- 부모 삭제 시 자식도 삭제
FOREIGN KEY (deptno) REFERENCES dept(deptno) ON DELETE SET NULL   -- 부모 삭제 시 FK 컬럼 NULL

-- CONSTRAINT 이름 명시 (권장)
CONSTRAINT pk_emp     PRIMARY KEY (empno)
CONSTRAINT fk_deptno  FOREIGN KEY (deptno) REFERENCES dept(deptno)

-- FK 확인
SHOW CREATE TABLE emp;
SELECT TABLE_NAME, CONSTRAINT_NAME, CONSTRAINT_TYPE
FROM information_schema.TABLE_CONSTRAINTS WHERE TABLE_SCHEMA = DATABASE();
```

---

## 5. DML - INSERT · UPDATE · DELETE (Configuration)

```sql
-- INSERT: 컬럼 명시 (권장)
INSERT INTO 테이블 (컬럼1, 컬럼2) VALUES (값1, 값2);

-- INSERT: 전체 컬럼 순서대로
INSERT INTO dept VALUES (10, 'ACCOUNTING', 'NEW YORK');

-- INSERT: 날짜 연산
INSERT INTO emp VALUES (7788, 'SCOTT', 'ANALYST', 7566,
    DATE_SUB('1987-07-13', INTERVAL 85 DAY), 3000, NULL, 20);

-- UPDATE: 반드시 WHERE 지정
UPDATE member SET smartPhone = '010-2222-3333' WHERE member_id = 3;
UPDATE member SET status = 'YOUNG' WHERE age < 35;

-- DELETE: 반드시 WHERE 지정
DELETE FROM member WHERE member_id = 1;
DELETE FROM member WHERE status = 'INACTIVE';

-- 안전 설정 (WHERE 없는 UPDATE/DELETE 차단)
SET sql_safe_updates = 1;
```

---

## 6. DML - SELECT (Configuration)

### 1. 기본 SELECT + 산술 + 별칭

```sql
SELECT * FROM emp;
SELECT empno, ename, sal FROM emp;

-- 산술 연산 (원본 불변, 출력 시에만 계산)
SELECT sal, sal * 12, (sal * 12) * 0.7 FROM emp;

-- 별칭 (AS)
SELECT sal AS '월급', sal * 12 AS '연봉', (sal * 12) * 0.7 AS '실수령액' FROM emp;

-- 문자열 연결 (CONCAT)
SELECT CONCAT('Hello', 'MariaDB');              -- HelloMariaDB
SELECT ename, CONCAT(ename, '_USA') FROM emp;
SELECT ename, CONCAT(ename, ' 사원') FROM emp;
SELECT CONCAT(IFNULL(comm, 0), '원') FROM emp;  -- NULL 처리 포함
```

### 2. WHERE 조건

```sql
-- 비교 연산자
WHERE sal >= 3000
WHERE sal >= 2000 AND sal < 3000
WHERE job != 'SALESMAN'
WHERE job <> 'SALESMAN'      -- != 와 동일

-- NOT
WHERE NOT (sal >= 2000 AND sal < 3000)

-- OR / AND 혼용 (괄호 필수)
WHERE status = 'ACTIVE' OR age >= 35
WHERE (age >= 30 AND age < 35) OR status = 'INACTIVE'

-- 날짜 비교
WHERE hiredate > '1981-12-31'

-- NULL 비교 (= NULL 불가)
WHERE comm IS NULL
WHERE comm IS NOT NULL
```

### 3. ORDER BY 정렬

```sql
ORDER BY sal ASC                  -- 오름차순 (기본값, 생략 가능)
ORDER BY sal DESC                 -- 내림차순
ORDER BY deptno ASC, sal DESC     -- 다중 정렬
ORDER BY 연봉 ASC                  -- 별칭으로 참조 (따옴표 없이)
ORDER BY 1                        -- 컬럼 번호로 참조
```

### 4. LIKE 패턴 검색

```sql
WHERE ename LIKE 'S%'           -- S로 시작
WHERE ename LIKE '%S'           -- S로 끝남
WHERE ename LIKE '%S%'          -- S 포함
WHERE ename LIKE '%SC%'         -- SC 포함
WHERE ename LIKE '_C%'          -- 두 번째 글자가 C
WHERE ename LIKE '__C%'         -- 세 번째 글자가 C
WHERE brand LIKE '___'          -- 정확히 3글자
WHERE ename NOT LIKE 'S%'       -- S로 시작 안 함
WHERE ename NOT LIKE '%S%'      -- S 미포함

-- OR + LIKE
WHERE ename LIKE 'A%' OR ename LIKE '%N'

-- 대소문자 구분 없이
WHERE LOWER(ename) LIKE 's%'

-- 복합 조건
WHERE (description LIKE '%office%' OR description LIKE '%business%')
  AND product_name NOT LIKE 'apple%'
ORDER BY product_name ASC
```

### 5. INNER JOIN

```sql
-- 기본 INNER JOIN
SELECT c.name, o.product, o.amount
FROM customer AS c
INNER JOIN orders AS o ON c.customer_id = o.customer_id;

-- WHERE + ORDER BY 결합
SELECT c.name, o.product, o.amount
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'PAID'
ORDER BY o.amount DESC;

-- LIKE + JOIN
SELECT c.name, o.product
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.product LIKE '%노트북%';

-- BETWEEN + JOIN
SELECT c.name, c.age, o.product, o.amount
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.age BETWEEN 30 AND 39;

-- IN + JOIN (OR 대신)
SELECT c.name, c.city, o.product
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.city IN ('부산', '대구')
ORDER BY c.city ASC, o.order_date DESC;

-- JOIN + GROUP BY + HAVING
SELECT c.customer_id, c.name, c.city,
       COUNT(o.order_id) AS order_count,
       SUM(o.amount)     AS order_sum
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city
HAVING COUNT(o.order_id) >= 2
ORDER BY order_count DESC, order_sum DESC;
```

### 6. GROUP BY · HAVING · 집계함수

```sql
-- 집계함수 5종
SELECT COUNT(*), SUM(sal), AVG(sal), MAX(sal), MIN(sal) FROM emp;

-- GROUP BY 기본
SELECT deptno, COUNT(*), SUM(sal), AVG(sal)
FROM emp
GROUP BY deptno
ORDER BY deptno;

-- GROUP BY + WHERE (행 필터 먼저 → 그룹화)
SELECT deptno, AVG(sal) AS sal_avg
FROM emp
WHERE sal >= 1000
GROUP BY deptno;

-- HAVING (그룹 조건 — 집계함수 사용 가능)
SELECT deptno, AVG(sal)
FROM emp
GROUP BY deptno
HAVING AVG(sal) >= 2000;

-- WHERE + GROUP BY + HAVING 3단 조합
SELECT deptno, AVG(sal) AS avg_sal
FROM emp
WHERE sal >= 1500
GROUP BY deptno
HAVING AVG(sal) >= 2500;

-- HAVING 복합 조건 (AND)
SELECT job, SUM(sal) AS total_sal, COUNT(*) AS job_cnt
FROM emp
GROUP BY job
HAVING SUM(sal) >= 5000 AND COUNT(*) >= 2;

-- HAVING 서브쿼리 (전체 평균보다 높은 부서)
SELECT deptno, AVG(sal) AS avg_sal
FROM emp
GROUP BY deptno
HAVING AVG(sal) > (SELECT AVG(sal) FROM emp);
```

---

## 7. DCL 문법 (Configuration)

```sql
-- 권한 부여
GRANT ALL PRIVILEGES ON *.* TO 'user1'@'%';
GRANT SELECT, INSERT ON mydb.* TO 'user2'@'localhost';

-- 권한 회수
REVOKE ALL PRIVILEGES ON *.* FROM 'user1'@'%';

-- 즉시 반영
FLUSH PRIVILEGES;
```

---

## 8. 빠른 조회표 (Configuration)

### 1. SQL 명령어 종류 요약

| DDL | DML | DCL |
|---|---|---|
| CREATE / DROP | SELECT | GRANT |
| ALTER / RENAME | INSERT | REVOKE |
| TRUNCATE | UPDATE / DELETE | FLUSH PRIVILEGES |

### 2. NULL 관련 비교

| 문법 | 결과 |
|---|---|
| `WHERE 컬럼 = NULL` | 항상 0건 (잘못된 방법) |
| `WHERE 컬럼 IS NULL` | NULL인 행만 조회  |
| `WHERE 컬럼 IS NOT NULL` | NULL이 아닌 행만 조회  |
| `IFNULL(컬럼, 기본값)` | NULL이면 기본값으로 대체 |
| `CONCAT(NULL, '문자')` | NULL 반환 → IFNULL 사용 |

### 3. LIKE 와일드카드

| 와일드카드 | 의미 | 예 |
|---|---|---|
| `%` | 0개 이상 임의 문자 | `'S%'` → S로 시작 |
| `_` | 정확히 1개 임의 문자 | `'_C%'` → 두 번째가 C |
| `NOT LIKE` | 패턴에 해당하지 않음 | `NOT LIKE 'S%'` |

### 4. ORDER BY 정렬 키워드

| 키워드 | 의미 |
|---|---|
| `ASC` | 오름차순 (기본값, 생략 가능) |
| `DESC` | 내림차순 |
| `,` | 다중 정렬 구분자 |

### 5. 혼동하기 쉬운 항목

| 비슷한 문법 | 차이 |
|---|---|
| `DROP TABLE` vs `TRUNCATE TABLE` | 구조+데이터 삭제 vs 데이터만 삭제(구조 유지, AUTO_INCREMENT 초기화) |
| `DELETE FROM` vs `TRUNCATE TABLE` | 조건 삭제 가능·롤백 가능 vs 전체·롤백 불가 |
| `CHAR(n)` vs `VARCHAR(n)` | 고정 길이 vs 가변 길이 |
| `FLOAT` vs `DECIMAL` | 근사값 vs 정확한 소수(금액에 필수) |
| `DATETIME` vs `TIMESTAMP` | 시간대 무관 vs 서버 시간대 영향 |
| `=` vs `IS NULL` | 값 비교 vs NULL 비교 |
| `AND` vs `OR` | 모두 만족 vs 하나라도 만족 (AND 우선순위 높음) |
| `AS '월급'` vs `AS 월급` | ORDER BY 시 리터럴 vs 별칭 참조 |
| `GRANT` vs `REVOKE` | 권한 부여 vs 권한 회수 |
| `COMMIT` vs `ROLLBACK` | 변경 확정 vs 변경 취소 |
| `WHERE AVG(sal)` vs `HAVING AVG(sal)` | WHERE에 집계함수 불가(오류) vs HAVING에 가능  |
| `PRIMARY KEY` vs `UNIQUE` | NULL불가·1개 vs NULL허용·여러개 |
| `ON DELETE CASCADE` vs (기본) | 부모 삭제 시 자식 함께 삭제 vs 부모 삭제 차단 |
| `COUNT(*)` vs `COUNT(컬럼)` | NULL 포함 전체 건수 vs NULL 제외 건수 |

### 6. GROUP BY 처리 순서

| 순서 | 절 | 집계함수 사용 |
|---|---|---|
| 1 | `FROM` | — |
| 2 | `WHERE` |  불가 |
| 3 | `GROUP BY` | — |
| 4 | `HAVING` |  가능 |
| 5 | `SELECT` |  가능 |
| 6 | `ORDER BY` |  가능 |

### 7. emp · dept 테이블 컬럼 요약

```sql
-- dept
deptno INT(2) PK | dname VARCHAR(14) | loc VARCHAR(13)

-- emp
empno INT(4) PK | ename VARCHAR(10) | job VARCHAR(9)
mgr INT(4)      | hiredate DATE     | sal DECIMAL(7,2)
comm DECIMAL(7,2) | deptno INT(2) FK→dept.deptno

-- 부서 데이터
10-ACCOUNTING-NEW YORK | 20-RESEARCH-DALLAS
30-SALES-CHICAGO       | 40-OPERATIONS-BOSTON

-- 직무 종류 (14명)
PRESIDENT(1): KING | MANAGER(3): BLAKE·CLARK·JONES
ANALYST(2): SCOTT·FORD | SALESMAN(4): ALLEN·WARD·MARTIN·TURNER
CLERK(4): SMITH·ADAMS·JAMES·MILLER

-- emp GROUP BY 집계 결과
deptno 10: COUNT=3, SUM=8750, AVG=2916.67
deptno 20: COUNT=5, SUM=10875, AVG=2175.00
deptno 30: COUNT=6, SUM=9400, AVG=1566.67
전체 AVG: 2073.21

-- customer
customer_id VARCHAR(30) PK | name VARCHAR(50) | gender ENUM('M','F')
age INT | phone VARCHAR(20) | city VARCHAR(50) | join_date DATE
→ 30명, 도시: 서울·부산·대구·인천·광주·대전

-- orders
order_id INT PK | customer_id FK→customer | product VARCHAR(50)
category VARCHAR(50) | amount INT | order_status ENUM('READY','PAID','CANCEL')
order_date DATE
→ 50건, PAID/READY/CANCEL
```

---

## 9. 검증 명령어 모음 (Verification)

```sql
-- 구조 및 데이터 확인
DESC 테이블명;
SHOW CREATE TABLE 테이블명;
SHOW TABLES;
SHOW DATABASES;
SELECT DATABASE();

-- 데이터 샘플 확인
SELECT * FROM emp;
SELECT COUNT(*) FROM emp;
SELECT * FROM emp LIMIT 5;

-- NULL 확인
SELECT * FROM emp WHERE comm IS NULL;
SELECT ename, comm FROM emp WHERE comm IS NOT NULL;

-- 중복 확인
SELECT deptno, COUNT(*) FROM emp GROUP BY deptno;

-- 계정·권한 확인
SELECT User, Host FROM mysql.user;
SHOW GRANTS FOR 'user1'@'%';

-- 서비스·방화벽 (bash)
systemctl status mariadb
firewall-cmd --list-port
firewall-cmd --list-service
```

---

>  **핵심 요약**
> - MariaDB 설치 후: `start → enable → secure_installation → 계정·권한 → 방화벽 → bind-address → restart`
> - DDL은 롤백 불가. DROP/TRUNCATE 전 반드시 백업
> - UPDATE/DELETE는 **항상 WHERE 조건** 지정. 사전 SELECT로 대상 확인
> - NULL 비교: `= NULL`  → `IS NULL` 
> - 금액 저장: `FLOAT`  → `DECIMAL(p,s)` 
> - LIKE: `%`(0개 이상) / `_`(정확히 1개) / 대소문자 → `LOWER()` 활용
> - FK: **부모 테이블 먼저 생성 / 자식 테이블 먼저 삭제** / `ON DELETE CASCADE` = 연쇄 삭제
> - INNER JOIN: `ON A.키 = B.키` 필수. ON 없으면 카티션 곱
> - GROUP BY 처리: FROM→WHERE→GROUP BY→HAVING→SELECT→ORDER BY
> - WHERE에 집계함수 불가 → HAVING / SELECT엔 GROUP BY 컬럼 + 집계함수만
> - 관련: DB - 통합 정리 · DB - 트러블슈팅 치트시트 · DB - 제약조건 (PK·UNIQUE·FK) · DB - INNER JOIN 실습 · DB - GROUP BY·HAVING·집계함수 · DB - 데이터와 데이터베이스 기초 · DB - SQL 문법 (DDL·DML·DCL) · DB - SELECT·WHERE·ORDER BY·LIKE 실습 · emp·dept 테이블 정의 및 데이터
