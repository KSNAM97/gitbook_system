# DB - 트러블슈팅 치트시트

> **Tag:** #Database #MariaDB #SQL #Troubleshooting #CheatSheet
> **핵심 요약:** DB 오류의 대부분은 ① WHERE 절 누락으로 전체 데이터 변경, ② FK 선후관계 위반(부모 없는 자식 삽입·자식 있는 부모 삭제), ③ NULL 비교 오류(`= NULL` 사용), ④ 대소문자 혼동으로 LIKE 결과 없음, ⑤ DDL 실행 후 롤백 불가, ⑥ INNER JOIN ON 절 누락, ⑦ WHERE에 집계함수 사용(HAVING 써야 함), ⑧ UNIQUE/FK 제약조건 중복·참조 오류에서 발생한다. **증상 → 원인 → 조치** 순으로 즉시 대응할 수 있게 정리한 문서다.

---

## 1. 개요 (Overview)

DB 오류를 진단할 때 가장 먼저 확인해야 할 것은 네 가지다. 첫째, 오류 메시지 전문을 확인한다(MariaDB는 정확한 원인을 메시지에 포함한다). 둘째, 대상 테이블 구조를 확인한다(`DESC 테이블명`). 셋째, 실제 데이터를 확인한다(`SELECT COUNT(*)`·`SELECT * ... LIMIT 5`). 넷째, FK 관계를 확인한다(`SHOW CREATE TABLE 테이블명`).

"분명히 맞게 썼는데 결과가 이상하다"는 상황의 90%는 WHERE 절 누락·AND/OR 우선순위 오류, NULL 비교에 `=` 사용, LIKE 대소문자 불일치, FK 선후관계 위반, `=`와 `IS NULL` 혼동이라는 다섯 가지 패턴에서 발생한다.

---

## 2. 증상별 즉시 대응표 (Configuration)

### 1. MariaDB 설치 · 접속

| 증상 | 원인 | 조치 |
|---|---|---|
| `mysql -u root -p` 접속 후 바로 종료 | 비밀번호 오류 또는 서비스 미시작 | `systemctl start mariadb` → 비밀번호 재확인 |
| 외부 Workbench 접속 불가 | `bind-address=127.0.0.1` 또는 방화벽 차단 | `bind-address=0.0.0.0` 설정 + `firewall-cmd --add-port=3306/tcp` |
| `Access denied for user 'user1'` | 계정 미생성 또는 GRANT 누락 | `GRANT ALL PRIVILEGES ON *.* TO 'user1'@'%'; FLUSH PRIVILEGES;` |
| 서비스 재부팅 후 MariaDB 미시작 | `systemctl enable` 미실행 | `systemctl enable --now mariadb` |
| `FLUSH PRIVILEGES` 없이 계정 설정 후 접속 안 됨 | 권한 테이블이 메모리에 반영 안 됨 | `FLUSH PRIVILEGES;` 실행 |

### 2. DDL (CREATE · ALTER · DROP · TRUNCATE)

| 증상 | 원인 | 조치 |
|---|---|---|
| `CREATE TABLE emp` 시 FK 오류 | 참조 대상인 `dept` 테이블 미존재 | `dept` 테이블 먼저 생성 |
| `DROP TABLE dept` 시 FK 오류 | `emp` 가 `dept.deptno` 를 FK로 참조 중 | `emp` 테이블 먼저 DROP 또는 데이터 삭제 후 dept DROP |
| `ALTER TABLE MODIFY` 후 데이터 손실 | 기존 값이 새 자료형과 호환 안 됨 | 변경 전 `SELECT DISTINCT 컬럼 FROM 테이블;` 로 값 확인 |
| `DROP TABLE` 실수로 실행 | DDL은 롤백 불가 | 사전 백업 필수. 운영 DB에서는 DROP 권한 분리 |
| `TRUNCATE` 후 AUTO_INCREMENT 초기화됨 | TRUNCATE는 AUTO_INCREMENT도 1로 초기화 | 의도치 않은 경우 `DELETE FROM 테이블;` 사용 |
| `CHECK` 제약조건이 MariaDB에서 동작 안 함 | MariaDB 일부 버전에서 CHECK 비강제 | 애플리케이션 레이어에서 유효성 검사 로직 추가 |

### 3. DML (SELECT · INSERT · UPDATE · DELETE)

| 증상 | 원인 | 조치 |
|---|---|---|
| `UPDATE member SET status='X';` 전체 변경 | WHERE 절 누락 | **반드시 WHERE 조건 지정**. 사전에 SELECT로 대상 확인 |
| `DELETE FROM member;` 전체 삭제 | WHERE 절 누락 | **반드시 WHERE 조건 지정**. 운영 DB는 `SET sql_safe_updates=1` 설정 |
| `INSERT INTO emp` FK 오류 | deptno 값이 dept 테이블에 없음 | dept에 먼저 해당 deptno 삽입 후 emp에 INSERT |
| `SELECT COUNT(*)` 결과가 예상보다 적음 | WHERE 조건이 NULL 행을 제외 | `IS NULL` / `IS NOT NULL` 조건 명시 확인 |
| `WHERE comm = NULL` 결과 없음 | NULL 비교에 `=` 사용 불가 | `WHERE comm IS NULL` 로 변경 |
| `AND`/`OR` 혼용 결과가 예상과 다름 | AND가 OR보다 우선순위 높음 | 괄호로 명시: `(조건1 AND 조건2) OR 조건3` |
| `SELECT sal * 12` 원본 데이터가 바뀐 줄 앎 | SELECT는 조회만, 원본 불변 | 실제 변경은 `UPDATE` 사용 |

### 4. WHERE · ORDER BY

| 증상 | 원인 | 조치 |
|---|---|---|
| `ORDER BY '월급'` 정렬 안 됨 | 문자열 리터럴로 인식 | 따옴표 없이 `ORDER BY 월급` 또는 컬럼 번호 `ORDER BY 1` |
| 다중 정렬이 의도와 다름 | ASC/DESC 기본값 혼동 | 각 컬럼마다 명시: `ORDER BY col1 ASC, col2 DESC` |
| `WHERE hiredate = '1981-12'` 결과 없음 | DATE 타입에 부분 날짜 문자열 비교 | `WHERE hiredate LIKE '1981-12%'` 또는 `BETWEEN '1981-12-01' AND '1981-12-31'` |
| `!=` 사용 시 NULL 행이 결과에 포함 안 됨 | `!=` 는 NULL을 false로 취급 | `WHERE 컬럼 != 'X' OR 컬럼 IS NULL` |

### 5. LIKE · CONCAT

| 증상 | 원인 | 조치 |
|---|---|---|
| `LIKE 'samsung%'` 결과 없음 | 실제 저장값이 `Samsung`(대문자) | `LOWER(컬럼) LIKE 'samsung%'` 또는 대소문자 맞추기 |
| `LIKE '%SC%'` 가 SCOTT를 찾지 못함 | 대소문자 구분 collation | `LIKE '%sc%'` 또는 `LOWER(ename) LIKE '%sc%'` |
| `LIKE '___'` 로 3글자 브랜드가 안 잡힘 | `_` 개수와 실제 글자 수 불일치 | `LENGTH(컬럼) = 3` 병행 사용 확인 |
| `CONCAT(ename, '사원')` 공백 없이 붙음 | CONCAT 인자에 공백 미포함 | `CONCAT(ename, ' 사원')` — 공백을 문자열로 추가 |
| `CONCAT(NULL, '사원')` 결과가 NULL | NULL이 포함되면 CONCAT 전체가 NULL | `CONCAT(IFNULL(컬럼, ''), '사원')` |

### 6. 자료형 · 제약조건

| 증상 | 원인 | 조치 |
|---|---|---|
| 금액 계산 결과에 오차 발생 | `FLOAT`/`DOUBLE` 로 저장(근사값) | 금액은 반드시 `DECIMAL(7,2)` 사용 |
| `VARCHAR(20)` 컬럼에 21자 입력 시 잘림 | 최대 길이 초과 | `ALTER TABLE MODIFY 컬럼 VARCHAR(100)` 로 길이 확장 |
| `NOT NULL` 컬럼에 INSERT 시 오류 | 해당 컬럼 값 미지정 | INSERT 문에 컬럼명과 값 명시 |
| `PRIMARY KEY` 중복 삽입 오류 | 동일 PK 값 재삽입 | `INSERT IGNORE` 또는 `ON DUPLICATE KEY UPDATE` 사용 |
| `DEFAULT NOW()` 설정했는데 값이 NULL | 컬럼명을 명시하고 값을 직접 NULL로 넣음 | 해당 컬럼을 INSERT 컬럼 목록에서 제외하면 DEFAULT 적용 |

### 7. 제약조건 심화 (PK · UNIQUE · FK)

| 증상 | 원인 | 조치 |
|---|---|---|
| FK INSERT → `Cannot add or update a child row` | 부모 테이블에 참조값 없음 | 부모 테이블에 먼저 행 삽입 후 자식 INSERT |
| 부모 행 DELETE → `Cannot delete or update a parent row` | 자식 테이블이 해당 행 참조 중 | 자식 행 먼저 삭제 → 부모 행 삭제. 또는 FK에 `ON DELETE CASCADE` 옵션 |
| UNIQUE 컬럼에 중복 INSERT → `Duplicate entry` | 이미 동일한 값 존재 | `SELECT COUNT(*) FROM 테이블 WHERE 컬럼 = '값';` 으로 사전 확인 |
| `SHOW CREATE TABLE`에서 FK 이름 모름 | CONSTRAINT 이름 미지정 시 자동 부여 | `SHOW CREATE TABLE 테이블명;` → CONSTRAINT 이름 확인 후 `ALTER TABLE DROP FOREIGN KEY` |
| 자식 테이블 DROP 시 FK 오류 | 다른 테이블에서 자식을 또 참조 중 | `SHOW CREATE TABLE` 로 참조 체인 전체 확인 |
| FK 설정 후 `ON DELETE CASCADE` 동작 안 함 | FK 생성 당시 옵션 미지정 | 테이블 재생성 또는 `ALTER TABLE DROP FK → ADD FK WITH CASCADE` |

### 8. INNER JOIN

| 증상 | 원인 | 조치 |
|---|---|---|
| INNER JOIN 결과 행이 폭증 (카티션 곱) | ON 절 없이 JOIN | 반드시 `ON A.키 = B.키` 지정 |
| `Ambiguous column 'customer_id'` 오류 | 두 테이블에 같은 이름 컬럼 → 어느 테이블인지 불명확 | `c.customer_id`, `o.customer_id` 처럼 테이블 별칭으로 명확히 지정 |
| JOIN 결과가 예상보다 적음 (행 누락) | 한쪽 테이블에만 있는 행은 INNER JOIN에서 제외 | 양쪽에 모두 있는지 확인. 누락 행도 포함하려면 LEFT JOIN 사용 |
| 여러 JOIN 조건이 필요한데 결과 이상 | AND/OR 조건을 ON 절과 WHERE 절에 혼용 | JOIN의 연결 조건은 ON, 행 필터는 WHERE로 명확히 분리 |
| 집계 결과가 예상과 다름 (중복 집계) | 1:N 관계에서 N쪽 행이 여러 개 → JOIN 후 COUNT 시 의도치 않게 중복 | `GROUP BY` 기준 컬럼을 정확히 지정 (ID 단위로) |

### 9. GROUP BY · HAVING

| 증상 | 원인 | 조치 |
|---|---|---|
| `WHERE AVG(sal) >= 2000` → `Invalid use of group function` | WHERE는 그룹 이전 단계 → 집계함수 실행 불가 | `HAVING AVG(sal) >= 2000` 으로 변경 |
| `SELECT deptno, ename, COUNT(*)` → 예상 밖 결과 | ename은 GROUP BY에 없는 컬럼 | SELECT에는 GROUP BY 지정 컬럼 + 집계함수만 사용 |
| HAVING이 적용 안 됨 | GROUP BY 없이 HAVING 사용 | HAVING은 반드시 GROUP BY와 함께 사용 |
| `COUNT(*)` vs `COUNT(comm)` 결과 다름 | `COUNT(comm)`: NULL은 카운트 제외 | NULL 포함 전체 건수는 `COUNT(*)`, NULL 제외는 `COUNT(컬럼)` |
| EX9 서브쿼리 결과가 전체 평균과 다름 | GROUP BY 후 부서별 AVG ≠ 전체 AVG | `SELECT AVG(sal) FROM emp;` 로 전체 평균 먼저 확인 (2073.21...) |

---

## 3. 핵심 진단 명령어 모음 (Verification)

```sql
-- 테이블 구조 확인 (컬럼·타입·NULL·기본값·KEY)
DESC 테이블명;

-- FK 포함 전체 CREATE 구문 확인
SHOW CREATE TABLE emp;

-- 전체 데이터 확인 (건수 먼저)
SELECT COUNT(*) FROM 테이블명;
SELECT * FROM 테이블명 LIMIT 10;

-- NULL 행 확인
SELECT * FROM emp WHERE comm IS NULL;
SELECT * FROM emp WHERE comm IS NOT NULL;

-- 중복 값 확인
SELECT 컬럼, COUNT(*) FROM 테이블 GROUP BY 컬럼 HAVING COUNT(*) > 1;

-- 현재 사용 중인 DB 확인
SELECT DATABASE();

-- 계정·권한 확인
SELECT User, Host FROM mysql.user;
SHOW GRANTS FOR 'user1'@'%';

-- 서비스 로그 확인 (Linux)
systemctl status mariadb
journalctl -u mariadb -n 50

-- 방화벽 상태 확인
firewall-cmd --list-port
firewall-cmd --list-service
```

---

## 4. 트러블슈팅 시나리오 상세

### 시나리오 1. WHERE 절 누락으로 전체 데이터 변경

```sql
-- 실수
UPDATE member SET status = 'INACTIVE';   -- WHERE 없음 → 전체 변경

-- 실행 전 반드시 SELECT로 대상 확인
SELECT * FROM member WHERE username = 'user10';
-- 확인 후 UPDATE
UPDATE member SET status = 'INACTIVE' WHERE username = 'user10';
```

**예방:** 운영 DB는 `SET sql_safe_updates = 1;` 설정 → WHERE 없는 UPDATE/DELETE 차단.

---

### 시나리오 2. FK 선후관계 위반

```sql
-- 실수 1: dept 없이 emp 생성
CREATE TABLE emp (..., FOREIGN KEY (deptno) REFERENCES dept(deptno));
-- ERROR: Can't create table ... (foreign key constraint fails)

-- 실수 2: emp가 참조 중인 dept 행 삭제
DELETE FROM dept WHERE deptno = 10;
-- ERROR: Cannot delete or update a parent row: a foreign key constraint fails
```

**해결:**
```sql
-- 생성: dept 먼저 → emp 나중에
-- 삭제: emp에서 해당 행 먼저 삭제 → dept 삭제
DELETE FROM emp WHERE deptno = 10;
DELETE FROM dept WHERE deptno = 10;
```

---

### 시나리오 3. NULL 비교 오류

```sql
-- 잘못된 방법 (항상 0건 반환)
SELECT * FROM emp WHERE comm = NULL;

-- 올바른 방법
SELECT * FROM emp WHERE comm IS NULL;
SELECT * FROM emp WHERE comm IS NOT NULL;

-- != 조건에서 NULL 행이 제외되는 문제
SELECT * FROM emp WHERE job != 'SALESMAN';   -- comm=NULL 행이 함께 제외됨
-- NULL 포함하려면:
SELECT * FROM emp WHERE job != 'SALESMAN' OR job IS NULL;
```

---

### 시나리오 4. AND/OR 우선순위 오류

```sql
-- 의도: (나이 30~35세) 이거나 (INACTIVE 상태)
-- 잘못된 쿼리
SELECT * FROM member WHERE age >= 30 AND age < 35 OR status = 'INACTIVE';
-- 실제 해석: (age >= 30 AND age < 35) OR status = 'INACTIVE' → 우연히 맞지만...

-- 의도가 다를 경우를 대비해 항상 괄호로 명확히
SELECT * FROM member WHERE (age >= 30 AND age < 35) OR status = 'INACTIVE';
```

---

### 시나리오 5. LIKE 대소문자로 결과 없음

```sql
-- 실제 저장: 'Samsung'
SELECT * FROM product_catalog WHERE product_name LIKE 'samsung%';
-- 결과 0건 (collation에 따라 다름)

-- 해결: LOWER() 로 변환 후 비교
SELECT * FROM product_catalog WHERE LOWER(product_name) LIKE 'samsung%';

-- 또는 대소문자 맞춰서 입력
SELECT * FROM product_catalog WHERE product_name LIKE 'Samsung%';
```

---

### 시나리오 6. CONCAT에 NULL 포함 시 전체 NULL 반환

```sql
-- 문제
SELECT CONCAT(comm, '원') FROM emp;
-- comm이 NULL인 행 → 결과도 NULL

-- 해결: IFNULL로 NULL 대체값 지정
SELECT CONCAT(IFNULL(comm, 0), '원') FROM emp;
```

---

### 시나리오 7. ORDER BY 별칭 따옴표로 정렬 안 됨

```sql
-- 잘못된 방법
SELECT sal * 12 AS '연봉' FROM emp ORDER BY '연봉';    -- 리터럴로 인식, 정렬 안 됨

-- 올바른 방법
SELECT sal * 12 AS 연봉 FROM emp ORDER BY 연봉;        -- 따옴표 없이
SELECT sal * 12 AS 연봉 FROM emp ORDER BY sal * 12;    -- 식 직접 사용
SELECT sal * 12 AS 연봉 FROM emp ORDER BY 1;           -- 컬럼 번호 사용
```

---

### 시나리오 8. INNER JOIN ON 절 누락 (카티션 곱)

```sql
-- 실수: ON 절 없음 → 30 × 50 = 1500행 반환
SELECT c.name, o.product
FROM customer c
INNER JOIN orders o;

-- 해결: ON 절 반드시 지정
SELECT c.name, o.product
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id;
```

---

### 시나리오 9. WHERE에 집계함수 사용 오류

```sql
-- 실수
SELECT deptno, AVG(sal)
FROM emp
WHERE AVG(sal) >= 2000   --  Invalid use of group function
GROUP BY deptno;

-- 해결: 집계함수 조건은 HAVING
SELECT deptno, AVG(sal)
FROM emp
GROUP BY deptno
HAVING AVG(sal) >= 2000;  -- 
```

---

### 시나리오 10. FK INSERT 오류 (부모에 없는 값)

```sql
-- 실수: dept에 99번 부서가 없는데 emp에 INSERT
INSERT INTO emp VALUES(9999, 'TEST', 'CLERK', NULL, NOW(), 1000, NULL, 99);
-- ERROR: Cannot add or update a child row: a foreign key constraint fails

-- 해결 방법 1: 부모 먼저 삽입
INSERT INTO dept VALUES(99, 'TEMP', 'SEOUL');
INSERT INTO emp VALUES(9999, 'TEST', 'CLERK', NULL, NOW(), 1000, NULL, 99);

-- 해결 방법 2: 존재하는 deptno 사용
INSERT INTO emp VALUES(9999, 'TEST', 'CLERK', NULL, NOW(), 1000, NULL, 10);
```

---

### 시나리오 11. UNIQUE 중복 INSERT 오류

```sql
-- 실수: username이 이미 존재하는 경우
INSERT INTO member (member_id, username, email, phone)
VALUES (100, 'user01', 'test@test.com', '010-0000-0000');
-- ERROR: Duplicate entry 'user01' for key 'username'

-- 해결 방법 1: 사전 확인
SELECT COUNT(*) FROM member WHERE username = 'user01';

-- 해결 방법 2: INSERT IGNORE (중복 행 건너뜀)
INSERT IGNORE INTO member (member_id, username, email, phone)
VALUES (100, 'user01', 'test@test.com', '010-0000-0000');

-- 해결 방법 3: ON DUPLICATE KEY UPDATE (중복 시 업데이트)
INSERT INTO member (member_id, username, email, phone)
VALUES (100, 'user01', 'test@test.com', '010-0000-0000')
ON DUPLICATE KEY UPDATE phone = '010-0000-0000';
```

---

>  **핵심 요약**
> - WHERE 없는 UPDATE/DELETE = 전체 변경 → **SELECT로 대상 먼저 확인**
> - FK 위반 → 생성은 **부모 먼저**, 삭제는 **자식 먼저**
> - NULL 비교는 `= NULL` 불가 → 반드시 `IS NULL` / `IS NOT NULL`
> - LIKE 대소문자 문제 → `LOWER(컬럼) LIKE '소문자패턴%'`
> - CONCAT + NULL → `IFNULL(컬럼, 대체값)` 으로 NULL 처리
> - ORDER BY 별칭은 따옴표 없이 사용
> - INNER JOIN ON 절 없으면 카티션 곱 → 반드시 `ON A.키 = B.키`
> - WHERE에 집계함수 불가 → `HAVING` 으로 변경
> - FK INSERT 오류 → 부모 먼저 삽입 / UNIQUE 오류 → INSERT IGNORE 또는 사전 확인
> - 관련: DB - 통합 정리 · DB - SQL 퀵 레퍼런스 · DB - SQL 문법 (DDL·DML·DCL) · DB - SELECT·WHERE·ORDER BY·LIKE 실습 · emp·dept 테이블 정의 및 데이터 · DB - 제약조건 (PK·UNIQUE·FK) · DB - INNER JOIN 실습 · DB - GROUP BY·HAVING·집계함수
