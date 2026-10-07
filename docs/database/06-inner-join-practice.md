# DB - INNER JOIN 실습 (customer · orders)

> **Tag:** #SQL #JOIN #INNERJOIN #집계함수 #GROUPBY #HAVING #COUNT #SUM #테이블조인
> **핵심 요약:** INNER JOIN은 두 테이블에서 **조건이 일치하는 행만** 결합해 조회하는 방식이다. `ON A.컬럼 = B.컬럼` 으로 연결 기준을 지정하며, 한쪽에만 존재하는 행은 결과에서 제외된다. customer(고객)·orders(주문) 실습 테이블로 INNER JOIN + WHERE + ORDER BY + LIKE + BETWEEN + IN + COUNT/SUM/GROUP BY/HAVING 전 개념을 통합 연습한다.

---

## 1. 개요 (Overview)

실무 DB는 데이터를 여러 테이블에 나눠 저장한다. "회원 이름 + 주문 상품명 + 결제 금액"처럼 한 테이블만으로 조회할 수 없는 정보를 한 화면에 보여주려면 JOIN이 반드시 필요하다. 쇼핑몰 DB를 예로 들면 회원 정보는 `member`, 주문 정보는 `orders`, 상품 정보는 `product`에 나뉘어 있는데, 이 세 테이블을 **한 번에** 조회하는 것이 JOIN이다. JOIN을 사용할 때 가장 중요한 것은 **두 테이블을 연결할 기준(컬럼)**을 `ON` 절에 지정하는 것이며, `member.member_id = orders.member_id`처럼 FK 관계가 JOIN의 연결 고리가 된다.

INNER JOIN은 두 테이블을 연결해 **조건(ON 절)이 일치하는 행만** 결과에 포함한다. 한쪽에만 존재하는 행(주문 없는 고객, 고객 없는 주문)은 제외된다. 즉 주문한 고객은 결과에 표시되지만 주문하지 않은 고객은 결과에서 제외되며, 고객 테이블에 없는 customer_id가 orders에 있다면 역시 제외된다(다만 FK가 있으면 이런 상황 자체가 발생할 수 없다). `AS` 테이블 별칭을 사용하면 `FROM customer AS c`처럼 지정해 `c.name`, `c.city`처럼 짧게 참조할 수 있다.

INNER JOIN의 기본 문법은 `FROM 테이블A AS A INNER JOIN 테이블B AS B ON A.공통컬럼 = B.공통컬럼;` 형태다.

```sql
SELECT   컬럼들
FROM     테이블A AS A
INNER JOIN 테이블B AS B
    ON A.공통컬럼 = B.공통컬럼;
```

`AS`는 생략 가능하며(`FROM customer c`), 조건은 반드시 `ON` 절에 작성하고 `WHERE`는 JOIN 이후 행 필터링에 사용한다. `ON A.id = B.id AND B.status = 'PAID'`처럼 JOIN 조건 안에 추가 조건을 작성할 수도 있으며, `WHERE`와 결합해 JOIN 이후 특정 도시·금액·날짜 조건으로 추가 필터링도 가능하다.

GROUP BY와 HAVING은 JOIN 이후 만들어진 결과 테이블에 `GROUP BY`로 그룹화하고 `COUNT()`, `SUM()` 등으로 집계한 뒤 `HAVING`으로 그룹 조건을 필터링하는 방식으로 결합한다. 예를 들어 EX11처럼 `GROUP BY c.city`는 도시별 주문 건수·금액을 집계하고, EX12처럼 `HAVING COUNT(o.order_id) >= 2`는 2건 이상 주문한 고객만 필터링한다. WHERE는 행 필터(JOIN 전/후)이고, HAVING은 그룹 필터(집계 후)라는 점이 핵심 차이다.

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** MariaDB/MySQL 계열 RDBMS.

### Step 1. customer 테이블 생성

```sql
-- customer 테이블 (고객 기본 정보 저장)
-- customer_id : 로그인 시 사용하는 실제 사용자 ID (PK)
-- name        : 고객 실명
-- gender      : 성별 (M/F)
-- age         : 나이
-- phone       : 휴대폰 번호
-- city        : 거주 도시
-- join_date   : 회원 가입일자
CREATE TABLE customer (
    customer_id VARCHAR(30) PRIMARY KEY,
    name        VARCHAR(50),
    gender      ENUM('M','F'),
    age         INT,
    phone       VARCHAR(20),
    city        VARCHAR(50),
    join_date   DATE
);
```

### Step 2. customer 데이터 삽입 (30명)

```sql
INSERT INTO customer VALUES
('minsu25',   '김민수', 'M', 25, '010-2001-0001', '서울', '2023-01-05'),
('seoyeon88', '이서연', 'F', 31, '010-2001-0002', '부산', '2023-01-10'),
('jihoon29',  '박지훈', 'M', 29, '010-2001-0003', '대구', '2023-02-12'),
('yurichoi',  '최유리', 'F', 40, '010-2001-0004', '인천', '2023-03-01'),
('dohyun22',  '정도현', 'M', 22, '010-2001-0005', '서울', '2023-03-03'),
('yerin_h',   '한예린', 'F', 35, '010-2001-0006', '광주', '2023-04-22'),
('seojoon7',  '윤서준', 'M', 28, '010-2001-0007', '부산', '2023-05-20'),
('hanul33',   '오하늘', 'F', 33, '010-2001-0008', '대전', '2023-06-25'),
('minho_k',   '강민호', 'M', 27, '010-2001-0009', '서울', '2023-07-15'),
('jieun24',   '이지은', 'F', 24, '010-2001-0010', '인천', '2023-08-18'),
('jun_h42',   '서준혁', 'M', 42, '010-2001-0011', '부산', '2023-09-23'),
('gayoung31', '문가영', 'F', 31, '010-2001-0012', '서울', '2023-10-01'),
('dohyun26',  '김도현', 'M', 26, '010-2001-0013', '대구', '2023-10-07'),
('yuna_l',    '이유나', 'F', 38, '010-2001-0014', '광주', '2023-10-15'),
('junwoo15',  '박준우', 'M', 29, '010-2001-0015', '부산', '2023-11-01'),
('seulgi23',  '강슬기', 'F', 23, '010-2001-0016', '서울', '2023-11-03'),
('jinwoo45',  '김진우', 'M', 45, '010-2001-0017', '대전', '2023-11-07'),
('arinlee',   '이아린', 'F', 33, '010-2001-0018', '인천', '2023-11-12'),
('nakhyun',   '박낙현', 'M', 36, '010-2001-0019', '부산', '2023-11-18'),
('hayeon30',  '최하연', 'F', 30, '010-2001-0020', '광주', '2023-11-20'),
('jiwon28',   '김지원', 'M', 28, '010-2001-0021', '서울', '2023-11-25'),
('soyoung41', '정소영', 'F', 41, '010-2001-0022', '부산', '2023-11-27'),
('seok33',    '오석진', 'M', 34, '010-2001-0023', '대구', '2023-11-29'),
('hyemi21',   '박혜미', 'F', 21, '010-2001-0024', '서울', '2023-12-01'),
('taemin32',  '이태민', 'M', 32, '010-2001-0025', '광주', '2023-12-05'),
('yeseo29',   '문예서', 'F', 29, '010-2001-0026', '부산', '2023-12-07'),
('yoonsik37', '조윤식', 'M', 37, '010-2001-0027', '대전', '2023-12-09'),
('jiwoo43',   '서지우', 'F', 43, '010-2001-0028', '서울', '2023-12-11'),
('geon22',    '최건우', 'M', 22, '010-2001-0029', '부산', '2023-12-15'),
('serin39',   '홍세린', 'F', 39, '010-2001-0030', '광주', '2023-12-20');
```

### Step 3. orders 테이블 생성

```sql
-- orders 테이블 (고객 주문 정보 저장)
-- order_id      : 주문 번호 (PK)
-- customer_id   : 주문한 고객 ID → customer(customer_id) FK
-- product       : 구매 상품명
-- category      : 상품 카테고리
-- amount        : 결제 금액
-- order_status  : 주문 상태 (READY/PAID/CANCEL)
-- order_date    : 주문 날짜
CREATE TABLE orders (
    order_id     INT PRIMARY KEY,
    customer_id  VARCHAR(30),
    product      VARCHAR(50),
    category     VARCHAR(50),
    amount       INT,
    order_status ENUM('READY','PAID','CANCEL'),
    order_date   DATE,
    FOREIGN KEY (customer_id) REFERENCES customer(customer_id)
);
```

### Step 4. orders 데이터 삽입 (50건)

```sql
INSERT INTO orders VALUES
(1,  'minsu25',   '키보드',           'IT기기',   45000,   'PAID',   '2024-01-05'),
(2,  'minsu25',   '마우스',           'IT기기',   25000,   'PAID',   '2024-01-10'),
(3,  'seoyeon88', '모니터',           'IT기기',   210000,  'PAID',   '2024-02-12'),
(4,  'jihoon29',  'USB허브',          'IT기기',   15000,   'READY',  '2024-03-01'),
(5,  'jihoon29',  '외장하드',         'IT기기',   98000,   'PAID',   '2024-03-03'),
(6,  'yurichoi',  '노트북',           'IT기기',   1250000, 'PAID',   '2024-01-22'),
(7,  'dohyun22',  '키보드',           'IT기기',   45000,   'PAID',   '2024-02-20'),
(8,  'yerin_h',   '마우스패드',       'IT기기',   12000,   'READY',  '2024-02-25'),
(9,  'yerin_h',   'USB케이블',        'IT기기',   8000,    'PAID',   '2024-03-10'),
(10, 'seojoon7',  '스피커',           'IT기기',   67000,   'PAID',   '2024-01-15'),
(11, 'hanul33',   '헤드셋',           'IT기기',   99000,   'PAID',   '2024-02-17'),
(12, 'hanul33',   '키보드',           'IT기기',   45000,   'PAID',   '2024-03-22'),
(13, 'minho_k',   '웹캠',             'IT기기',   55000,   'PAID',   '2024-03-25'),
(14, 'jieun24',   '노트북',           'IT기기',   1320000, 'PAID',   '2024-02-14'),
(15, 'jieun24',   '마우스',           'IT기기',   25000,   'PAID',   '2024-02-15'),
(16, 'jun_h42',   '스탠드',           '생활용품', 30000,   'READY',  '2024-03-11'),
(17, 'gayoung31', '키보드',           'IT기기',   45000,   'PAID',   '2024-02-01'),
(18, 'gayoung31', '마우스',           'IT기기',   25000,   'PAID',   '2024-02-02'),
(19, 'dohyun26',  '블루투스 이어폰', 'IT기기',   89000,   'PAID',   '2024-01-07'),
(20, 'dohyun26',  '스피커',           'IT기기',   67000,   'PAID',   '2024-01-09'),
(21, 'yuna_l',    'USB허브',          'IT기기',   15000,   'READY',  '2024-03-15'),
(22, 'junwoo15',  '모니터',           'IT기기',   210000,  'PAID',   '2024-02-10'),
(23, 'seulgi23',  '키보드',           'IT기기',   45000,   'PAID',   '2024-01-18'),
(24, 'jinwoo45',  '헤드셋',           'IT기기',   99000,   'PAID',   '2024-03-18'),
(25, 'arinlee',   'USB케이블',        'IT기기',   8000,    'READY',  '2024-03-21'),
(26, 'arinlee',   '마우스패드',       'IT기기',   12000,   'PAID',   '2024-03-22'),
(27, 'nakhyun',   '웹캠',             'IT기기',   55000,   'PAID',   '2024-02-11'),
(28, 'hayeon30',  '스피커',           'IT기기',   67000,   'PAID',   '2024-01-12'),
(29, 'hayeon30',  '키보드',           'IT기기',   45000,   'PAID',   '2024-01-13'),
(30, 'jiwon28',   '노트북',           'IT기기',   1290000, 'READY',  '2024-03-01'),
(31, 'soyoung41', '모니터암',         'IT기기',   89000,   'PAID',   '2024-03-04'),
(32, 'seok33',    '책상조명',         '생활용품', 35000,   'PAID',   '2024-03-05'),
(33, 'hyemi21',   'USB허브',          'IT기기',   15000,   'READY',  '2024-02-20'),
(34, 'taemin32',  '키보드',           'IT기기',   45000,   'PAID',   '2024-03-09'),
(35, 'taemin32',  '마우스',           'IT기기',   25000,   'PAID',   '2024-03-10'),
(36, 'taemin32',  '노트북받침대',     '생활용품', 27000,   'PAID',   '2024-03-11'),
(37, 'yeseo29',   '스탠드',           '생활용품', 30000,   'PAID',   '2024-02-18'),
(38, 'yoonsik37', '블루투스 스피커',  'IT기기',   56000,   'READY',  '2024-03-12'),
(39, 'jiwoo43',   '가습기',           '생활용품', 42000,   'PAID',   '2024-01-25'),
(40, 'geon22',    '헤드셋',           'IT기기',   99000,   'PAID',   '2024-02-03'),
(41, 'serin39',   '분무기',           '생활용품', 7000,    'PAID',   '2024-03-05'),
(42, 'jieun24',   'USB C타입 젠더',   'IT기기',   9000,    'PAID',   '2024-03-08'),
(43, 'minho_k',   '장패드',           '생활용품', 16000,   'CANCEL', '2024-02-22'),
(44, 'junwoo15',  '소형 선풍기',      '생활용품', 19000,   'READY',  '2024-03-10'),
(45, 'minsu25',   '마우스패드',       'IT기기',   12000,   'PAID',   '2024-02-14'),
(46, 'jihoon29',  '스마트폰 거치대',  '생활용품', 11000,   'READY',  '2024-02-21'),
(47, 'seoyeon88', '노트북 받침대',    '생활용품', 27000,   'PAID',   '2024-02-28'),
(48, 'nakhyun',   '키보드 루프',      'IT기기',   9000,    'PAID',   '2024-03-02'),
(49, 'geon22',    '웹캠 가림막',      '기타',     5000,    'READY',  '2024-03-15'),
(50, 'gayoung31', '스마트워치 충전기','IT기기',   29000,   'PAID',   '2024-03-18');
```

### Step 5. INNER JOIN 실습 쿼리 (EX1~EX12)

```sql
-- EX1) 고객 이름(name)과 주문 상품명(product)을 INNER JOIN으로 조회
SELECT c.name, o.product
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id;

-- EX2) 고객 이름, 상품명, 주문 날짜 조회
SELECT c.name, o.product, o.order_date
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id;

-- EX3) 거주 도시가 서울인 고객의 이름, 주문 상품명, 주문일 조회
SELECT c.name, o.product, o.order_date
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.city = '서울';

-- EX3 확장) city 컬럼도 함께 출력
SELECT c.name, c.city, o.product, o.order_date
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.city = '서울';

-- EX4) 주문 상태가 'PAID'인 주문에 대해 고객 이름, 상품명, 결제 금액 조회
SELECT c.name, o.product, o.amount
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'PAID';

-- EX5) 취소되지 않은 주문 조회 (CANCEL 제외)
SELECT c.name, o.product, o.amount, o.order_status
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status <> 'CANCEL';

-- EX6) 결제 금액 100,000 이상인 주문을 금액 내림차순으로 조회
SELECT c.name, o.product, o.amount
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.amount >= 100000
ORDER BY o.amount DESC;

-- EX7) 2024년 2월 주문을 날짜 오름차순으로 조회 (AND 방식 / BETWEEN 방식 두 가지)
SELECT c.name, o.product, o.amount, o.order_date
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-02-01' AND o.order_date <= '2024-02-29'
ORDER BY o.order_date ASC;

-- 또는 BETWEEN 사용 (시작일·종료일 포함)
SELECT c.name, o.product, o.amount, o.order_date
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date BETWEEN '2024-02-01' AND '2024-02-29'
ORDER BY o.order_date ASC;

-- EX8) 상품명에 '노트북'이 포함된 주문의 이름, 상품명, 금액, 상태 조회
SELECT c.name, o.product, o.amount, o.order_status
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.product LIKE '%노트북%';

-- EX9) 거주 도시가 부산 또는 대구인 고객 주문 (OR / IN 두 가지)
--       도시 오름차순 → 주문일 내림차순 정렬
SELECT c.name, c.city, o.product, o.order_date
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.city = '부산' OR c.city = '대구'
ORDER BY c.city ASC, o.order_date DESC;

-- IN 방식 (동일)
SELECT c.name, c.city, o.product, o.order_date
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.city IN ('부산', '대구')
ORDER BY c.city ASC, o.order_date DESC;

-- EX10) 나이 30~39세 고객의 이름, 나이, 주문 상품명, 결제 금액 조회 (AND / BETWEEN 두 가지)
SELECT c.name, c.age, o.product, o.amount
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.age >= 30 AND c.age <= 39;

-- BETWEEN 방식 (동일)
SELECT c.name, c.age, o.product, o.amount
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.age BETWEEN 30 AND 39;

-- EX11) 도시별 주문 건수(order_cnt)와 총 주문 금액(order_sum) 집계
--        주문 건수 내림차순 → 금액 내림차순 정렬
SELECT c.city,
       COUNT(o.order_id) AS order_cnt,
       SUM(o.amount)     AS order_sum
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.city
ORDER BY order_cnt DESC, order_sum DESC;

-- EX12) 주문 건수 2건 이상인 고객의 이름, 도시, 주문 건수, 총 금액 조회
--        주문 건수 내림차순 → 총 금액 내림차순 정렬
SELECT c.customer_id, c.name, c.city,
       COUNT(o.order_id) AS order_count,
       SUM(o.amount)     AS order_sum
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city
HAVING COUNT(o.order_id) >= 2
ORDER BY order_count DESC, order_sum DESC;
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

```sql
-- 전체 데이터 확인
SELECT * FROM customer;
SELECT * FROM orders;

-- 건수 확인 (customer: 30명, orders: 50건)
SELECT COUNT(*) FROM customer;
SELECT COUNT(*) FROM orders;

-- INNER JOIN 기본 연결 확인
SELECT c.customer_id, c.name, o.order_id, o.product
FROM customer c
INNER JOIN orders o ON c.customer_id = o.customer_id
LIMIT 10;
```

### 3-2. 트러블슈팅 시나리오

#### 시나리오 1. ON 절 없이 JOIN → 카티션 곱 발생

- **증상:** `FROM customer c INNER JOIN orders o;` → 30 × 50 = 1500행 출력
- **원인:** ON 절 없이 INNER JOIN 시 모든 행 조합이 출력됨
- **해결:** 반드시 `ON c.customer_id = o.customer_id` 지정

#### 시나리오 2. 어느 테이블 컬럼인지 모호 오류

- **증상:** `SELECT customer_id FROM customer INNER JOIN orders ON ...` → `Ambiguous column name` 오류
- **원인:** 두 테이블 모두 `customer_id` 컬럼을 가지고 있어 어느 것인지 불명확
- **해결:** 테이블 별칭으로 명확히 지정 → `SELECT c.customer_id`

#### 시나리오 3. HAVING에서 WHERE 사용 오류

- **증상:** `WHERE COUNT(o.order_id) >= 2` → 오류
- **원인:** 집계함수는 GROUP BY 이후에 계산되므로 WHERE에서 사용 불가
- **해결:** `HAVING COUNT(o.order_id) >= 2` 로 변경

#### 시나리오 4. IN 연산자 vs OR 성능 차이

- `WHERE c.city = '부산' OR c.city = '대구'` 와 `WHERE c.city IN ('부산', '대구')` 는 동일한 결과
- 항목이 많을수록 `IN` 이 가독성 측면에서 유리

---

>  **핵심 요약**
> - INNER JOIN = 두 테이블에서 ON 조건이 **일치하는 행만** 결합 (한쪽에만 있으면 제외)
> - `FROM A INNER JOIN B ON A.키 = B.키` — ON 절 필수
> - 조건 추가: JOIN 후 WHERE로 추가 필터 / BETWEEN으로 범위 / IN으로 복수 조건
> - GROUP BY + COUNT/SUM + HAVING: JOIN 결과를 집계 → 집계 조건 필터
> - customer(30명) · orders(50건) — FK: `orders.customer_id → customer.customer_id`
> - 관련: 제약조건 (PK·UNIQUE·FK) · DB - GROUP BY·HAVING·집계함수 · DB - SELECT·WHERE·ORDER BY·LIKE 실습
