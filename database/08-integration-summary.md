# 🧩 DB - 통합 정리 (데이터 기초부터 INNER JOIN·GROUP BY·HAVING까지 한눈에)

> **Tag:** #Database #MariaDB #SQL #Summary #통합정리 #제약조건 #INNERJOIN #GROUPBY #HAVING
> **핵심 요약:** DB는 **데이터 → 정보 가치 창출**을 목적으로 하고, SQL은 **DDL(구조 정의) → DML(데이터 조작) → DCL(권한 제어)** 세 계층으로 나뉜다. 조회 문법은 `SELECT → WHERE → ORDER BY → LIKE → GROUP BY → HAVING → JOIN` 순으로 확장되고, 제약조건(PK·UNIQUE·FK)으로 데이터 무결성을 보장한다. 이 문서는 1~4번·8~10번 문서를 한 장으로 닫는 색인이다.

---

## 1. 📖 개요 (Overview)

DB와 SQL 전체를 관통하는 원리는, 모든 작업이 **"어느 테이블(FROM)에서, 어떤 조건(WHERE)으로, 어떤 컬럼(SELECT)을"**이라는 세 축으로 귀결된다는 것이다. 구조를 먼저 정의(DDL)하고, 그 안의 데이터를 다루며(DML), 접근 권한을 제어(DCL)하는 순서로 계층이 쌓인다. 데이터(raw)를 의미 있는 정보로 변환하는 것이 DB의 본질적 목적이다. DDL은 **실행 즉시 반영, 롤백 불가**이고, DML은 **COMMIT/ROLLBACK**으로 되돌릴 수 있다. SELECT는 원본을 바꾸지 않으며, 산술 연산·별칭(AS)·CONCAT은 **출력 시에만** 적용된다. WHERE 없는 UPDATE/DELETE는 테이블 전체를 변경하므로 **항상 WHERE로 범위를 한정해야 한다.**

10개 문서는 MariaDB 설치(환경) → DDL/DML/DCL(문법) → SELECT·WHERE·ORDER BY·LIKE(조회) → emp·dept(실습 데이터) → 제약조건(PK·UNIQUE·FK) → INNER JOIN(다중 테이블 조회) → GROUP BY·HAVING(집계) 순서로 계층을 형성하며 서로 연결된다. FK(외래키)가 있어야 JOIN이 의미 있으므로 제약조건 → INNER JOIN 순서로 이해하면 된다. GROUP BY는 JOIN 결과에도 적용 가능하다(customer·orders 예제 EX11~EX12). `SELECT`의 전체 내부 처리 순서(GROUP BY 포함)는 `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY` 다.

DDL · DML · DCL은 각각 쓰이는 시점이 다르다. DDL은 **테이블 구조(그릇)를 만들고 바꿀 때**, DML은 **그 안의 데이터(내용)를 추가·조회·수정·삭제할 때**, DCL은 **누가 어떤 DB/테이블에 접근할 수 있는지 권한을 줄 때** 사용한다. DDL의 대표 명령어는 CREATE·ALTER·DROP·TRUNCATE·RENAME이며 롤백이 불가능하고 주로 개발·설계 단계에서 쓰인다. DML의 대표 명령어는 SELECT·INSERT·UPDATE·DELETE이며 롤백이 가능하고 서비스 운영 전반에서 쓰인다. DCL의 대표 명령어는 GRANT·REVOKE·FLUSH PRIVILEGES이며 계정 및 보안 관리에 쓰인다.

제약조건(Constraint) 3종의 핵심은 PK(대표 식별자·중복불가·NULL불가), UNIQUE(중복불가·NULL허용·여러개 가능), FK(부모 테이블 PK 참조·고아 데이터 차단)로 요약된다. `PRIMARY KEY`는 중복과 NULL을 모두 허용하지 않고 테이블당 1개만 가능하며 행의 대표 식별자 역할을 한다. `UNIQUE`는 중복은 허용하지 않지만 NULL은 허용하고 여러 개 설정 가능하며 이메일·아이디 중복 금지에 쓰인다. `FOREIGN KEY`는 중복과 NULL을 모두 허용하고 여러 개 가능하며 타 테이블 참조·관계 표현에 쓰인다. `NOT NULL`은 중복은 허용하지만 NULL은 허용하지 않으며 필수 입력에 쓰이고, `DEFAULT`는 기본값 자동 설정에, `CHECK`는 값 범위 제한에 쓰인다. FK의 선후관계는 **생성은 부모 먼저, 삭제는 자식 먼저**이며, `ON DELETE CASCADE` 옵션을 쓰면 부모 삭제 시 자식 행도 함께 자동 삭제된다.

INNER JOIN은 두 테이블에서 ON 조건이 **일치하는 행만** 결합하며, 한쪽에만 존재하는 행은 제외된다.

```sql
SELECT c.name, o.product, o.amount
FROM customer AS c
INNER JOIN orders AS o ON c.customer_id = o.customer_id
WHERE o.order_status = 'PAID'
ORDER BY o.amount DESC;
```

ON 절이 없으면 카티션 곱(30×50=1500행)이 발생하므로 반드시 `ON A.키 = B.키`를 지정해야 한다. AS로 테이블 별칭을 지정하면 `c.name`, `o.product`처럼 짧게 참조할 수 있다. JOIN 이후 WHERE로 추가 필터링을 하거나 GROUP BY+HAVING으로 집계할 수도 있다(EX11·EX12).

GROUP BY · HAVING의 핵심 원리는, GROUP BY가 동일 값 행들을 그룹으로 묶어 집계함수를 적용하고, HAVING이 그룹이 만들어진 **후** 집계 조건을 건다는 점이다(WHERE는 그룹 이전 행 조건). WHERE는 그룹 이전 단계에서 집계함수를 사용할 수 없이 개별 행을 필터링하는 반면, HAVING은 그룹 이후 단계에서 집계함수를 사용할 수 있게 그룹을 필터링한다. 집계함수 5종은 `SUM(합계)` · `AVG(평균)` · `MAX(최대)` · `MIN(최소)` · `COUNT(개수)`이며, `WHERE + GROUP BY + HAVING` 3단 조합도 가능하다. HAVING 서브쿼리의 예로는 `HAVING AVG(sal) > (SELECT AVG(sal) FROM emp)`가 있다.

---

## 2. 🛠️ 표준 개념 정리 (Configuration)

> **적용 환경:** MariaDB/MySQL 계열 RDBMS.

### Step 1. 10개 문서 요약 흐름도

```text
[1] 데이터와 DB 기초   → 데이터≠정보, DB 필요성, MariaDB 설치·보안 설정·계정·방화벽
[2] SQL 문법           → DDL · DML · DCL + 자료형 + 제약조건
                        + ALTER TABLE member 실습 7단계 (EX1~EX7, DESC 결과 확인)
                        + member INSERT 20건 + SELECT/UPDATE/DELETE 실습 (EX8~EX24)
[3] SELECT 실습        → SELECT · WHERE · ORDER BY · LIKE · CONCAT · AS · 산술 연산
                        + product_catalog 테이블 (26종 상품) LIKE 실습 (EX1~EX18)
                          EX15: product_code 세번째 -  EX16: 브랜드 정확히 3글자
                          EX17: 브랜드 두번째 p        EX18: office/business 복합조건
[4] emp·dept 테이블    → 1:N FK 관계 · 샘플 데이터 14행 · 실습 기반 테이블
[8] 제약조건 심화      → PK·UNIQUE·FK 개념·비교표·예제·ON DELETE CASCADE
[9] INNER JOIN 실습    → customer(30명)·orders(50건) DDL+INSERT + EX1~EX12
                          EX11: 도시별 COUNT+SUM 집계  EX12: HAVING COUNT>=2 필터
[10] GROUP BY·HAVING   → 집계함수 5종 + GROUP BY EX1~EX6(emp) + HAVING EX1~EX9(emp)
                          EX9: HAVING AVG(sal) > (SELECT AVG(sal) FROM emp) 서브쿼리
[5] 통합 정리 (이 문서)
[6] 트러블슈팅 치트시트
[7] SQL 퀵 레퍼런스
```

### Step 2. SQL 명령어 계층 한눈에 보기

| DDL | DML | DCL |
|---|---|---|
| CREATE DATABASE | SELECT | GRANT |
| CREATE TABLE | INSERT | REVOKE |
| ALTER TABLE | UPDATE | FLUSH PRIVILEGES |
| DROP TABLE | DELETE | — |
| TRUNCATE TABLE | — | — |
| RENAME | — | — |

### Step 3. SELECT 전체 확장 흐름

```sql
-- [기본] SELECT
SELECT 컬럼 FROM 테이블;

-- [필터] WHERE
SELECT * FROM emp WHERE sal >= 3000;

-- [정렬] ORDER BY
SELECT * FROM emp ORDER BY sal DESC;

-- [패턴] LIKE
SELECT * FROM emp WHERE ename LIKE 'S%';

-- [연산+별칭] AS
SELECT sal AS '월급', sal * 12 AS '연봉', (sal * 12) * 0.7 AS '실수령액' FROM emp;

-- [문자연결] CONCAT
SELECT CONCAT(ename, ' 사원') FROM emp;

-- [그룹집계] GROUP BY + 집계함수
SELECT deptno, COUNT(*), SUM(sal), AVG(sal)
FROM emp
GROUP BY deptno
ORDER BY deptno;

-- [그룹필터] HAVING
SELECT deptno, AVG(sal)
FROM emp
GROUP BY deptno
HAVING AVG(sal) >= 2000;

-- [JOIN] INNER JOIN
SELECT c.name, o.product, o.amount
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'PAID';

-- [JOIN + GROUP BY + HAVING 조합]
SELECT c.customer_id, c.name, COUNT(o.order_id) AS order_count, SUM(o.amount) AS order_sum
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city
HAVING COUNT(o.order_id) >= 2
ORDER BY order_count DESC;
```

### Step 4. MariaDB 설치~운영 흐름 한눈에

```text
① dnf install -y mariadb-server
② systemctl start mariadb && systemctl enable mariadb
③ mysql_secure_installation   (root 비밀번호·익명 사용자·test DB 설정)
④ mysql -u root -p            (접속)
   CREATE USER 'user1'@'%' IDENTIFIED BY '1234';
   GRANT ALL PRIVILEGES ON *.* TO 'user1'@'%';
   FLUSH PRIVILEGES;
⑤ 방화벽: firewall-cmd --permanent --add-port=3306/tcp && --add-service=mysql && --reload
⑥ bind-address=0.0.0.0  (외부 접속 허용, /etc/my.cnf.d/mariadb-server.cnf)
⑦ systemctl restart mariadb
⑧ Workbench 설치 후 연결 테스트
```

### Step 5. 자료형 · 제약조건 핵심 선택 기준

| 저장 데이터 | 자료형 | 제약조건 |
|---|---|---|
| 돈·가격 | `DECIMAL(p,s)` | `NOT NULL` |
| 일반 문자 | `VARCHAR(n)` | `NOT NULL` or `UNIQUE` |
| 날짜+시간 | `DATETIME` | `DEFAULT NOW()` |
| 고유 번호 | `INT` | `AUTO_INCREMENT PRIMARY KEY` |
| 고정 길이 코드 | `CHAR(n)` | — |
| 긴 텍스트 | `TEXT` | — |

### Step 6. emp · dept 관계 요약

```text
dept (1)  ──────── (N)  emp
 deptno  ◀──FK──── deptno
 dname              empno  ← PK
 loc                ename
                    mgr  ──┐ (자기 참조)
                    sal    └──▶ empno
                    comm  (NULL = 커미션 없음)
                    hiredate
```

- 부서 4개 (10·20·30·40), 사원 14명
- 직무: PRESIDENT(1) · MANAGER(3) · ANALYST(2) · SALESMAN(4) · CLERK(4)
- `comm` 값이 있는 사원: SALESMAN 4명만 (ALLEN·WARD·MARTIN·TURNER)

### Step 7. customer · orders 관계 요약

```text
customer (1)  ──────── (N)  orders
 customer_id  ◀──FK──── customer_id
 name                   order_id  ← PK
 gender (M/F ENUM)      product
 age                    category
 city                   amount
 join_date              order_status (READY/PAID/CANCEL ENUM)
                        order_date
```

- customer: 30명 (서울·부산·대구·인천·광주·대전 도시)
- orders: 50건 (PAID·READY·CANCEL 상태)
- 주요 INNER JOIN 패턴: 도시별 집계 (EX11), 2건 이상 주문 고객 (EX12)

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 설치 후 공통 검증 순서

```bash
systemctl status mariadb          # ① 서비스 상태 확인
mysql -u root -p                  # ② root 접속 테스트
```

```sql
SHOW DATABASES;                   -- ③ DB 목록 확인
USE mydb;                         -- ④ DB 선택
SHOW TABLES;                      -- ⑤ 테이블 목록
DESC emp;                         -- ⑥ 테이블 구조 확인
SELECT COUNT(*) FROM emp;         -- ⑦ 데이터 건수 확인 (14건)
SELECT COUNT(*) FROM customer;    -- customer: 30건
SELECT COUNT(*) FROM orders;      -- orders: 50건
```

### 3-2. 대표 함정 요약

| 함정 | 결과 | 정답 |
|---|---|---|
| `DROP TABLE` 실행 | 구조+데이터 완전 삭제, 롤백 불가 | 사전 백업 필수 |
| `UPDATE member SET status='X';` (WHERE 누락) | 전체 행이 변경됨 | 반드시 `WHERE` 조건 지정 |
| emp 보다 dept 먼저 삭제 시도 | FK 제약 오류 | **자식(emp) 먼저 삭제 → 부모(dept) 삭제** |
| emp 생성 전 dept 미생성 | FK 참조 오류로 생성 실패 | **부모(dept) 먼저 생성 → 자식(emp) 생성** |
| `LIKE 'samsung%'` 결과 없음 | 대소문자 불일치 | `LOWER(컬럼) LIKE 'samsung%'` |
| `ORDER BY '월급'` 정렬 안 됨 | 문자열로 인식 | 따옴표 없이 `ORDER BY 월급` |
| `TRUNCATE` 후 AUTO_INCREMENT 그대로 | `DELETE`와 혼동 | `TRUNCATE`는 AUTO_INCREMENT도 초기화됨 |
| `comm IS NULL` vs `comm = NULL` | `= NULL` 은 항상 거짓 | NULL 비교는 반드시 `IS NULL` / `IS NOT NULL` |
| `AND`/`OR` 혼용 우선순위 오류 | 의도와 다른 결과 | 괄호로 우선순위 명확히 지정 |
| `DECIMAL` 대신 `FLOAT` 로 금액 저장 | 근사값 오차 | 금액은 반드시 `DECIMAL(p,s)` |
| ON 절 없이 INNER JOIN | 카티션 곱 발생 (행 폭증) | 반드시 `ON A.키 = B.키` 지정 |
| `WHERE AVG(sal) >= 2000` | 오류 (집계함수 WHERE 불가) | `HAVING AVG(sal) >= 2000` 으로 변경 |
| FK INSERT (부모에 없는 값) | `foreign key constraint fails` | 부모 테이블에 먼저 삽입 |
| UNIQUE 중복 INSERT | `Duplicate entry` 오류 | `SELECT COUNT(*)` 로 사전 확인 |

---

> 📌 **핵심 요약**
> - DB 목적 = **데이터 → 정보 가치 창출** / 단순 저장이 목적이 아님
> - DDL(구조, 롤백 불가) · DML(데이터, COMMIT/ROLLBACK 가능) · DCL(권한)
> - SELECT 전체 처리 순서: `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`
> - WHERE 없는 UPDATE/DELETE = 전체 변경. **반드시 WHERE로 범위 한정**
> - FK 선후관계: **생성은 부모 먼저 / 삭제는 자식 먼저**
> - INNER JOIN: ON 조건 일치 행만 결합. ON 절 필수
> - GROUP BY: 그룹 단위 집계. SELECT엔 GROUP BY 컬럼 + 집계함수만
> - HAVING: 그룹 조건 (집계함수 가능) ↔ WHERE: 행 조건 (집계함수 불가)
> - 관련: DB - 데이터와 데이터베이스 기초 · DB - SQL 문법 (DDL·DML·DCL) · DB - SELECT·WHERE·ORDER BY·LIKE 실습 · emp·dept 테이블 정의 및 데이터 · DB - 제약조건 (PK·UNIQUE·FK) · DB - INNER JOIN 실습 · DB - GROUP BY·HAVING·집계함수 · DB - 트러블슈팅 치트시트 · DB - SQL 퀵 레퍼런스
