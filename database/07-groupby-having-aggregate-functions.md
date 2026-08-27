# 📊 DB - GROUP BY · HAVING · 집계함수

> **Tag:** #SQL #GROUPBY #HAVING #집계함수 #SUM #AVG #MAX #MIN #COUNT #서브쿼리
> **핵심 요약:** GROUP BY는 동일한 값을 가진 행들을 그룹으로 묶어 **집계 함수(SUM·AVG·MAX·MIN·COUNT)** 를 그룹 단위로 계산하는 기능이다. HAVING은 WHERE와 달리 **그룹이 만들어진 후** 그룹 자체에 조건을 걸 수 있으며, 집계함수 조건은 반드시 HAVING에서 사용해야 한다.

---

## 1. 📖 개요 (Overview)

GROUP BY가 필요한 이유는 데이터를 그룹 단위로 분석·비교하기 위해서다. 전체 평균이 아닌 "부서별 평균 급여", "직무별 최고 급여", "연도별 매출 합계"처럼 **기준별 통계**를 구하려면 GROUP BY가 필수다. GROUP BY 없이 `SUM(sal)`을 쓰면 전체 합계(행 1개)가 나오지만, `GROUP BY deptno` 후 `SUM(sal)`을 쓰면 부서별 합계(부서 개수만큼 행)가 나온다. SELECT 절에는 두 종류의 값만 올 수 있는데, `GROUP BY에 적은 컬럼` 또는 `집계함수(SUM·AVG·COUNT·MAX·MIN)`뿐이다. WHERE에는 집계함수를 사용할 수 없으므로 집계 결과에 조건을 걸려면 **HAVING**을 사용해야 한다.

GROUP BY의 처리 순서는 SQL 엔진이 내부적으로 `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY` 순으로 처리한다. 즉 먼저 `FROM`으로 테이블을 읽고, `WHERE`로 조건에 맞지 않는 행을 제거하는데 이 단계에서는 집계함수를 사용할 수 없다. 다음으로 `GROUP BY`가 조건을 통과한 행들을 그룹으로 묶고, `HAVING`이 그룹 결과를 필터링하는데 이 단계에서는 집계함수 조건 사용이 가능하다. 마지막으로 `SELECT`가 필요한 컬럼만 출력하고 `ORDER BY`가 정렬한다.

집계함수는 그룹 내 여러 행을 하나의 값으로 계산하며 5종이 있다. `SUM(컬럼)`은 합계를 구하고(예: `SUM(sal)` → 급여 총합), `AVG(컬럼)`은 평균을 구하며(예: `AVG(sal)` → 평균 급여), `MAX(컬럼)`은 최대값을(예: `MAX(sal)` → 최고 급여), `MIN(컬럼)`은 최소값을(예: `MIN(sal)` → 최저 급여) 구한다. `COUNT(*)`는 행 개수를 구한다(예: `COUNT(*)` → 사원 수).

HAVING과 WHERE의 차이는, WHERE는 행(Row) 조건으로 그룹 **이전** 단계에서 실행되고, HAVING은 그룹 조건으로 그룹이 **만들어진 후** 집계 결과에 조건을 건다는 점이다. `WHERE AVG(sal) >= 2000`은 그룹이 아직 없어 AVG 계산이 불가능하므로 **오류**가 나지만, `HAVING AVG(sal) >= 2000`은 GROUP BY 이후 실행되므로 정상 동작한다. **WHERE와 HAVING은 동시에 사용 가능**한데, WHERE로 대상 행을 먼저 필터링한 뒤 GROUP BY로 그룹화하고 HAVING으로 그룹을 필터링하는 방식이다. EX8처럼 `WHERE sal >= 1500`(행 필터)와 `HAVING AVG(sal) >= 2500`(그룹 필터)을 조합하는 패턴이 자주 출제된다.

HAVING 절에 서브쿼리를 사용하는 이유는 조건으로 사용할 값이 고정값이 아닌 **계산 결과(전체 평균 등)**일 때 서브쿼리를 활용해 동적으로 비교하기 위해서다. `HAVING AVG(sal) > (SELECT AVG(sal) FROM emp)`는 전체 평균보다 높은 부서를 조회하는 예로, 서브쿼리가 먼저 실행되어 전체 평균값(2073.21...)을 계산한 뒤 메인쿼리의 HAVING 조건에 적용된다. 전체 평균을 `all_total_sal` 컬럼으로 함께 SELECT해 검증할 수도 있다.

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

### 2-1. GROUP BY 기본 형식

```sql
SELECT  그룹기준컬럼, 집계함수(컬럼)
FROM    테이블명
[WHERE  행 조건]
GROUP BY 그룹기준컬럼
[HAVING  그룹 조건]
[ORDER BY 정렬기준];
```

### 2-2. GROUP BY 실습 쿼리 (EX1~EX6, emp 테이블)

```sql
-- EX1) 부서별(deptno) 사원 수를 구하시오
SELECT deptno, COUNT(*)
FROM emp
GROUP BY deptno;
-- deptno | COUNT(*)
--     10 | 3
--     20 | 5
--     30 | 6
-- deptno 기준으로 그룹을 만들고 해당 그룹의 행 개수만큼 COUNT

-- EX2) 직무(job)별 최고 급여(MAX(sal))를 구하시오
SELECT job, MAX(sal)
FROM emp
GROUP BY job;
-- job       | MAX(sal)
-- ANALYST   | 3000.00
-- CLERK     | 1300.00
-- MANAGER   | 2975.00
-- PRESIDENT | 5000.00
-- SALESMAN  | 1600.00

-- EX3) 직무(job)별 평균 급여를 구하시오
SELECT job, AVG(sal)
FROM emp
GROUP BY job;
-- job       | AVG(sal)
-- ANALYST   | 3000.000000
-- CLERK     | 1037.500000
-- MANAGER   | 2758.333333
-- PRESIDENT | 5000.000000
-- SALESMAN  | 1400.000000

-- EX4) 부서별 급여 총합(SUM)을 구하고 부서번호 기준으로 정렬 (단일 집계)
SELECT deptno, SUM(sal)
FROM emp
GROUP BY deptno
ORDER BY deptno;
-- deptno | SUM(sal)
--     10 |  8750.00
--     20 | 10875.00
--     30 |  9400.00

-- EX4 확장) 부서별 사원수, 총급여, 평균급여 한 번에
SELECT deptno, COUNT(*), SUM(sal), AVG(sal)
FROM emp
GROUP BY deptno
ORDER BY deptno;
-- deptno | COUNT(*) | SUM(sal) |   AVG(sal)
--     10 |        3 |  8750.00 | 2916.666667
--     20 |        5 | 10875.00 | 2175.000000
--     30 |        6 |  9400.00 | 1566.666667

-- EX6) 부서별(deptno) 평균 급여를 구하되, 급여 1000 이상인 사원만 대상으로 계산
--      (WHERE + GROUP BY 조합: WHERE로 행 먼저 필터 → 필터된 행으로 GROUP BY)
SELECT deptno, AVG(sal) AS sal_avg
FROM emp
WHERE sal >= 1000
GROUP BY deptno;
-- deptno |    sal_avg
--     10 | 2916.666667
--     20 | 2518.750000
--     30 | 1690.000000
```

### 2-3. HAVING 실습 쿼리 (EX1~EX9, emp 테이블)

```sql
-- 잘못된 예) WHERE에 집계함수 사용 → 오류
-- AVG(sal)은 그룹이 만들어진 후 계산 → WHERE는 그룹 이전 단계 → 사용 불가
SELECT deptno, AVG(sal)
FROM emp
WHERE AVG(sal) >= 2000   -- ❌ 오류
GROUP BY deptno;

-- EX1) 부서별(deptno) 평균 급여가 2000 이상인 부서만 조회
SELECT deptno, AVG(sal)
FROM emp
GROUP BY deptno
HAVING AVG(sal) >= 2000;
-- deptno |   AVG(sal)
--     10 | 2916.666667
--     20 | 2175.000000

-- EX2) 부서별(deptno) 급여 합계(SUM)가 9000 이상인 부서를 조회
SELECT deptno, SUM(sal) AS total_sal
FROM emp
GROUP BY deptno
HAVING SUM(sal) >= 9000;
-- deptno | total_sal
--     20 | 10875.00
--     30 |  9400.00

-- EX3) 직무(job)별 최대 급여(MAX)가 3000 이상인 직무를 조회
SELECT job, MAX(sal) AS max_sal
FROM emp
GROUP BY job
HAVING MAX(sal) >= 3000;
-- job       | max_sal
-- ANALYST   | 3000.00
-- PRESIDENT | 5000.00

-- EX4) 직무(job)별 평균 급여(AVG)가 1500 이상인 직무를 조회하고 평균 급여 내림차순 정렬
SELECT job, AVG(sal) AS avg_sal
FROM emp
GROUP BY job
HAVING AVG(sal) >= 1500
ORDER BY avg_sal DESC;
-- job       |    avg_sal
-- PRESIDENT | 5000.000000
-- ANALYST   | 3000.000000
-- MANAGER   | 2758.333333

-- EX5) 부서(deptno)별 사원 수(COUNT)가 4명 이상인 부서만 조회
SELECT deptno, COUNT(*) AS deptno_cnt
FROM emp
GROUP BY deptno
HAVING COUNT(*) >= 4;
-- deptno | deptno_cnt
--     20 |          5
--     30 |          6

-- EX6) 직무(job)별 최저 급여(MIN)가 1000 이상인 직무만 조회
SELECT job, MIN(sal) AS min_sal
FROM emp
GROUP BY job
HAVING MIN(sal) >= 1000;
-- job       | min_sal
-- ANALYST   | 3000.00
-- MANAGER   | 2450.00
-- PRESIDENT | 5000.00
-- SALESMAN  | 1250.00

-- EX7) 직무(job)별 급여 합계(SUM)와 사원 수(COUNT)를 계산하고,
--      급여 합계 >= 5000 이면서 사원 수 >= 2인 직무만 조회 (HAVING 복합 조건)
SELECT job,
       SUM(sal) AS total_sal,
       COUNT(*) AS job_cnt
FROM emp
GROUP BY job
HAVING SUM(sal) >= 5000
   AND COUNT(*) >= 2;
-- job      | total_sal | job_cnt
-- ANALYST  |   6000.00 |       2
-- MANAGER  |   8275.00 |       3
-- SALESMAN |   5600.00 |       4

-- EX8) 급여(sal) 1500 이상인 사원만 대상으로 부서(deptno)별 평균 급여(AVG)를 계산하고,
--      평균 급여 >= 2500인 부서 조회 (WHERE + GROUP BY + HAVING 3단 조합)
SELECT deptno, AVG(sal) AS avg_sal
FROM emp
WHERE sal >= 1500        -- 행 필터: 1500 미만 사원 제거
GROUP BY deptno
HAVING AVG(sal) >= 2500; -- 그룹 필터: 평균 2500 미만 부서 제거
-- deptno |    avg_sal
--     10 | 3725.000000
--     20 | 2991.666667

-- EX9) 부서(deptno)별 평균 급여가 전체 사원 평균 급여보다 높은 부서만 조회 (HAVING 서브쿼리)
SELECT deptno, AVG(sal) AS avg_sal
FROM emp
GROUP BY deptno
HAVING AVG(sal) > (
    SELECT AVG(sal) FROM emp   -- 전체 평균 = 2073.214286
);
-- deptno |    avg_sal
--     10 | 2916.666667
--     20 | 2175.000000

-- EX9 확장) 전체 평균도 함께 출력해서 검증
SELECT deptno, AVG(sal) AS avg_sal,
       (SELECT AVG(sal) FROM emp) AS all_total_sal
FROM emp
GROUP BY deptno
HAVING AVG(sal) > (
    SELECT AVG(sal) FROM emp
);
-- deptno |    avg_sal | all_total_sal
--     10 | 2916.666667 | 2073.214286
--     20 | 2175.000000 | 2073.214286
```

### 2-4. EX9 서브쿼리 실행 순서 해설

| 단계 | 내용 |
|---|---|
| 1) 서브쿼리 실행 | `SELECT AVG(sal) FROM emp` → 전체 평균 계산 (2073.21…) |
| 2) FROM 실행 | `FROM emp` 테이블 읽기 |
| 3) GROUP BY 실행 | `GROUP BY deptno` → 부서별 그룹 생성 |
| 4) 집계 계산 | `AVG(sal)` → 부서별 평균 급여 계산 |
| 5) HAVING 비교 | 부서별 평균 > 전체 평균 조건 비교 |
| 6) 조건 만족 부서 선택 | 10번·20번 부서 선택 |
| 7) SELECT 출력 | `deptno, AVG(sal) AS avg_sal` 출력 |

---

## 3. 🔍 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```sql
-- GROUP BY 없이 집계 (전체 기준)
SELECT COUNT(*), SUM(sal), AVG(sal) FROM emp;

-- 부서별 집계 전체 확인
SELECT deptno, COUNT(*), SUM(sal), AVG(sal), MAX(sal), MIN(sal)
FROM emp
GROUP BY deptno
ORDER BY deptno;

-- 전체 평균 급여 확인 (EX9 서브쿼리 검증)
SELECT AVG(sal) FROM emp;
```

### 3-2. 트러블슈팅 시나리오

#### 🚨 시나리오 1. WHERE에서 집계함수 사용 오류

- **증상:** `WHERE AVG(sal) >= 2000` → `Invalid use of group function` 오류
- **원인:** WHERE는 그룹 생성 이전 단계 → 집계함수 실행 불가
- **해결:** `HAVING AVG(sal) >= 2000` 으로 변경

#### 🚨 시나리오 2. SELECT에 GROUP BY 컬럼 외 일반 컬럼 포함

- **증상:** `SELECT deptno, ename, COUNT(*) FROM emp GROUP BY deptno;` → 오류 또는 예상 밖 결과
- **원인:** `ename`은 GROUP BY에 없음 → 그룹 내 어떤 행의 ename을 출력할지 알 수 없음
- **해결:** SELECT에는 `GROUP BY에 지정한 컬럼` 또는 `집계함수`만 사용

#### 🚨 시나리오 3. HAVING vs WHERE 혼동

- WHERE: 행 조건 (집계 이전), 개별 행 필터 → 집계함수 사용 불가
- HAVING: 그룹 조건 (집계 이후), 그룹 필터 → 집계함수 사용 가능
- 두 절 동시 사용 시 처리 순서: `WHERE(행 필터) → GROUP BY → HAVING(그룹 필터)`

#### 🚨 시나리오 4. COUNT(*) vs COUNT(컬럼) 차이

- `COUNT(*)`: NULL 포함 모든 행 개수
- `COUNT(comm)`: comm 컬럼이 NULL이 아닌 행만 개수 (emp의 경우 SALESMAN만 comm이 있음)

---

> 📌 **핵심 요약**
> - GROUP BY = 동일 값 행들을 그룹화 → 집계함수로 그룹 단위 통계 계산
> - 집계함수 5종: `SUM(합계)` · `AVG(평균)` · `MAX(최대)` · `MIN(최소)` · `COUNT(개수)`
> - SELECT 제약: GROUP BY 지정 컬럼 + 집계함수만 사용 가능
> - 처리 순서: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
> - HAVING: 그룹 조건 (집계함수 조건 가능) ↔ WHERE: 행 조건 (집계함수 조건 불가)
> - WHERE + GROUP BY + HAVING 3단 조합 가능 (EX8 패턴)
> - HAVING 서브쿼리: `HAVING AVG(sal) > (SELECT AVG(sal) FROM emp)` (EX9 패턴)
> - 관련: DB - SELECT·WHERE·ORDER BY·LIKE 실습 · emp·dept 테이블 정의 및 데이터 · DB - INNER JOIN 실습
