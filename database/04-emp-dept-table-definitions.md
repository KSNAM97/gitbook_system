# 📋 emp · dept 테이블 정의 및 샘플 데이터

> **Tag:** #SQL #emp #dept #샘플데이터 #DDL #INSERT #FOREIGNKEY #PRIMARYKEY #실습테이블
> **핵심 요약:** `dept`(부서)와 `emp`(사원) 테이블은 SQL 실습의 표준 예제 테이블이다. `dept.deptno`를 `emp.deptno`가 외래키로 참조하는 **1:N 관계** 구조이며, 부서 4개·사원 14명 데이터가 포함된다. 이 테이블로 SELECT, WHERE, ORDER BY, JOIN, 집계함수 등 모든 DML 실습을 수행한다.

---

## 1. 📖 개요 (Overview)

`dept` 테이블의 `deptno`(부서 번호)를 `emp` 테이블의 `deptno` 컬럼이 외래키(FOREIGN KEY)로 참조하는 **1:N 관계**다. 부서 1개에 여러 사원이 소속될 수 있다. `FOREIGN KEY (deptno) REFERENCES dept(deptno)`는 emp의 deptno가 반드시 dept 테이블에 존재하는 deptno 값이어야 함을 의미하는 참조 무결성 제약이다. `CONSTRAINT pk_dept PRIMARY KEY (deptno)`처럼 제약조건에 이름(`pk_dept`)을 붙이면 나중에 제약조건을 삭제·수정할 때 이름으로 참조할 수 있다. 사원 `KING`(7839)은 `mgr`(직속 상관) 컬럼이 NULL인데, 이는 최상위 관리자(사장)임을 의미한다. `mgr` 컬럼은 같은 `emp` 테이블의 `empno`를 참조하는 **자기 참조(Self-referencing)** 구조다.

emp 테이블의 각 컬럼은 `empno`(사원번호, PK), `ename`(사원이름), `job`(직무), `mgr`(직속상관의 empno), `hiredate`(입사일), `sal`(급여), `comm`(커미션/보너스), `deptno`(소속부서, FK)를 의미한다. `sal`은 DECIMAL(7,2)로 최대 7자리, 소수점 2자리까지 표현하며 예를 들어 `5000.00`과 같은 값을 가진다. `comm`은 일부 사원만 커미션이 있고 없으면 NULL로 저장되는데, SALESMAN 직무에만 comm 값이 있다. `DATE_SUB('1987-07-13', INTERVAL 85 DAY)`는 SCOTT의 입사일을 날짜 연산으로 계산해 삽입한 것으로 결과는 `1987-04-19`다. 직무(job) 종류는 `PRESIDENT`(1명), `MANAGER`(3명), `ANALYST`(2명), `SALESMAN`(4명), `CLERK`(4명)로 구성된다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** MariaDB/MySQL 계열 RDBMS.

### Step 1. dept 테이블 생성

```sql
/* 부서 정보를 저장하는 테이블 */
CREATE TABLE dept (
    deptno  INT(2)       NOT NULL,           -- 부서 번호(정수 2자리)
    dname   VARCHAR(14),                      -- 부서명 (최대 14글자)
    loc     VARCHAR(13),                      -- 부서 위치 (최대 13글자)
    CONSTRAINT pk_dept PRIMARY KEY (deptno)  -- 기본키 설정(부서 번호)
);
```

### Step 2. emp 테이블 생성

```sql
/* 사원 정보를 저장하는 테이블 */
CREATE TABLE emp (
    empno    INT(4)        NOT NULL,            -- 사원 번호(4자리 정수)
    ename    VARCHAR(10),                        -- 사원 이름(최대 10글자)
    job      VARCHAR(9),                         -- 직무(최대 9글자)
    mgr      INT(4),                             -- 직속 상관(empno 참조)
    hiredate DATE,                               -- 입사일
    sal      DECIMAL(7,2),                       -- 급여(전체 7자리, 소수점 2자리)
    comm     DECIMAL(7,2),                       -- 보너스(전체 7자리, 소수점 2자리)
    deptno   INT(2),                             -- 소속 부서 번호

    CONSTRAINT pk_emp  PRIMARY KEY (empno),      -- 기본키 설정
    CONSTRAINT fk_deptno FOREIGN KEY (deptno)    -- 외래키 설정
        REFERENCES dept(deptno)                  -- dept 테이블 deptno 참조
);
```

> **참고:** ⚠️ **생성 순서:** `dept` 테이블 먼저 생성 → 이후 `emp` 테이블 생성. emp가 dept의 deptno를 외래키로 참조하기 때문에 dept가 먼저 존재해야 한다.

### Step 3. dept 기본 데이터 (INSERT)

```sql
INSERT INTO dept (deptno, dname, loc) VALUES (10, 'ACCOUNTING', 'NEW YORK');
INSERT INTO dept VALUES (20, 'RESEARCH',   'DALLAS');
INSERT INTO dept VALUES (30, 'SALES',      'CHICAGO');
INSERT INTO dept VALUES (40, 'OPERATIONS', 'BOSTON');
```

**부서 데이터 요약**

| deptno | dname      | loc      |
|--------|------------|----------|
| 10     | ACCOUNTING | NEW YORK |
| 20     | RESEARCH   | DALLAS   |
| 30     | SALES      | CHICAGO  |
| 40     | OPERATIONS | BOSTON   |

### Step 4. emp 기본 데이터 (INSERT)

```sql
INSERT INTO emp VALUES (7839, 'KING',   'PRESIDENT', NULL, '1981-11-17', 5000, NULL,   10);
INSERT INTO emp VALUES (7698, 'BLAKE',  'MANAGER',   7839, '1981-05-01', 2850, NULL,   30);
INSERT INTO emp VALUES (7782, 'CLARK',  'MANAGER',   7839, '1981-06-09', 2450, NULL,   10);
INSERT INTO emp VALUES (7566, 'JONES',  'MANAGER',   7839, '1981-04-02', 2975, NULL,   20);

-- SCOTT: 날짜 연산 사용 (1987-07-13 에서 85일 전 = 1987-04-19)
INSERT INTO emp VALUES (7788, 'SCOTT',  'ANALYST',   7566,
    DATE_SUB('1987-07-13', INTERVAL 85 DAY), 3000, NULL, 20);

INSERT INTO emp VALUES (7902, 'FORD',   'ANALYST',   7566, '1981-12-03', 3000, NULL,   20);
INSERT INTO emp VALUES (7369, 'SMITH',  'CLERK',     7902, '1980-12-17',  800, NULL,   20);
INSERT INTO emp VALUES (7499, 'ALLEN',  'SALESMAN',  7698, '1981-02-20', 1600,  300,   30);
INSERT INTO emp VALUES (7521, 'WARD',   'SALESMAN',  7698, '1981-02-22', 1250,  500,   30);
INSERT INTO emp VALUES (7654, 'MARTIN', 'SALESMAN',  7698, '1981-09-28', 1250, 1400,   30);
INSERT INTO emp VALUES (7844, 'TURNER', 'SALESMAN',  7698, '1981-09-08', 1500,    0,   30);

-- ADAMS: 날짜 연산 사용 (1987-07-13 에서 51일 전 = 1987-05-23)
INSERT INTO emp VALUES (7876, 'ADAMS',  'CLERK',     7788,
    DATE_SUB('1987-07-13', INTERVAL 51 DAY), 1100, NULL, 20);

INSERT INTO emp VALUES (7900, 'JAMES',  'CLERK',     7698, '1981-12-03',  950, NULL,   30);
INSERT INTO emp VALUES (7934, 'MILLER', 'CLERK',     7782, '1982-01-23', 1300, NULL,   10);
```

**사원 데이터 요약**

| empno | ename  | job       | mgr  | hiredate   | sal     | comm    | deptno |
|-------|--------|-----------|------|------------|---------|---------|--------|
| 7369  | SMITH  | CLERK     | 7902 | 1980-12-17 | 800.00  |         | 20     |
| 7499  | ALLEN  | SALESMAN  | 7698 | 1981-02-20 | 1600.00 | 300.00  | 30     |
| 7521  | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250.00 | 500.00  | 30     |
| 7566  | JONES  | MANAGER   | 7839 | 1981-04-02 | 2975.00 |         | 20     |
| 7654  | MARTIN | SALESMAN  | 7698 | 1981-09-28 | 1250.00 | 1400.00 | 30     |
| 7698  | BLAKE  | MANAGER   | 7839 | 1981-05-01 | 2850.00 |         | 30     |
| 7782  | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450.00 |         | 10     |
| 7788  | SCOTT  | ANALYST   | 7566 | 1987-04-19 | 3000.00 |         | 20     |
| 7839  | KING   | PRESIDENT | NULL | 1981-11-17 | 5000.00 |         | 10     |
| 7844  | TURNER | SALESMAN  | 7698 | 1981-09-08 | 1500.00 | 0.00    | 30     |
| 7876  | ADAMS  | CLERK     | 7788 | 1987-05-23 | 1100.00 |         | 20     |
| 7900  | JAMES  | CLERK     | 7698 | 1981-12-03 | 950.00  |         | 30     |
| 7902  | FORD   | ANALYST   | 7566 | 1981-12-03 | 3000.00 |         | 20     |
| 7934  | MILLER | CLERK     | 7782 | 1982-01-23 | 1300.00 |         | 10     |

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```sql
-- 테이블 구조 확인
DESC dept;
DESC emp;

-- 전체 데이터 확인
SELECT * FROM dept;
SELECT * FROM emp;

-- 사원 수 확인 (14명)
SELECT COUNT(*) FROM emp;

-- 부서별 사원 수
SELECT deptno, COUNT(*) FROM emp GROUP BY deptno;

-- comm이 NULL인 사원
SELECT ename, comm FROM emp WHERE comm IS NULL;

-- 급여 통계
SELECT MIN(sal), MAX(sal), AVG(sal) FROM emp;
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. emp INSERT 시 외래키 오류

- **증상:** `INSERT INTO emp VALUES (9999, 'TEST', 'CLERK', NULL, NOW(), 1000, NULL, 99);` 실행 시 오류.
- **원인:** deptno 99는 dept 테이블에 존재하지 않아 외래키 제약 위반.
- **해결:**
  ```sql
  -- 1) dept에 먼저 삽입 후 emp 삽입
  INSERT INTO dept VALUES (99, 'NEW_DEPT', 'SEOUL');
  INSERT INTO emp VALUES (9999, 'TEST', 'CLERK', NULL, NOW(), 1000, NULL, 99);

  -- 2) 또는 존재하는 deptno(10/20/30/40) 사용
  INSERT INTO emp VALUES (9999, 'TEST', 'CLERK', NULL, NOW(), 1000, NULL, 10);
  ```

#### 🚨 시나리오 2. dept 테이블 없이 emp 테이블 생성 시 오류

- **증상:** `CREATE TABLE emp (...)` 실행 시 `Can't create table ... foreign key constraint fails` 오류.
- **원인:** FOREIGN KEY `REFERENCES dept(deptno)` 에서 dept 테이블이 아직 존재하지 않음.
- **해결:** 반드시 **dept 먼저 생성 → emp 생성** 순서 준수.

#### 🚨 시나리오 3. dept 레코드 삭제 시 오류

- **증상:** `DELETE FROM dept WHERE deptno = 10;` 실행 시 오류.
- **원인:** deptno=10을 참조하는 emp 행(KING, CLARK, MILLER)이 존재해 참조 무결성 위반.
- **해결:**
  ```sql
  -- 1) emp에서 해당 사원 먼저 삭제 후 dept 삭제
  DELETE FROM emp WHERE deptno = 10;
  DELETE FROM dept WHERE deptno = 10;

  -- 2) 또는 FK에 ON DELETE CASCADE 옵션 추가 (테이블 재생성 필요)
  ```

---

> 📌 **핵심 요약**
> - dept(부서) → emp(사원): **1:N 관계**, emp의 deptno는 FK로 dept.deptno 참조
> - 생성 순서: **dept 먼저, emp 나중에** (FK 참조 대상이 먼저 존재해야 함)
> - 삭제 순서: **emp 먼저, dept 나중에** (참조하는 자식 테이블 먼저 처리)
> - KING만 mgr이 NULL → 최상위 관리자. comm은 SALESMAN만 값이 있음
> - 관련: 🗄️ DB - 데이터와 데이터베이스 기초 · 🔧 DB - SQL 문법 (DDL·DML·DCL) · 🔍 DB - SELECT·WHERE·ORDER BY·LIKE 실습
