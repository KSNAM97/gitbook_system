# 🔑 DB - 제약조건 (PK · UNIQUE · FK)

> **Tag:** #SQL #Constraint #PrimaryKey #UniqueKey #ForeignKey #제약조건 #데이터무결성
> **핵심 요약:** 제약조건은 테이블에 잘못된·중복된·엉뚱한 데이터가 들어가지 않도록 DB가 자동으로 막아주는 안전장치다. `PRIMARY KEY`(대표 식별자), `UNIQUE`(중복 금지), `FOREIGN KEY`(타 테이블 참조)가 핵심 3종이며, 이를 올바르게 설계하면 DB가 데이터 일관성을 스스로 보장한다.

---

## 1. 📖 개요 (Overview)

제약조건(Constraint)은 테이블에 들어가는 데이터를 규칙으로 관리하는 장치다. 잘못된 데이터·중복 데이터·참조 오류가 애플리케이션 레이어를 통과하더라도 DB가 마지막으로 걸러준다. 예를 들어 존재하지 않는 부서번호로 사원을 등록하면 안 되는 것은 **FOREIGN KEY**가 막아주고, 같은 사람을 두 번 저장하면 안 되는 것은 **PRIMARY KEY**/**UNIQUE**가 막아준다. 주민등록번호·학번처럼 절대 중복되면 안 되는 값은 **UNIQUE**로 보장한다. 제약조건 없이 설계하면 애플리케이션 버그 하나로 DB 전체가 오염될 수 있다. 제약조건의 종류에는 `PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`, `NOT NULL`, `CHECK`, `DEFAULT`가 있다.

PRIMARY KEY(기본키)는 테이블에서 각 행(레코드)을 **유일하게 식별**하는 기준 컬럼이다. **중복 불가 + NULL 불가** 두 조건을 동시에 만족해야 하며, 테이블당 딱 하나만 지정 가능하다. 학생 테이블의 `student_no`(학번), 사원 테이블의 `empno`, 상품 테이블의 `product_id`가 대표적인 PK다. 같은 `student_no`로 INSERT하면 즉시 오류가 나고(중복 금지), `student_no`에 NULL을 INSERT해도 즉시 오류가 난다(NULL 금지). `AUTO_INCREMENT`와 자주 함께 사용되며(예: `id INT AUTO_INCREMENT PRIMARY KEY`), PK는 테이블의 **대표 식별자**이자 JOIN의 연결 기준이 된다.

UNIQUE(고유키)와 PRIMARY KEY는 둘 다 중복을 허용하지 않지만, UNIQUE는 **NULL 허용 + 여러 개 설정 가능**하다는 점이 다르다. PRIMARY KEY는 대표 식별자이고, UNIQUE는 "이 값도 중복되면 안 됨"을 보장하는 보조 제약이다. 비교하면 PRIMARY KEY는 NULL을 허용하지 않고 테이블당 1개만 가능하며 테이블의 대표 식별자 역할을 하는 반면, UNIQUE는 NULL을 허용하고 여러 개 설정 가능하며 중복 금지 보조 제약 역할을 한다. UNIQUE의 활용 예로는 이메일(`email UNIQUE`), 로그인 ID(`username UNIQUE`), 주민등록번호, 사업자등록번호가 있다. NULL은 "값 없음"이라 비교가 불가능하므로 여러 개 있어도 괜찮다.

FOREIGN KEY(외래키)는 다른 테이블의 PRIMARY KEY(또는 UNIQUE 키)를 참조하는 컬럼이다. 참조 대상에 없는 값은 삽입할 수 없어 **고아 데이터(잘못된 참조)**를 DB 레벨에서 차단한다. 예를 들어 `emp.deptno`가 `dept.deptno`를 참조할 때 dept에 없는 부서번호로 emp에 INSERT하면 오류가 발생한다. **부모(참조되는 쪽) 테이블을 먼저 생성하고, 자식(참조하는 쪽) 테이블을 나중에 생성**해야 하며, **삭제 순서는 반대로** 자식(emp)을 먼저 삭제한 뒤 부모(dept)를 삭제해야 한다. `ON DELETE CASCADE` 옵션을 사용하면 부모 행 삭제 시 자식 행도 함께 자동 삭제된다. FK는 대부분 INNER JOIN의 연결 기준(`ON A.컬럼 = B.컬럼`)이 되며, dept에 10·20번만 있는데 deptno=30으로 emp에 INSERT하면 FK가 차단하는 식으로 **데이터 일관성을 보장**한다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** MariaDB/MySQL 계열 RDBMS.

### Step 1. PRIMARY KEY 설정 예제

```sql
-- 인라인 방식
CREATE TABLE student (
    student_no  INT          PRIMARY KEY,   -- 기본키 (학번)
    name        VARCHAR(50),
    major       VARCHAR(50)
);

-- CONSTRAINT 이름 명시 방식
CREATE TABLE student (
    student_no  INT          NOT NULL,
    name        VARCHAR(50),
    major       VARCHAR(50),
    CONSTRAINT pk_student PRIMARY KEY (student_no)
);

-- AUTO_INCREMENT 함께 사용
CREATE TABLE product (
    product_id  INT          AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL
);
```

> **참고:** ⚠️ 같은 `student_no`로 INSERT 시도 → `Duplicate entry` 오류
> **참고:** ⚠️ `student_no = NULL` INSERT 시도 → `Column cannot be null` 오류

### Step 2. UNIQUE 설정 예제

```sql
CREATE TABLE member (
    member_id   INT          PRIMARY KEY,
    username    VARCHAR(30)  UNIQUE,    -- 고유키: 아이디 중복 불가
    email       VARCHAR(100) UNIQUE,    -- 고유키: 이메일 중복 불가
    phone       VARCHAR(20)
);
```

- `member_id`: 테이블 대표 식별자 (PK)
- `username`, `email`: 각각 중복 불가 (UNIQUE) — 여러 개 설정 가능

### Step 3. FOREIGN KEY 설정 예제

```sql
-- 부모 테이블 먼저 생성
CREATE TABLE dept (
    deptno  INT PRIMARY KEY,
    dname   VARCHAR(50)
);

-- 자식 테이블 (FK 포함)
CREATE TABLE emp (
    empno   INT PRIMARY KEY,
    ename   VARCHAR(50),
    deptno  INT,
    FOREIGN KEY (deptno) REFERENCES dept(deptno)
);

-- CONSTRAINT 이름 명시 버전 (더 명확)
CREATE TABLE emp (
    empno    INT(4)        NOT NULL,
    ename    VARCHAR(10),
    deptno   INT(2),
    CONSTRAINT pk_emp     PRIMARY KEY (empno),
    CONSTRAINT fk_deptno  FOREIGN KEY (deptno) REFERENCES dept(deptno)
);
```

> **참고:** ⚠️ dept에 없는 deptno로 emp INSERT → `Cannot add or update a child row: a foreign key constraint fails` 오류

### Step 4. ON DELETE CASCADE (연쇄 삭제)

```sql
CREATE TABLE emp (
    empno    INT PRIMARY KEY,
    ename    VARCHAR(50),
    deptno   INT,
    FOREIGN KEY (deptno) REFERENCES dept(deptno)
        ON DELETE CASCADE    -- 부모(dept) 행 삭제 시 자식(emp) 행도 자동 삭제
);
```

| FK 옵션 | 동작 |
|---|---|
| (기본값) | 자식 행이 있으면 부모 삭제 **차단** |
| `ON DELETE CASCADE` | 부모 삭제 시 자식 행도 **함께 삭제** |
| `ON DELETE SET NULL` | 부모 삭제 시 자식의 FK 컬럼을 **NULL**로 설정 |

### Step 5. 제약조건 전체 비교 참조표

| 제약조건 | 중복 | NULL | 테이블당 개수 | 주요 용도 |
|---|---|---|---|---|
| `PRIMARY KEY` | ❌ | ❌ | 1개 | 행의 대표 식별자 |
| `UNIQUE` | ❌ | ✅ | 여러 개 | 이메일, 아이디, 주민번호 |
| `FOREIGN KEY` | ✅ | ✅ | 여러 개 | 타 테이블 참조, 관계 표현 |
| `NOT NULL` | ✅ | ❌ | 여러 개 | 필수 입력 컬럼 |
| `DEFAULT` | ✅ | — | 여러 개 | 기본값 자동 설정 |
| `CHECK` | ✅ | ✅ | 여러 개 | 값 범위 제한 |

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```sql
-- 테이블 제약조건 포함 DDL 전체 확인
SHOW CREATE TABLE emp;

-- 테이블 구조 확인 (Key 컬럼에서 PK/UNI 확인)
DESC member;

-- 외래키 참조 관계 확인 (information_schema)
SELECT TABLE_NAME, CONSTRAINT_NAME, CONSTRAINT_TYPE
FROM information_schema.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = DATABASE();
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. FK INSERT 오류 (자식 → 부모 없음)

- **증상:** `INSERT INTO emp VALUES(9999, 'TEST', 99);` → FK 오류
- **원인:** `dept.deptno = 99` 가 존재하지 않음
- **해결:**
  ```sql
  -- 방법 1: 부모 먼저 삽입
  INSERT INTO dept VALUES(99, 'NEW_DEPT');
  INSERT INTO emp VALUES(9999, 'TEST', 99);
  -- 방법 2: 존재하는 deptno 사용
  INSERT INTO emp VALUES(9999, 'TEST', 10);
  ```

#### 🚨 시나리오 2. 부모 행 삭제 오류 (자식이 참조 중)

- **증상:** `DELETE FROM dept WHERE deptno = 10;` → FK 오류
- **원인:** `emp.deptno = 10` 인 행이 존재 → 참조 무결성 위반
- **해결:**
  ```sql
  -- 자식 먼저 삭제 후 부모 삭제
  DELETE FROM emp WHERE deptno = 10;
  DELETE FROM dept WHERE deptno = 10;
  -- 또는 ON DELETE CASCADE 옵션 사용 (테이블 재생성 필요)
  ```

#### 🚨 시나리오 3. UNIQUE 중복 오류

- **증상:** `INSERT INTO member ... VALUES (..., 'user01', ...)` → 오류 (username이 이미 존재)
- **원인:** UNIQUE 제약조건 위반
- **해결:**
  ```sql
  -- 중복 여부 사전 확인
  SELECT COUNT(*) FROM member WHERE username = 'user01';
  -- 또는 INSERT IGNORE (오류 무시, 중복 행 건너뜀)
  INSERT IGNORE INTO member (...) VALUES (...);
  -- 또는 ON DUPLICATE KEY UPDATE
  INSERT INTO member (...) VALUES (...)
  ON DUPLICATE KEY UPDATE phone = '010-0000-0000';
  ```

---

> 📌 **핵심 요약**
> - 제약조건 = DB 레벨 안전장치. 애플리케이션이 놓친 것도 DB가 마지막에 차단
> - `PRIMARY KEY`: 중복 불가 + NULL 불가 + 테이블당 1개 → 대표 식별자
> - `UNIQUE`: 중복 불가 + NULL 허용 + 여러 개 가능 → 이메일, 아이디 등
> - `FOREIGN KEY`: 부모 테이블 PK 참조 → 생성은 부모 먼저, 삭제는 자식 먼저
> - `ON DELETE CASCADE`: 부모 삭제 시 자식도 함께 삭제되는 연쇄 옵션
> - 관련: 🔧 DB - SQL 문법 (DDL·DML·DCL) · 📋 emp·dept 테이블 정의 및 데이터 · 🔗 DB - INNER JOIN 실습
