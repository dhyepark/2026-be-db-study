# 5주차 DB 스터디 - JOIN 실행 방식

> 주제: MySQL 옵티마이저는 JOIN을 어떻게 실행할까?  
> 목표: JOIN 문법이 아니라, MySQL이 여러 테이블을 어떤 순서로 읽고 어떤 방식으로 조인하는지 이해한다.

---

# 1. 오늘의 목표

이번 시간에는 JOIN을 단순히 “테이블을 붙이는 문법”으로 보지 않는다.

대신 다음 질문에 답하는 것을 목표로 한다.

1. SQL에 적은 테이블 순서대로 JOIN이 실행될까?
2. 드라이빙 테이블과 드리븐 테이블은 무엇일까?
3. MySQL은 왜 JOIN 순서를 바꿀까?
4. JOIN 컬럼의 데이터 타입이 다르면 왜 문제가 될까?
5. OUTER JOIN은 왜 조인 최적화에 제약을 줄까?
6. LEFT JOIN에서 오른쪽 테이블 조건을 WHERE에 쓰면 왜 위험할까?
7. 지연된 조인은 어떤 상황에서 사용할 수 있을까?

---

# 2. 문제 제기

다음 쿼리를 보자.

```sql
SELECT *
FROM employees e
JOIN dept_emp de ON de.emp_no = e.emp_no
JOIN departments d ON d.dept_no = de.dept_no
WHERE d.dept_name = 'Development';
```

질문:

```text
이 쿼리는 정말 employees → dept_emp → departments 순서로 실행될까?
```

정답은 다음과 같다.

```text
꼭 그렇지 않다.
MySQL 옵티마이저가 비용을 보고 조인 순서를 결정한다.
```

SQL에 적은 순서는 사람이 읽기 위한 논리적 표현에 가깝다.

DBMS는 같은 결과를 만들 수 있다면 더 적은 비용의 실행 순서를 선택할 수 있다.

---

# 3. SELECT 처리 순서와 JOIN

SQL은 보통 다음 절들로 구성된다.

```sql
SELECT ...
FROM ...
JOIN ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
LIMIT ...
```

논리적으로는 대략 다음 순서로 처리된다고 이해할 수 있다.

```text
FROM / JOIN
→ WHERE
→ GROUP BY
→ HAVING
→ SELECT
→ ORDER BY
→ LIMIT
```

하지만 실제 물리 실행에서는 옵티마이저가 더 효율적인 방식으로 처리할 수 있다.

특히 JOIN에서는 다음 질문이 중요하다.

```text
어떤 테이블을 먼저 읽을 것인가?
```

오늘 수업의 핵심 문장은 이것이다.

> SQL 작성 순서와 실제 실행 순서는 다를 수 있다.

---

# 4. JOIN이란?

JOIN은 두 개 이상의 테이블을 연결해서 하나의 결과를 만드는 작업이다.

예를 들어 회원과 주문 테이블이 있다고 하자.

```text
members

id | name
---|------
1  | 라이
2  | 봉구스
3  | 티모
```

```text
orders

id | member_id | price
---|-----------|------
1  | 1         | 10000
2  | 1         | 20000
3  | 2         | 30000
```

회원 이름과 주문 금액을 같이 보고 싶으면 JOIN을 사용한다.

```sql
SELECT m.name, o.price
FROM members m
JOIN orders o ON o.member_id = m.id;
```

결과:

```text
name   | price
-------|------
라이   | 10000
라이   | 20000
봉구스 | 30000
```

여기서 중요한 것은 `ON` 조건이다.

```sql
ON o.member_id = m.id
```

이 조건이 두 테이블을 연결하는 기준이다.

---

## 4.1 JOIN 실행 계획을 EXPLAIN으로 보기

JOIN은 결과만 보는 것보다 실행 계획을 같이 보는 것이 중요하다.

MySQL이 실제로 어떤 테이블을 먼저 읽고, 어떤 인덱스를 사용했는지는 `EXPLAIN`으로 확인한다.

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE employees (
    emp_no BIGINT NOT NULL,
    name VARCHAR(30) NOT NULL,
    gender CHAR(1) NOT NULL,
    PRIMARY KEY (emp_no),
    INDEX idx_employees_gender (gender)
) ENGINE = InnoDB;

CREATE TABLE departments (
    dept_no CHAR(4) NOT NULL,
    dept_name VARCHAR(40) NOT NULL,
    PRIMARY KEY (dept_no),
    UNIQUE KEY uk_departments_dept_name (dept_name)
) ENGINE = InnoDB;

CREATE TABLE dept_emp (
    emp_no BIGINT NOT NULL,
    dept_no CHAR(4) NOT NULL,
    PRIMARY KEY (emp_no, dept_no),
    INDEX idx_dept_emp_dept_no (dept_no)
) ENGINE = InnoDB;

INSERT INTO employees (emp_no, name, gender) VALUES
(10001, '라이', 'M'),
(10002, '봉구스', 'M'),
(10003, '티모', 'F'),
(10004, '서여', 'F');

INSERT INTO departments (dept_no, dept_name) VALUES
('d001', 'Marketing'),
('d002', 'Finance'),
('d005', 'Development');

INSERT INTO dept_emp (emp_no, dept_no) VALUES
(10001, 'd005'),
(10002, 'd001'),
(10003, 'd005'),
(10004, 'd002');

EXPLAIN FORMAT=TRADITIONAL
SELECT *
FROM employees e
JOIN dept_emp de ON de.emp_no = e.emp_no
JOIN departments d ON d.dept_no = de.dept_no
WHERE d.dept_name = 'Development';
```

실제 실행 결과는 MySQL 버전과 통계에 따라 조금 달라질 수 있다.

하지만 `table` 컬럼을 위에서 아래로 읽으면 MySQL이 어떤 순서로 테이블을 읽는지 확인할 수 있다.

`EXPLAIN` 결과에서 JOIN을 읽을 때 주로 보는 컬럼은 다음이다.

| 컬럼 | JOIN에서 보는 것 |
|---|---|
| id | SELECT 단위와 실행 블록 |
| select_type | 단순 SELECT인지, 서브쿼리/파생 테이블인지 |
| table | 어떤 테이블을 읽는지 |
| type | 테이블 접근 방식 |
| possible_keys | 사용할 수 있었던 후보 인덱스 |
| key | 실제 사용한 인덱스 |
| key_len | 사용한 인덱스 길이 |
| ref | 인덱스 탐색에 사용한 값 또는 컬럼 |
| rows | 읽을 것으로 예상한 row 수 |
| filtered | 읽은 row 중 조건을 통과할 것으로 예상한 비율 |
| Extra | 추가 실행 정보 |

처음에는 모든 컬럼을 외우려고 하지 말고, 아래 컬럼부터 보면 된다.

```text
table
type
possible_keys
key
ref
rows
Extra
```

---

## 4.2 table 컬럼으로 조인 순서 확인하기

`table` 컬럼은 MySQL이 어떤 테이블을 어떤 순서로 읽는지 보여준다.

예를 들어 실행 계획이 다음과 같다고 하자.

```text
table
-----
m
o
```

그러면 대략 다음 순서로 읽는다고 이해할 수 있다.

```text
members 먼저 읽기
→ orders 읽기
```

SQL에 작성한 순서가 다음과 같더라도:

```sql
FROM orders o
JOIN members m
```

실제 실행 계획은 다음처럼 다르게 나올 수 있다.

```text
orders → members
```

### 핵심

> SQL에 적은 JOIN 순서와 실제 실행 순서는 다를 수 있다.  
> `EXPLAIN`의 `table` 컬럼을 위에서 아래로 읽으면 실제 조인 순서를 추정할 수 있다.

### 실습: table 컬럼으로 조인 순서 확인하기

목표:

```text
인덱스 없는 두 테이블을 조인하고,
SQL에 작성한 테이블 순서가 EXPLAIN의 table 컬럼에 어떻게 나타나는지 비교한다.
```

반대 순서를 확실히 관찰하기 위해 이 실습에서는 MySQL의 `STRAIGHT_JOIN`을 사용한다.

`STRAIGHT_JOIN`은 왼쪽에 쓴 테이블을 먼저 읽도록 조인 순서를 고정한다.

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE no_index_members (
    member_id BIGINT NOT NULL,
    name VARCHAR(30) NOT NULL
) ENGINE = InnoDB;

CREATE TABLE no_index_orders (
    order_id BIGINT NOT NULL,
    member_id BIGINT NOT NULL,
    price INT NOT NULL
) ENGINE = InnoDB;

INSERT INTO no_index_members (member_id, name) VALUES
(1, '라이'),
(2, '봉구스'),
(3, '티모');

INSERT INTO no_index_orders (order_id, member_id, price) VALUES
(101, 1, 10000),
(102, 1, 20000),
(103, 2, 15000),
(104, 3, 30000);

EXPLAIN FORMAT=TRADITIONAL
SELECT *
FROM no_index_members m
STRAIGHT_JOIN no_index_orders o ON o.member_id = m.member_id;

EXPLAIN FORMAT=TRADITIONAL
SELECT *
FROM no_index_orders o
STRAIGHT_JOIN no_index_members m ON m.member_id = o.member_id;
```

실제 실행 결과 1:

```text
id | select_type | table | type | possible_keys | key  | rows | Extra
---+-------------+-------+------+---------------+------+------+--------------------------------------------
1  | SIMPLE      | m     | ALL  | NULL          | NULL | 3    |
1  | SIMPLE      | o     | ALL  | NULL          | NULL | 4    | Using where; Using join buffer (hash join)
```

실제 실행 결과 2:

```text
id | select_type | table | type | possible_keys | key  | rows | Extra
---+-------------+-------+------+---------------+------+------+--------------------------------------------
1  | SIMPLE      | o     | ALL  | NULL          | NULL | 4    |
1  | SIMPLE      | m     | ALL  | NULL          | NULL | 3    | Using where; Using join buffer (hash join)
```

```text
첫 번째 쿼리는 m → o 순서로 작성했다.
두 번째 쿼리는 o → m 순서로 작성했다.
```

예상 해석:

```text
JOIN은 옵티마이저가 비용을 보고 조인 순서를 바꿀 수 있다.
STRAIGHT_JOIN은 작성한 순서대로 조인 순서를 고정한다.
이번 실습은 table 컬럼 순서 차이를 확실히 보기 위해 STRAIGHT_JOIN을 사용했다.
```

---

# 5. 조인 조건이 없으면 어떻게 될까?

조인 조건이 없거나 잘못되면 두 테이블의 모든 조합이 만들어질 수 있다. 데이터가 많아지면 문제가 심각해진다.

```text
members 100만 건
orders  100만 건

조건 없는 조인
→ 1조 조합
```

### 핵심

> JOIN은 테이블을 연결하는 작업이고, 조인 조건이 결과와 성능을 결정한다.

---

# 6. 드라이빙 테이블과 드리븐 테이블

MySQL 조인을 이해할 때 가장 중요한 용어다.

| 용어 | 의미 |
|---|---|
| 드라이빙 테이블 | 조인에서 먼저 읽는 테이블 |
| 드리븐 테이블 | 드라이빙 테이블의 결과를 바탕으로 나중에 읽는 테이블 |

예를 들어 다음 쿼리가 있다고 하자.

```sql
SELECT *
FROM members m
JOIN orders o ON o.member_id = m.id
WHERE m.id = 1;
```

실행 흐름은 대략 이렇게 될 수 있다.

```text
1. members에서 id = 1인 회원을 찾는다.
2. 찾은 member.id로 orders에서 주문을 찾는다.
```

이 경우:

```text
드라이빙 테이블: members
드리븐 테이블: orders
```

---

## 6.1 왜 드라이빙 테이블이 중요할까?

드라이빙 테이블에서 읽은 row 수만큼 드리븐 테이블을 반복해서 찾을 수 있기 때문이다.

```text
드라이빙 테이블 결과 1건
→ 드리븐 테이블 1번 탐색

드라이빙 테이블 결과 10만 건
→ 드리븐 테이블 10만 번 탐색 가능
```

그래서 보통은 먼저 읽는 테이블에서 결과를 최대한 줄이는 것이 중요하다.

```text
좋은 출발
→ 먼저 10건으로 줄임
→ 다음 테이블 10번만 탐색

나쁜 출발
→ 먼저 100만 건 읽음
→ 다음 테이블 100만 번 탐색
```

### 핵심

> 조인 성능은 어떤 테이블을 먼저 읽느냐에 크게 영향을 받는다.

---

# 7. MySQL의 기본 조인 모델: Nested Loop Join

MySQL 조인은 먼저 Nested Loop Join으로 이해하는 것이 좋다.

Nested Loop Join은 이름 그대로 반복문처럼 동작한다.

```text
for each row in driving_table:
    find matching rows in driven_table
```

예시:

```sql
SELECT *
FROM members m
JOIN orders o ON o.member_id = m.id
WHERE m.id = 1;
```

실행 흐름:

```text
1. members에서 id = 1 찾기
2. 찾은 id = 1로 orders.member_id = 1 찾기
3. 결과 조합
```

여러 건이면 이렇게 된다.

```text
members row 1 → orders 검색
members row 2 → orders 검색
members row 3 → orders 검색
members row 4 → orders 검색
```

### 핵심

> Nested Loop Join은 먼저 읽은 테이블의 결과마다 다음 테이블을 반복해서 찾는 방식이다.

---

# 8. JOIN의 순서와 인덱스

Real MySQL에서 강조하는 JOIN의 핵심은 다음이다.

```text
드라이빙 테이블을 읽을 때는 인덱스 탐색을 한 번 수행한 뒤 스캔하면 된다.

하지만 드리븐 테이블은 드라이빙 테이블에서 읽은 레코드 수만큼
인덱스 탐색과 스캔을 반복할 수 있다.
```

즉, 조인에서는 드리븐 테이블을 어떻게 읽는지가 매우 중요하다.

---

## 8.1 두 테이블 모두 조인 컬럼에 인덱스가 있는 경우

```text
employees.emp_no 인덱스 있음
dept_emp.emp_no 인덱스 있음
```

이 경우 어느 테이블을 먼저 읽어도 드리븐 테이블을 인덱스로 찾을 수 있다.

```text
employees → dept_emp 가능
dept_emp → employees 가능
```

옵티마이저는 통계 정보를 바탕으로 더 유리한 쪽을 선택한다.

- 각 테이블의 row 수
- 인덱스를 사용할 때 읽을 예상 row 수
- 조건의 선택도
- 어떤 테이블을 먼저 읽을 때 비용이 낮은지

예를 들어 `employees.gender = 'M'` 조건이 전체의 절반 정도를 반환한다고 판단하면, 옵티마이저는 `employees`에서 시작할 때 약 15만 건을 읽는다고 추정할 수 있다.

### 실습: 두 테이블 모두 조인 컬럼에 인덱스가 있는 JOIN

목표:

```text
members와 orders에 각각 1000건의 데이터를 넣고,
members.id와 orders.member_id 양쪽 모두 인덱스가 있을 때
어느 테이블을 먼저 읽어도 드리븐 테이블을 인덱스로 찾을 수 있음을 확인한다.
```

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE members (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(30) NOT NULL,
    grade VARCHAR(20) NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_members_grade (grade)
) ENGINE = InnoDB;

CREATE TABLE orders (
    id BIGINT NOT NULL AUTO_INCREMENT,
    member_id BIGINT NOT NULL,
    price INT NOT NULL,
    status VARCHAR(20) NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_orders_member_id (member_id),
    INDEX idx_orders_status (status)
) ENGINE = InnoDB;

INSERT INTO members (name, grade)
WITH RECURSIVE seq(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 1000
)
SELECT
    CONCAT('member-', n),
    IF(n % 2 = 0, 'VIP', 'BASIC')
FROM seq;

INSERT INTO orders (member_id, price, status)
WITH RECURSIVE seq(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 1000
)
SELECT
    n,
    10000 + n,
    IF(n % 3 = 0, 'READY', 'PAID')
FROM seq;

EXPLAIN FORMAT=TRADITIONAL
SELECT m.id, m.name, o.id AS order_id, o.price
FROM members m
JOIN orders o ON o.member_id = m.id
WHERE m.grade = 'VIP';

EXPLAIN FORMAT=TRADITIONAL
SELECT m.id, m.name, o.id AS order_id, o.price
FROM orders o
JOIN members m ON m.id = o.member_id
WHERE m.grade = 'VIP';
```

실제 실행 결과 1:

```text
id | select_type | table | type | possible_keys                     | key                  | ref                       | rows | Extra
---+-------------+-------+------+-----------------------------------+----------------------+---------------------------+------+-------------
1  | SIMPLE      | m     | ref  | PRIMARY,idx_members_grade         | idx_members_grade    | const                     | 500  | Using index condition
1  | SIMPLE      | o     | ref  | idx_orders_member_id              | idx_orders_member_id | join_explain_study.m.id   | 1    |
```

실제 실행 결과 2:

```text
id | select_type | table | type | possible_keys                     | key                  | ref                       | rows | Extra
---+-------------+-------+------+-----------------------------------+----------------------+---------------------------+------+-------------
1  | SIMPLE      | m     | ref  | PRIMARY,idx_members_grade         | idx_members_grade    | const                     | 500  | Using index condition
1  | SIMPLE      | o     | ref  | idx_orders_member_id              | idx_orders_member_id | join_explain_study.m.id   | 1    |
```

```text
members와 orders에 각각 1000건의 데이터가 들어간 것을 전제로 실행 계획을 확인한다.
members는 idx_members_grade 인덱스로 VIP 회원을 찾는 것을 관찰할 수 있다.
orders는 idx_orders_member_id 인덱스로 회원별 주문을 찾는 것을 관찰할 수 있다.
SQL 작성 순서를 바꿔도 옵티마이저가 같은 실행 순서를 선택할 수 있음을 관찰할 수 있다.
```

예상 해석:

```text
두 테이블 모두 조인 컬럼에 인덱스가 있으면 드리븐 테이블을 인덱스로 찾을 수 있다.
다만 실제 조인 순서는 SQL 작성 순서가 아니라 옵티마이저의 비용 판단에 따라 결정된다.
```

### 실습: 데이터가 많고 조건 선택도가 다른 경우

목표:

```text
데이터가 많아지고 조건의 선택도가 달라지면
옵티마이저가 어떤 테이블을 먼저 읽는지 달라질 수 있음을 확인한다.
```

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE members (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(30) NOT NULL,
    grade VARCHAR(20) NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_members_grade (grade)
) ENGINE = InnoDB;

CREATE TABLE orders (
    id BIGINT NOT NULL AUTO_INCREMENT,
    member_id BIGINT NOT NULL,
    price INT NOT NULL,
    status VARCHAR(20) NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_orders_member_id (member_id),
    INDEX idx_orders_status (status)
) ENGINE = InnoDB;

INSERT INTO members (name, grade)
WITH RECURSIVE seq(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 1000
)
SELECT
    CONCAT('member-', n),
    IF(n <= 10, 'VIP', 'BASIC')
FROM seq;

INSERT INTO orders (member_id, price, status)
WITH RECURSIVE seq(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 1000
)
SELECT
    IF(n <= 10, 1, ((n - 11) % 999) + 2),
    10000 + n,
    IF(n <= 10, 'CANCEL', 'PAID')
FROM seq;

EXPLAIN FORMAT=TRADITIONAL
SELECT m.id, m.name, o.id AS order_id, o.status
FROM members m
JOIN orders o ON o.member_id = m.id
WHERE m.grade = 'VIP';

EXPLAIN FORMAT=TRADITIONAL
SELECT m.id, m.name, o.id AS order_id, o.status
FROM members m
JOIN orders o ON o.member_id = m.id
WHERE o.status = 'CANCEL';
```

실제 실행 결과 1:

```text
table | type | key                  | ref                     | rows | Extra
------+------+----------------------+-------------------------+------+----------------------
m     | ref  | idx_members_grade    | const                   | 10   | Using index condition
o     | ref  | idx_orders_member_id | join_explain_study.m.id | 1    |
```

실제 실행 결과 2:

```text
table | type   | key               | ref                            | rows | Extra
------+--------+-------------------+--------------------------------+------+----------------------
o     | ref    | idx_orders_status | const                          | 10   | Using index condition
m     | eq_ref | PRIMARY           | join_explain_study.o.member_id | 1    |
```

```text
첫 번째 쿼리는 m.grade = 'VIP' 조건 때문에 members를 먼저 읽는 것을 관찰할 수 있다.
두 번째 쿼리는 o.status = 'CANCEL' 조건 때문에 orders를 먼저 읽는 것을 관찰할 수 있다.
두 쿼리 모두 SQL에는 members를 먼저 썼지만, 조건에 따라 실제 시작 테이블이 달라질 수 있다.
```

예상 해석:

```text
옵티마이저는 단순히 SQL 작성 순서를 따르지 않는다.
어떤 조건이 더 적은 row로 줄일 수 있는지 보고 조인 순서를 선택한다.
데이터 양과 조건 선택도가 달라지면 같은 JOIN 구조라도 실행 계획이 달라질 수 있다.
```

---

## 8.2 한쪽에만 인덱스가 있는 경우

드라이빙 테이블은 조인의 출발점으로 먼저 읽힌다.

그 다음 드라이빙 테이블에서 읽힌 각 row마다 조인 조건에 맞는 row를 드리븐 테이블에서 찾는다.

그래서 조인에서는 보통 드리븐 테이블의 조인 컬럼에 인덱스가 있는 것이 중요하다.

employees에서 3건을 읽었다면

```
employees row 1 → dept_emp에서 emp_no로 검색
employees row 2 → dept_emp에서 emp_no로 검색
employees row 3 → dept_emp에서 emp_no로 검색
```

이때 dept_emp.emp_no에 인덱스가 없으면:

```
employees row 1 → dept_emp 전체 스캔
employees row 2 → dept_emp 전체 스캔
employees row 3 → dept_emp 전체 스캔
```

반대로 dept_emp.emp_no에 인덱스가 있으면:

```
employees row 1 → dept_emp.emp_no 인덱스로 검색
employees row 2 → dept_emp.emp_no 인덱스로 검색
employees row 3 → dept_emp.emp_no 인덱스로 검색
```


### 실습: 드리븐 테이블을 인덱스로 찾는 JOIN

목표:

```text
one_index_orders.member_id 인덱스만 있을 때
JOIN 작성 순서를 바꿔도 옵티마이저가 어떤 실행 순서를 선택하는지 확인한다.
```

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE one_index_members (
    id BIGINT NOT NULL,
    name VARCHAR(30) NOT NULL,
    grade VARCHAR(20) NOT NULL
) ENGINE = InnoDB;

CREATE TABLE one_index_orders (
    id BIGINT NOT NULL AUTO_INCREMENT,
    member_id BIGINT NOT NULL,
    price INT NOT NULL,
    status VARCHAR(20) NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_one_index_orders_member_id (member_id)
) ENGINE = InnoDB;

INSERT INTO one_index_members (id, name, grade) VALUES
(1, '라이', 'VIP'),
(2, '봉구스', 'BASIC'),
(3, '티모', 'VIP'),
(4, '서여', 'BASIC');

INSERT INTO one_index_orders (member_id, price, status) VALUES
(1, 10000, 'PAID'),
(1, 20000, 'PAID'),
(2, 15000, 'READY'),
(3, 30000, 'PAID'),
(3, 40000, 'CANCEL');

EXPLAIN FORMAT=TRADITIONAL
SELECT m.id, m.name, o.id AS order_id, o.price
FROM one_index_members m
JOIN one_index_orders o ON o.member_id = m.id
WHERE m.id = 1;

EXPLAIN FORMAT=TRADITIONAL
SELECT m.id, m.name, o.id AS order_id, o.price
FROM one_index_orders o
JOIN one_index_members m ON o.member_id = m.id
WHERE m.id = 1;
```

실제 실행 결과 1:

```text
table | type | key                            | ref  | rows | Extra
------+------+--------------------------------+------+------+-------------
m     | ALL  | NULL                           | NULL | 4    | Using where
o     | ref  | idx_one_index_orders_member_id | const | 2    |
```

실제 실행 결과 2:

```text
table | type | key                            | ref  | rows | Extra
------+------+--------------------------------+------+------+-------------
m     | ALL  | NULL                           | NULL | 4    | Using where
o     | ref  | idx_one_index_orders_member_id | const | 2    |
```

```text
첫 번째 쿼리는 members → orders 순서로 작성했다.
두 번째 쿼리는 orders → members 순서로 작성했다.
하지만 두 실행 계획 모두 one_index_members를 먼저 전체 스캔하는 것을 관찰할 수 있다.
one_index_orders는 idx_one_index_orders_member_id 인덱스를 사용하는 것을 관찰할 수 있다.
```

예상 해석:

```text
one_index_members.id에는 인덱스가 없으므로 m.id = 1 조건도 전체 스캔으로 찾는다.
그 다음 찾은 id 값으로 one_index_orders.member_id 인덱스를 탐색한다.
한쪽에만 인덱스가 있으면 어느 테이블이 드리븐이 되는지가 더 중요해진다.
```

---

## 8.3 두 테이블 모두 인덱스가 없는 경우

두 테이블 모두 조인 컬럼에 인덱스가 없다면 어느 쪽을 드리븐으로 선택해도 부담이 크다.

```text
A에서 한 건 읽기
→ B 전체 스캔

또는

B에서 한 건 읽기
→ A 전체 스캔
```

이럴 때는 MySQL이 조인 버퍼나 해시 조인을 사용할 수 있다.

이 부분은 뒤에서 가볍게만 다룬다.


### 실습: 드리븐 테이블에 인덱스가 없는 JOIN

목표:

```text
두 테이블 모두 조인 컬럼에 인덱스가 없을 때
JOIN 실행 계획이 어떻게 달라지는지 확인한다.
```

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE no_index_members (
    member_id BIGINT NOT NULL,
    name VARCHAR(30) NOT NULL
) ENGINE = InnoDB;

CREATE TABLE no_index_orders (
    order_id BIGINT NOT NULL,
    member_id BIGINT NOT NULL,
    price INT NOT NULL
) ENGINE = InnoDB;

INSERT INTO no_index_members (member_id, name) VALUES
(1, '라이'),
(2, '봉구스'),
(3, '티모');

INSERT INTO no_index_orders (order_id, member_id, price) VALUES
(101, 1, 10000),
(102, 1, 20000),
(103, 2, 15000),
(104, 3, 30000);

EXPLAIN FORMAT=TRADITIONAL
SELECT m.member_id, m.name, o.order_id, o.price
FROM no_index_members m
JOIN no_index_orders o ON o.member_id = m.member_id
WHERE m.member_id = 1;
```

실제 실행 결과(핵심 컬럼):

```text
table | type | possible_keys | key  | rows | Extra
------+------+---------------+------+------+--------------------------------------------
m     | ALL  | NULL          | NULL | 3    | Using where
o     | ALL  | NULL          | NULL | 4    | Using where; Using join buffer (hash join)
```

```text
두 테이블 모두 key가 NULL인 것을 관찰할 수 있다.
두 테이블 모두 type이 ALL인 것을 관찰할 수 있다.
Extra에 Using join buffer 또는 hash join 관련 메시지가 나오는 것을 관찰할 수 있다.
```

예상 해석:

```text
두 테이블 모두 조인 컬럼에 인덱스가 없으므로
member_id 값으로 상대 테이블을 빠르게 찾기 어렵다.

그래서 orders를 전체 스캔하거나 조인 버퍼를 사용할 수 있다.
```

---

# 9. JOIN 컬럼의 데이터 타입

JOIN 조건의 양쪽 컬럼은 의미만 같으면 되는 것이 아니다.

데이터 타입도 맞아야 한다.

예를 들어 다음 두 테이블이 있다고 하자.

```sql
CREATE TABLE tb_test1 (
    user_id INT,
    user_type INT,
    PRIMARY KEY(user_id)
);
```

```sql
CREATE TABLE tb_test2 (
    user_type CHAR(1),
    type_desc VARCHAR(10),
    PRIMARY KEY(user_type)
);
```

그리고 다음과 같이 조인한다.

```sql
SELECT *
FROM tb_test1 t1
JOIN tb_test2 t2 ON t1.user_type = t2.user_type;
```

겉으로는 `user_type`끼리 조인하므로 괜찮아 보인다.

하지만 실제 타입은 다르다.

```text
t1.user_type = INT

t2.user_type = CHAR(1)
```

이 경우 MySQL은 비교를 위해 타입 변환을 해야 한다.

문제는 타입 변환이 인덱스 컬럼에 적용되면 인덱스를 효율적으로 사용하기 어렵다는 점이다.

인덱스를 제대로 활용하기 어렵기 때문에 조인 버퍼나 해시 조인이 사용될 수 있다.


## 실습: JOIN 컬럼 타입 불일치

목표:

```text
JOIN 컬럼의 데이터 타입이 같을 때는 인덱스로 조인 대상을 찾을 수 있고,
데이터 타입이 다르면 인덱스를 제대로 사용하지 못할 수 있음을 확인한다.
```

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE user_logs (
    id BIGINT NOT NULL AUTO_INCREMENT,
    user_type INT NOT NULL,
    message VARCHAR(100) NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_user_logs_user_type (user_type)
) ENGINE = InnoDB;

CREATE TABLE user_types_int (
    user_type INT NOT NULL,
    type_desc VARCHAR(30) NOT NULL,
    PRIMARY KEY (user_type)
) ENGINE = InnoDB;

CREATE TABLE user_types (
    user_type CHAR(1) NOT NULL,
    type_desc VARCHAR(30) NOT NULL,
    PRIMARY KEY (user_type)
) ENGINE = InnoDB;

INSERT INTO user_logs (user_type, message) VALUES
(1, 'login'),
(1, 'order'),
(2, 'logout'),
(3, 'cancel');

INSERT INTO user_types_int (user_type, type_desc) VALUES
(1, 'normal'),
(2, 'admin'),
(3, 'guest');

INSERT INTO user_types (user_type, type_desc) VALUES
('1', 'normal'),
('2', 'admin'),
('3', 'guest');

-- 타입이 일치하는 경우
EXPLAIN FORMAT=TRADITIONAL
SELECT *
FROM user_logs ul
STRAIGHT_JOIN user_types_int uti ON ul.user_type = uti.user_type;

-- 타입이 불일치하는 경우
EXPLAIN FORMAT=TRADITIONAL
SELECT *
FROM user_logs ul
JOIN user_types ut ON ul.user_type = ut.user_type;
```

실제 실행 결과 1: 타입이 일치하는 경우

```text
table | type   | possible_keys           | key     | ref                             | rows | Extra
------+--------+-------------------------+---------+---------------------------------+------+-------
ul    | ALL    | idx_user_logs_user_type | NULL    | NULL                            | 4    |
uti   | eq_ref | PRIMARY                 | PRIMARY | join_explain_study.ul.user_type | 1    |
```

```text
user_logs.user_type과 user_types_int.user_type은 둘 다 INT인 것을 관찰할 수 있다.
user_logs를 먼저 읽은 뒤, user_types_int는 PRIMARY KEY로 1건씩 찾는 것을 관찰할 수 있다.
type이 eq_ref인 것은 조인 조건으로 매번 최대 1건만 매칭된다는 의미다.
STRAIGHT_JOIN은 SQL에 작성한 순서대로 조인 순서를 고정해서 확인하기 위해 사용했다.
```

실제 실행 결과 2: 타입이 불일치하는 경우

```text
table | type | possible_keys           | key  | ref  | rows | Extra
------+------+-------------------------+------+------+--------------------------------------------
ut    | ALL  | PRIMARY                 | NULL | NULL | 3    |
ul    | ALL  | idx_user_logs_user_type | NULL | NULL | 4    | Using where; Using join buffer (hash join)
```

```text
user_logs.user_type은 INT이고 user_types.user_type은 CHAR(1)인 것을 관찰할 수 있다.
조인 컬럼의 타입이 달라 두 테이블 모두 인덱스를 사용하지 못하는 것을 관찰할 수 있다.
Extra에 Using join buffer 또는 hash join 관련 메시지가 나오는 것을 관찰할 수 있다.
```

예상 해석:

```text
조인 컬럼의 타입이 같으면 한쪽 테이블을 읽고 다른 테이블을 인덱스로 찾을 수 있다.
조인 컬럼의 의미가 같아도 타입이 다르면 비교 과정에서 변환이 필요하다.
이 변환 때문에 인덱스를 효율적으로 사용하지 못할 수 있다.
```

---

## 9.1 타입이 달라도 괜찮은 경우

모든 타입 차이가 문제를 만드는 것은 아니다.

대체로 다음은 큰 문제가 되지 않는다.

```text
CHAR와 VARCHAR
INT와 BIGINT
DATE와 DATETIME
```

하지만 다음은 주의해야 한다.

```text
CHAR와 INT
문자 집합이나 콜레이션이 다른 문자열 컬럼
SIGNED INT와 UNSIGNED INT
```

---

## 9.2 콜레이션이 다른 경우

문자열 컬럼끼리 조인할 때는 문자 집합과 콜레이션도 중요하다.

> 콜레이션이란?
> 
> 콜레이션은 문자열을 비교하고 정렬하는 규칙


```text
ascii 문자 집합의 콜레이션
- ascii_general_ci
- ascii_bin

utf8mb4 문자 집합의 콜레이션
- utf8mb4_0900_ai_ci
- utf8mb4_general_ci
- utf8mb4_bin
```

서로 다른 콜레이션의 컬럼을 비교하면 변환이 필요할 수 있고, 이로 인해 인덱스를 효율적으로 사용하지 못할 수 있다.

### 핵심

> JOIN 컬럼은 데이터 타입, 문자 집합, 콜레이션까지 맞추는 것이 좋다.

---

# 10. INNER JOIN과 OUTER JOIN

## 10.1 INNER JOIN

INNER JOIN은 양쪽 테이블에 매칭되는 데이터만 반환한다.

```sql
SELECT m.name, o.price
FROM members m
INNER JOIN orders o ON o.member_id = m.id;
```

```text
주문이 있는 회원만 결과에 나옴
```

---

## 10.2 LEFT OUTER JOIN

LEFT JOIN은 왼쪽 테이블의 데이터는 모두 살리고, 오른쪽 테이블은 매칭되는 경우에만 붙인다.

```sql
SELECT m.name, o.price
FROM members m
LEFT JOIN orders o ON o.member_id = m.id;
```

```text
주문이 없는 회원도 결과에 나옴
주문 정보는 NULL
```

예시:

```text
members

id | name
---|------
1  | 라이
2  | 봉구스
3  | 티모
```

```text
orders

id | member_id | price
---|-----------|------
1  | 1         | 10000
2  | 2         | 20000
```

LEFT JOIN 결과:

```text
name   | price
-------|------
라이    | 10000
봉구스   | 20000
티모    | NULL
```

---

# 11. OUTER JOIN의 성능과 주의사항

OUTER JOIN은 결과를 보존해야 한다.

LEFT OUTER JOIN에서는 왼쪽 테이블의 결과를 보존해야 하므로, 일반적으로 오른쪽 테이블이 드라이빙 테이블이 되기 어렵다.

이 특성 때문에 옵티마이저가 조인 순서를 자유롭게 바꾸기 어렵다.

---

## 11.1 불필요한 OUTER JOIN은 피하자

다음 쿼리를 보자.

```sql
SELECT *
FROM employees e
LEFT JOIN dept_emp de ON de.emp_no = e.emp_no
LEFT JOIN departments d ON d.dept_no = de.dept_no
                           AND d.dept_name = 'Development';
```

만약 모든 직원이 반드시 부서 정보를 가지고 있고, 부서가 없는 직원을 결과에 포함할 필요가 없다면 LEFT JOIN이 필요하지 않을 수 있다.

이 경우 INNER JOIN을 사용하면 옵티마이저가 더 자유롭게 조인 순서를 선택할 수 있다.

```sql
SELECT *
FROM employees e
JOIN dept_emp de ON de.emp_no = e.emp_no
JOIN departments d ON d.dept_no = de.dept_no
WHERE d.dept_name = 'Development';
```

> OUTER JOIN은 꼭 필요할 때만 사용하자.  
> 불필요한 OUTER JOIN은 옵티마이저의 조인 순서 선택을 제한할 수 있다.

### 실습: OUTER JOIN 때문에 조인 순서가 제한되는 경우

목표:

```text
LEFT JOIN은 왼쪽 테이블의 row를 보존해야 하므로
옵티마이저가 선택도 높은 오른쪽 테이블부터 읽기 어려울 수 있음을 확인한다.
```

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE employees (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(30) NOT NULL,
    PRIMARY KEY (id)
) ENGINE = InnoDB;

CREATE TABLE departments (
    dept_no CHAR(4) NOT NULL,
    dept_name VARCHAR(30) NOT NULL,
    PRIMARY KEY (dept_no),
    INDEX idx_departments_dept_name (dept_name)
) ENGINE = InnoDB;

CREATE TABLE dept_emp (
    emp_id BIGINT NOT NULL,
    dept_no CHAR(4) NOT NULL,
    PRIMARY KEY (emp_id, dept_no),
    INDEX idx_dept_emp_dept_no (dept_no)
) ENGINE = InnoDB;

INSERT INTO employees (name)
WITH RECURSIVE seq(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 1000
)
SELECT CONCAT('employee-', n)
FROM seq;

INSERT INTO departments (dept_no, dept_name) VALUES
('d001', 'Development'),
('d002', 'Sales'),
('d003', 'HR');

INSERT INTO dept_emp (emp_id, dept_no)
WITH RECURSIVE seq(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 1000
)
SELECT
    n,
    CASE
        WHEN n <= 10 THEN 'd001'
        WHEN n <= 500 THEN 'd002'
        ELSE 'd003'
    END
FROM seq;

-- LEFT JOIN: 모든 직원을 보존해야 한다.
EXPLAIN FORMAT=TRADITIONAL
SELECT e.id, e.name, d.dept_name
FROM employees e
LEFT JOIN dept_emp de ON de.emp_id = e.id
LEFT JOIN departments d ON d.dept_no = de.dept_no
                       AND d.dept_name = 'Development';

SELECT e.id, e.name, d.dept_name
FROM employees e
LEFT JOIN dept_emp de ON de.emp_id = e.id
LEFT JOIN departments d ON d.dept_no = de.dept_no
                       AND d.dept_name = 'Development'
ORDER BY e.id
LIMIT 12;

-- INNER JOIN: Development 부서에 속한 직원만 필요하다.
EXPLAIN FORMAT=TRADITIONAL
SELECT e.id, e.name, d.dept_name
FROM employees e
JOIN dept_emp de ON de.emp_id = e.id
JOIN departments d ON d.dept_no = de.dept_no
WHERE d.dept_name = 'Development';

SELECT e.id, e.name, d.dept_name
FROM employees e
JOIN dept_emp de ON de.emp_id = e.id
JOIN departments d ON d.dept_no = de.dept_no
WHERE d.dept_name = 'Development'
ORDER BY e.id;
```

실제 실행 결과 예시 1: LEFT JOIN으로 작성한 경우

```text
EXPLAIN 결과

table | type   | key                       | ref                            | rows | Extra
------+--------+---------------------------+--------------------------------+------+-------------
e     | ALL    | NULL                      | NULL                           | 1000 |
de    | ref    | PRIMARY                   | join_explain_study.e.id        | 1    | Using index
d     | eq_ref | PRIMARY                   | join_explain_study.de.dept_no  | 1    | Using where

SELECT 결과

id | name        | dept_name
---+-------------+------------
1  | employee-1  | Development
2  | employee-2  | Development
3  | employee-3  | Development
4  | employee-4  | Development
5  | employee-5  | Development
6  | employee-6  | Development
7  | employee-7  | Development
8  | employee-8  | Development
9  | employee-9  | Development
10 | employee-10 | Development
11 | employee-11 | NULL
12 | employee-12 | NULL
```

```text
LEFT JOIN은 employees의 모든 row를 보존해야 하므로 employees부터 읽는 것을 관찰할 수 있다.
departments의 dept_name 조건은 선택도가 높지만, 먼저 읽어서 전체 결과를 줄이는 방식으로 사용하기 어렵다.
Development가 아닌 직원도 결과에 남고 dept_name만 NULL이 되는 것을 관찰할 수 있다.
```

실제 실행 결과 예시 2: INNER JOIN으로 작성한 경우

```text
EXPLAIN 결과

table | type   | key                       | ref                            | rows | Extra
------+--------+---------------------------+--------------------------------+------+-------------
d     | ref    | idx_departments_dept_name | const                          | 1    | Using index
de    | ref    | idx_dept_emp_dept_no      | join_explain_study.d.dept_no   | 10   | Using index
e     | eq_ref | PRIMARY                   | join_explain_study.de.emp_id   | 1    |

SELECT 결과

id | name        | dept_name
---+-------------+------------
1  | employee-1  | Development
2  | employee-2  | Development
3  | employee-3  | Development
4  | employee-4  | Development
5  | employee-5  | Development
6  | employee-6  | Development
7  | employee-7  | Development
8  | employee-8  | Development
9  | employee-9  | Development
10 | employee-10 | Development
```

```text
INNER JOIN은 departments의 dept_name 조건을 먼저 사용해 Development 부서부터 읽는 것을 관찰할 수 있다.
그 다음 dept_emp의 dept_no 인덱스로 해당 부서의 직원만 찾는 것을 관찰할 수 있다.
보존해야 할 왼쪽 row가 없기 때문에 옵티마이저가 더 자유롭게 조인 순서를 선택할 수 있다.
```

예상 해석:

```text
LEFT JOIN은 왼쪽 테이블의 모든 row를 결과에 남겨야 한다.
그래서 오른쪽 테이블에 선택도 높은 조건이 있어도, 그 테이블을 먼저 읽어 전체 결과를 줄이기 어렵다.
반면 INNER JOIN은 어느 테이블의 row도 보존할 필요가 없으므로 선택도 높은 조건을 가진 테이블부터 읽을 수 있다.
부서가 없는 직원이나 다른 부서 직원을 결과에 포함할 필요가 없다면 INNER JOIN으로 작성하는 것이 더 명확하다.
```

---

## 11.2 LEFT JOIN 오른쪽 테이블 조건을 WHERE에 쓰는 실수(그런갑다)

다음 쿼리를 보자.

```sql
SELECT *
FROM employees e
LEFT JOIN dept_manager dm ON dm.emp_no = e.emp_no
WHERE dm.dept_no = 'd001';
```

`LEFT JOIN`은 왼쪽 테이블인 `employees`의 row를 보존한다.

하지만 오른쪽 테이블인 `dept_manager`에 매칭되는 row가 없으면 `dm` 쪽 컬럼은 `NULL`이 된다.

```text
e.emp_no | dm.dept_no
---------|-----------
10001    | d001
10002    | NULL
10003    | d002
```

이 상태에서 다음 조건이 적용된다.

```sql
WHERE dm.dept_no = 'd001'
```

그러면 `dm.dept_no`가 `NULL`인 row는 조건을 통과하지 못한다.

결국 `LEFT JOIN`으로 살려둔 row가 `WHERE`에서 제거되어, 결과적으로 `INNER JOIN`처럼 동작할 수 있다.


### 실습: LEFT JOIN 조건을 WHERE에 둔 경우

목표:

```text
LEFT JOIN 오른쪽 테이블 조건을 WHERE에 두면
LEFT JOIN으로 살린 row가 제거될 수 있음을 확인한다.
```

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE members (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(30) NOT NULL,
    PRIMARY KEY (id)
) ENGINE = InnoDB;

CREATE TABLE coupons (
    id BIGINT NOT NULL AUTO_INCREMENT,
    member_id BIGINT NOT NULL,
    coupon_name VARCHAR(30) NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_coupons_member_id (member_id)
) ENGINE = InnoDB;

INSERT INTO members (name) VALUES
('라이'),
('봉구스'),
('티모');

INSERT INTO coupons (member_id, coupon_name) VALUES
(1, 'WELCOME'),
(2, 'EVENT');

EXPLAIN FORMAT=TRADITIONAL
SELECT m.id, m.name, c.coupon_name
FROM members m
LEFT JOIN coupons c ON c.member_id = m.id
WHERE c.coupon_name = 'WELCOME';

SELECT m.id, m.name, c.coupon_name
FROM members m
LEFT JOIN coupons c ON c.member_id = m.id
WHERE c.coupon_name = 'WELCOME';
```

실제 실행 결과(핵심 컬럼):

```text
EXPLAIN 결과

id | select_type | table | type   | possible_keys         | key     | ref                            | rows | Extra
---+-------------+-------+--------+-----------------------+---------+--------------------------------+------+-------------
1  | SIMPLE      | c     | ALL    | idx_coupons_member_id | NULL    | NULL                           | 2    | Using where
1  | SIMPLE      | m     | eq_ref | PRIMARY              | PRIMARY | join_explain_study.c.member_id | 1    |

SELECT 결과

id | name | coupon_name
---+------+------------
1  | 라이 | WELCOME
```

```text
결과에 쿠폰이 없는 티모가 남지 않는 것을 관찰할 수 있다.
LEFT JOIN을 사용했지만 WHERE 조건 때문에 NULL row가 제거되는 것을 관찰할 수 있다.
```

예상 결과:

```text
라이만 조회된다.

쿠폰이 없는 티모는 LEFT JOIN으로 살려졌지만,
WHERE c.coupon_name = 'WELCOME' 조건에서 제거된다.
```

---

## 11.3 조건을 ON으로 옮기기(그런갑다)

정말 OUTER JOIN의 의미를 유지하고 싶다면 오른쪽 테이블 조건을 ON 절에 두는 것이 좋다.

```sql
SELECT *
FROM employees e
LEFT JOIN dept_manager dm
  ON dm.emp_no = e.emp_no
 AND dm.dept_no = 'd001';
```

이렇게 하면 employees의 row는 유지된다.

단, `d001` 조건에 맞는 dept_manager가 있을 때만 오른쪽 정보가 붙는다.


> LEFT JOIN에서 오른쪽 테이블 조건을 WHERE에 쓰면 OUTER JOIN의 의미가 깨질 수 있다.  
> 보존하고 싶은 테이블이 있다면 조건을 ON 절에 둘지 WHERE 절에 둘지 구분해야 한다.


### 실습: LEFT JOIN 조건을 ON에 둔 경우

목표:

```text
오른쪽 테이블 조건을 ON에 두면
왼쪽 테이블 row가 보존되는 것을 확인한다.
```

실행 SQL:

```sql
DROP DATABASE IF EXISTS join_explain_study;
CREATE DATABASE join_explain_study;
USE join_explain_study;

CREATE TABLE members (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(30) NOT NULL,
    PRIMARY KEY (id)
) ENGINE = InnoDB;

CREATE TABLE coupons (
    id BIGINT NOT NULL AUTO_INCREMENT,
    member_id BIGINT NOT NULL,
    coupon_name VARCHAR(30) NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_coupons_member_id (member_id)
) ENGINE = InnoDB;

INSERT INTO members (name) VALUES
('라이'),
('봉구스'),
('티모');

INSERT INTO coupons (member_id, coupon_name) VALUES
(1, 'WELCOME'),
(2, 'EVENT');

EXPLAIN FORMAT=TRADITIONAL
SELECT m.id, m.name, c.coupon_name
FROM members m
LEFT JOIN coupons c
 ON c.member_id = m.id
 AND c.coupon_name = 'WELCOME';

SELECT m.id, m.name, c.coupon_name
FROM members m
LEFT JOIN coupons c
  ON c.member_id = m.id
 AND c.coupon_name = 'WELCOME';
```

실제 실행 결과(핵심 컬럼):

```text
EXPLAIN 결과

id | select_type | table | type  | possible_keys         | key                   | ref                            | rows | Extra
---+-------------+-------+-------+-----------------------+-----------------------+--------------------------------+------+-------------
1  | SIMPLE      | m     | ALL   | NULL                  | NULL                  | NULL                           | 3    |
1  | SIMPLE      | c     | ref   | idx_coupons_member_id | idx_coupons_member_id | join_explain_study.m.id        | 1    | Using where

SELECT 결과

id | name   | coupon_name
---+--------+------------
1  | 라이   | WELCOME
2  | 봉구스 | NULL
3  | 티모   | NULL
```

```text
members의 모든 row가 결과에 남는 것을 관찰할 수 있다.
WELCOME 쿠폰이 없는 회원은 coupon_name이 NULL인 것을 관찰할 수 있다.
```

예상 결과:

```text
라이     WELCOME
봉구스   NULL
티모     NULL
```

### 핵심

> `WHERE` 조건은 최종 결과 row를 제거한다.  
> `ON` 조건은 오른쪽 테이블을 붙일지 말지를 결정한다.

---

## 11.4 예외: 안티 조인(그런갑다)

OUTER JOIN된 테이블의 컬럼을 WHERE에 쓰는 것이 항상 잘못은 아니다.

대표적인 예외는 안티 조인이다.

```sql
SELECT *
FROM employees e
LEFT JOIN dept_manager dm ON dm.emp_no = e.emp_no
WHERE dm.emp_no IS NULL;
```

이 쿼리는 다음 의미다.

```text
매니저가 아닌 직원만 찾기
```

즉, 오른쪽 테이블에 매칭되는 row가 없는 경우만 찾는 것이다.

이런 경우에는 `WHERE dm.emp_no IS NULL`이 의도된 사용이다.

---

# 12. JOIN과 외래키

외래키가 있어야만 JOIN할 수 있다고 오해하는 경우가 있다.

하지만 외래키와 JOIN은 직접적인 필수 관계가 없다.

```sql
SELECT *
FROM orders o
JOIN members m ON m.id = o.member_id;
```

이 쿼리는 외래키가 없어도 실행된다.

외래키의 목적은 JOIN이 아니라 참조 무결성 보장이다.

```text
orders.member_id 값은 반드시 members.id에 존재해야 한다.
```

이런 규칙을 DB 레벨에서 보장하려고 외래키를 사용한다.


---

# 13. 지연된 조인

지연된 조인은 JOIN을 나중으로 미루는 최적화 패턴이다.

특히 `ORDER BY`, `GROUP BY`, `LIMIT`이 있는 쿼리에서 효과가 있을 수 있다.

---

## 13.1 일반적인 처리

다음과 같은 쿼리가 있다고 하자.

```sql
SELECT e.*
FROM salaries s
JOIN employees e ON e.emp_no = s.emp_no
WHERE s.emp_no BETWEEN 10001 AND 13000
GROUP BY s.emp_no
ORDER BY SUM(s.salary) DESC
LIMIT 10;
```

일반적인 흐름은 다음과 같을 수 있다.

```text
1. salaries와 employees를 먼저 조인한다.
2. 조인 결과를 GROUP BY 한다.
3. ORDER BY 한다.
4. LIMIT 10을 적용한다.
```

문제는 조인을 먼저 하면 중간 결과가 커질 수 있다는 점이다.

```text
조인 많이 함
→ GROUP BY
→ ORDER BY
→ LIMIT 10
```

---

## 13.2 지연된 조인 방식

지연된 조인은 보통 개발자가 쿼리를 직접 바꿔서 사용하는 최적화 패턴이다.

먼저 필요한 결과를 줄이고, 마지막에 조인한다.

```sql
SELECT e.*
FROM (
    SELECT s.emp_no
    FROM salaries s
    WHERE s.emp_no BETWEEN 10001 AND 13000
    GROUP BY s.emp_no
    ORDER BY SUM(s.salary) DESC
    LIMIT 10
) x
JOIN employees e ON e.emp_no = x.emp_no;
```

흐름:

```text
1. salaries에서 먼저 GROUP BY, ORDER BY, LIMIT 10 처리
2. 최종 10건의 emp_no만 남김
3. 그 10건만 employees와 조인
```

비교:

```text
일반 조인
→ 많은 row를 조인한 뒤 줄임

지연된 조인
→ 먼저 줄이고, 적은 row만 조인
```

---

## 13.3 지연된 조인의 주의사항

지연된 조인은 아무 쿼리에나 적용하면 안 된다.

결과가 바뀌지 않는다는 보장이 있어야 한다.

특히 `LIMIT`을 먼저 적용해도 최종 결과가 동일한지 확인해야 한다.

```text
잘 쓰면 빠르다.
잘못 쓰면 결과가 달라질 수 있다.
```

---

# 14. 심화: 래터럴 조인(그런갑다)

래터럴 조인은 서브쿼리가 바깥쪽 테이블의 컬럼을 참조할 수 있게 하는 조인이다.

예를 들어 직원별 최근 급여 2개를 가져오고 싶다고 하자.

```sql
SELECT *
FROM employees e
LEFT JOIN LATERAL (
    SELECT *
    FROM salaries s
    WHERE s.emp_no = e.emp_no
    ORDER BY s.from_date DESC
    LIMIT 2
) s2 ON s2.emp_no = e.emp_no
WHERE e.first_name = 'Matt';
```

일반적인 FROM 절 서브쿼리는 바깥쪽 테이블의 컬럼을 참조할 수 없다.

하지만 `LATERAL`을 사용하면 가능하다.

```text
employees에서 한 명 읽기
→ 그 직원의 emp_no를 이용해 salaries에서 최근 2건 조회
```

### 핵심

> 래터럴 조인은 “각 row마다 실행되는 서브쿼리 조인”처럼 이해할 수 있다.  
> 편리하지만 내부적으로 반복 실행과 임시 테이블 비용이 생길 수 있으므로 꼭 필요할 때 사용한다.

---

# 15. 실행 계획으로 인한 정렬 흐트러짐

정렬이 필요한 결과라면 반드시 ORDER BY를 명시해야 한다.

예전에는 Nested Loop Join의 특성 때문에 드라이빙 테이블을 읽은 순서가 결과에도 유지되는 것처럼 보이는 경우가 있었다.

하지만 MySQL 8.0부터 해시 조인 같은 다른 조인 방식이 사용될 수 있다.

이 경우 결과 순서가 달라질 수 있다.

```text
Nested Loop Join
→ 드라이빙 테이블 순서가 유지되는 것처럼 보일 수 있음

Hash Join
→ 결과 순서를 예측하기 어려움
```
