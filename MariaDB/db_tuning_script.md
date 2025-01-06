# mariadb로 진행한 쿼리 튜닝 기법

```sql
select emp.EMP_ID , emp.FIRST_NAME , emp.LAST_NAME , grade.GRADE_NAME 
from GRADE grade, EMP emp
WHERE grade.GRADE_NAME = 'Staff'
AND grade.EMP_ID = emp.EMP_ID ;

SHOW VARIABLES LIKE 'join_buffer_size';
SET SESSION join_buffer_size = 16 * 1024 * 1024; -- 16MB로 설정

select emp.EMP_ID , emp.FIRST_NAME , emp.LAST_NAME , grade.GRADE_NAME 
from GRADE grade, EMP emp
WHERE grade.EMP_ID = emp.EMP_ID ;

-- table full scan

-- index range scan

-- index unique scan
select * from EMP where emp_id = 20000;
-- index full scan
select last_name from EMP where gender <> 'F';
-- index loose scan
-- 인덱스의 필요한 부분만 골라 스캔하는 방식
-- where 절 조건문을 기준으로 필요한 부분과 그렇지 않은 부분을 구분하여 선택적으로 스캔
-- group by, max, min
SELECT gender, count(distinct last_name) cnt
from EMP e 
where gender = 'F'
group by gender;

-- index skip scan
-- 조건절에 인덱스의 선두 컬럼이 없어도, 선두 컬럼을 뛰어넘어 스캔을 수행하는 방식
SELECT MAX(EMP_ID) max_emp_id
FROM EMP e 
WHERE LAST_NAME = 'Peha';

-- index merge scan
-- 생성된 2개의 인덱스를 통합한 후, 테이블을 스캔하는 방식
-- where 절 조건문에 서로 다른 인덱스를 구성하는 컬럼들로 구성되어 있는 경우
-- 인덱스의 결합(union) 또는 교차 (intersection)
SELECT EMP_ID , LAST_NAME , FIRST_NAME 
FROM EMP e 
WHERE (HIRE_DATE BETWEEN '1989-01-01' and '1989-06-30')
or emp_id > 600000;


-- mariadb hint
SELECT e.FIRST_NAME, e.LAST_NAME 
FROM EMP e,
	MANAGER m 
WHERE e.EMP_ID = m.EMP_ID ;

SELECT /*! STRAIGHT_JOIN */ e.FIRST_NAME, e.LAST_NAME
FROM EMP e
JOIN MANAGER m ON e.EMP_ID = m.EMP_ID;

SELECT e.FIRST_NAME, e.LAST_NAME
FROM EMP e
JOIN MANAGER m USE INDEX (PRIMARY) ON e.EMP_ID = m.EMP_ID;

SELECT e.FIRST_NAME, e.LAST_NAME 
FROM EMP e,
	MANAGER m USE INDEX (PRIMARY)
WHERE e.EMP_ID = m.EMP_ID ;


SELECT e.FIRST_NAME, e.LAST_NAME
FROM EMP e
JOIN MANAGER m FORCE INDEX (PRIMARY) ON e.EMP_ID = m.EMP_ID;

EXPLAIN SELECT e.FIRST_NAME, e.LAST_NAME 
FROM EMP e,
	MANAGER m FORCE INDEX (PRIMARY)
WHERE e.EMP_ID = m.EMP_ID ;

SELECT e.FIRST_NAME, e.LAST_NAME
FROM EMP e
JOIN MANAGER m USE INDEX (PRIMARY) ON e.EMP_ID = m.EMP_ID;

SELECT EMP_ID, COUNT(*) 
FROM MANAGER 
GROUP BY EMP_ID;

EXPLAIN SELECT e.FIRST_NAME, e.LAST_NAME
FROM EMP e
JOIN MANAGER m ON e.EMP_ID = m.EMP_ID;

create table coll_table (
	bin_col VARCHAR(10) NOT NULL,
	ci_col VARCHAR(10) NOT NULL COLLATE 'utf8mb3_general_ci'
)
COLLATE = 'utf8mb3_bin'
ENGINE = INNODB;

INSERT INTO `tuning`.`coll_table` (`bin_col`, `ci_col`)
VALUES('A', 'A'), ('B', 'a'), ('a', 'B'), ('b', 'b');

SELECT * FROM coll_table
order by bin_col, ci_col;

Drop table coll_table;

-- 데이터베이스 오브젝트(테이블, 인덱스 등)에 대한 특징을 수집한 정보
-- mysql.innodb_table_stats
-- mysql.innodb_index_stats
SELECT * FROM mysql.innodb_table_stats
where database_name = 'tuning';

SELECT * FROM mysql.innodb_index_stats
where database_name = 'tuning';

-- 실행 계획 수립
-- explain 쿼리문
-- describe 쿼리문
-- desc 쿼리문
explain select count(1) from DEPT;

explain SELECT e.emp_id, e.first_name, e.last_name, s.annual_salary,
	(SELECT g.grade_name from GRADE g
	  WHERE g.emp_id = e.emp_id
	    AND g.end_date = '9999-01-01') AS grade_name
FROM EMP e, SALARY s 
WHERE e.EMP_ID = 10001
  AND e.EMP_ID = s.EMP_ID
  AND s.IS_YN = 1;
-- id: 최소한의 단위 select 문마다 부여되는 식별자
-- id가 같다는 것은 조인이 이루어졌다는 뜻
 
-- table: 최소한의 단위 Select 문마다 부여되는 식별자
 
-- select_type: 쿼리문의 Select 유형을 나타내는 항목
-- simple: 서브쿼리 또는 Union 구문이 없는 단순한 Select문
EXPLAIN
select e.emp_id, e.first_name, e.last_name, s.annual_salary
from EMP e, SALARY s 
WHERE e.EMP_ID = s.EMP_ID
AND e.emp_id BETWEEN 10001 and 10010
and s.annual_salary > 80000;
-- primary: 서브쿼리 또는 Union 구문이 포함된 쿼리문에서 최초 접근한 테이블
EXPLAIN 
select emp_id, first_name, last_name,
	(select MAX(annual_salary)
	   from SALARY
	  WHERE emp_id = e.emp_id
	) as max_salary
from EMP e
WHERE emp_id = 10001;

-- subquery: 독립적인 서브쿼리
EXPLAIN
SELECT (SELECT COUNT(*) FROM EMP) AS e_count,
	   (SELECT MAX(annual_salary) FROM SALARY) AS s_max; 
	 
-- derieved: 단위 쿼리를 메모리나 디스크에 생성한 임시 테이블
-- <derived2> means id 2번을 봐라
EXPLAIN
SELECT e.emp_id, s.annual_salary
FROM EMP e ,
	(SELECT emp_id, MAX(annual_salary) annual_salary
	   FROM SALARY
	  WHERE emp_id = 10001) s
WHERE e.emp_id = s.emp_id;

-- union: Union 또는 Union all 구문에서 첫 번째 이후의 테이블
explain
select gender, max(hire_date) hire_date
from EMP e1
where gender = 'M'
UNION ALL 
select gender, max(hire_date) hire_date
from EMP e2
where gender = 'F';

-- union result: Union 구문에서 중복을 제거하기 위해 메모리나 디스크에 생성한 임시 테이블
explain
select gender, max(hire_date) hire_date
from EMP e1
where gender = 'M'
UNION -- 합집합(중복 제거)
select gender, max(hire_date) hire_date
from EMP e2
where gender = 'F';

-- dependent subquery, dependent union: Union 또는 Union all 구문에서 메인 테이블의 영향을 받는 테이블
explain select m.dept_id,
	  (select concat(gender,' : ',last_name)
	     from EMP e1
	    where gender = 'F' AND e1.EMP_ID = m.EMP_ID
	    UNION ALL 
	    select concat(gender,' : ',last_name)
	     from EMP e2
	    where gender = 'M' AND e2.EMP_ID = m.EMP_ID
	  ) name
from MANAGER m;

-- materialized: 조인 등의 가공 작업을 위해서 생성한 임시 테이블
explain
select *
from EMP e 
where emp_id in (select emp_id from SALARY where start_date > '2020-01-01');

-- paritions: 접근하게 되는 특정 파티션

-- type: 데이터를 어떻게 접근할 것인가에 관한 정보
-- const: 단 1건의 데이터만 접근하는 유형
explain
select * from EMP e where emp_id = 10001;

-- eq_ref: 조인 시 드리븐 테이블에서 매번 단 1건의 데이터만 접근하는 유형
explain
select d.dept_id, d.dept_name
from DEPT_EMP_MAPPING de,
	 DEPT d 
where de.dept_id = d.dept_id
and de.end_date = '9999-01-01'
and de.emp_id = 10001;

-- ref: 데이터 접근 범위가 2개 이상인 유형
EXPLAIN
SELECT *
FROM DEPT_EMP_MAPPING DE
WHERE DE.END_DATE = '9999-01-01' AND DE.EMP_ID = 10001;

EXPLAIN
SELECT D.DEPT_ID, D.DEPT_NAME
FROM DEPT_EMP_MAPPING DE, DEPT D
WHERE DE.DEPT_ID = D.DEPT_ID AND DE.END_DATE = '9999-01-01' AND DE.EMP_ID = 10001;

-- range: 연속되는 범위를 접근하게 되는 유형
EXPLAIN
SELECT * FROM EMP WHERE EMP_ID BETWEEN 10001 AND 100000;

-- index_merge: 특정 테이블에 생성된 2개 이상 인덱스가 병합되어 동시에 적용되는 유형
EXPLAIN
SELECT *
FROM EMP
WHERE EMP_ID BETWEEN 10001 AND 100000
AND HIRE_DATE = '1985-11-21';

-- index: 인덱스를 처음부터 끝까지 접근하는 유형(Full index scan)
EXPLAIN
SELECT EMP_ID
FROM GRADE
WHERE GRADE_NAME = 'Manager';

-- all: 테이블을 처음부터 끝까지 접근하는 유형(Full table scan)
EXPLAIN
SELECT EMP_ID, FIRST_NAME
FROM EMP;

-- possible_keys: 사용할 수 있는 인덱스 후보군

-- key: 사용한 인덱스명
explain select emp_id from EMP where emp_id BETWEEN 100000 and 110000;

-- key_len: 사용된 인덱스의 bytes

-- ref: 테이블을 접근한 조건(Reference 약자)

-- rows: 접근할 레코드 행 수(예상 수치)

-- filtered: MYSQL 엔진에서 필터 조건에 의해 최종 반환되는 비율(%)

-- extra: 쿼리문을 어떻게 수행할 것인지에 대한 부가 정보
-- distinct: 중복이 제거되어 유일한 값을 찾을 때 출력되는 정보(distinct, union)
-- Using where: Where 절의 필터 조건을 사용해 Mysql 엔진으로 가져온 데이터를 추출
-- Using temporary: 데이터의 중간 결과를 저장하고자 임시 테이블을 생성
-- Using index: 물리적인 데이터 파일을 읽지 않고 인덱스만 읽어 쿼리 수행(Covering index)
-- Covering index: 쿼리문에서 인덱스에 구성된 컬럼만 사용하는 방식
EXPLAIN 
SELECT emp_id
from EMP e 
where emp_id BETWEEN 10001 and 10100;

-- 실행 계획의 판단 기준
-- select_type 항목: simple, primary <-> dependent, uncacheable
-- type 항목: system, const, eq_ref <-> index, all
-- extra 항목: Using index: Using filesort, Using temporary
-- Using filesort: order by 가 인덱스 활용하지 못하고, 메모리에 올려서 추가적인 정렬 작업 수행
-- Using index for group-by: 쿼리문에 Group by 구문이나 Distinct 구문이 포함될 때, 정렬된 인덱스를 순서대로 읽으면서 group by 연산 수행
-- Using index for skip scan: 인덱스의 모든 값을 비교하는 것이 아닌, 필요한 값만 건너뛰면서 스캔하는 방식
-- FirstMatch: 인덱스 스캔 시에 첫 번째로 일치하는 레코드만 찾으면 검색을 중단하는 방식


EXPLAIN FORMAT=traditional
SELECT emp_id
from EMP e 
where emp_id BETWEEN 10001 and 10100;

EXPLAIN FORMAT=tree
SELECT emp_id
from EMP e 
where emp_id BETWEEN 10001 and 10100;

EXPLAIN FORMAT=json
SELECT emp_id
from EMP e 
where emp_id BETWEEN 10001 and 10100;

EXPLAIN analyze
SELECT emp_id
from EMP e 
where emp_id BETWEEN 10001 and 10100;


-- SQL 프로파일링: 이벤트별 소요시간이 포함된 세부적인 실행정보를 제공하는 성능분석 도구

show variables like 'profiling';

set profiling = 'ON';

show profiles;

select count(1) FROM EMP;

show profile for query 5;

show profile all for query 5;

-- OOO를 변형하는 나쁜 SQL
explain
select * 
from EMP e 
where SUBSTRING(emp_id, 1, 4) = 1100
and length(emp_id) = 5;
-- 1100 ? 11000 ~ 11009

-- 개선된 쿼리
explain
select * 
from EMP e 
where EMP_ID BETWEEN 11000 AND 11009;

show index from EMP;
-- PRIMARY: EMP_ID
-- I_HIRE_DATE: HIRE_DATE
-- I_GENDER_LAST_NAME : GENDER + LAST_NAME

-- 불필요한 OO를 포함하는 나쁜 SQL

-- using index, using temporary 
EXPLAIN
SELECT IFNULL(gender, 'NO DATA') gender, 
	   COUNT(1)
FROM EMP e 
GROUP BY IFNULL(gender, 'NO DATA');

-- using index
-- gender는 not null
EXPLAIN
SELECT gender, 
	   COUNT(1)
FROM EMP e 
GROUP BY gender;

-- OOO를 활용하지 못하는 나쁜 SQL
-- 인덱스
EXPLAIN
SELECT COUNT(*) count
FROM SALARY
WHERE IS_YN = 1;

EXPLAIN
SELECT COUNT(*) count
FROM SALARY
WHERE IS_YN = '1';

-- OOO 방식으로 수행하는 나쁜 SQL
EXPLAIN
SELECT FIRST_NAME, LAST_NAME
FROM EMP e
WHERE HIRE_DATE LIKE '1994%';

-- like -> between 앞 뒤 범위가 정해져 있는 경우
EXPLAIN
SELECT FIRST_NAME, LAST_NAME
FROM EMP e
WHERE HIRE_DATE BETWEEN '1994-01-01' and '1994-12-31';

-- OO을 결합해서 사용하는 나쁜 SQL
-- 컬럼을 결합해서 사용하는 나쁜 SQL
EXPLAIN
SELECT * 
FROM EMP e 
WHERE CONCAT(gender, ' ', last_name) = 'M Radwan';

EXPLAIN
SELECT * 
FROM EMP e 
WHERE gender = 'M' and last_name = 'Radwan';

-- 습관적으로 OO을 제거하는 나쁜 SQL
-- 습관적으로 중복을 제거하는 나쁜 SQL
explain
SELECT DISTINCT e.EMP_ID , e.FIRST_NAME , e.LAST_NAME , s.ANNUAL_SALARY 
FROM EMP e
JOIN SALARY s 
  ON (e.emp_id = s.emp_id)
WHERE s.IS_YN = '1';

explain
SELECT e.EMP_ID , e.FIRST_NAME , e.LAST_NAME , s.ANNUAL_SALARY 
FROM EMP e
JOIN (SELECT * FROM SALARY WHERE IS_YN = '1') s 
  ON (e.emp_id = s.emp_id);
  
 
-- OOOOO 문으로 쿼리를 합치는 나쁜 SQL
explain
SELECT 'M' AS gender, emp_id
from EMP
WHERE gender = 'M'
AND LAST_NAME = 'Baba'
UNION 
SELECT 'F', emp_id
FROM EMP
WHERE gender = 'F'
AND last_name = 'Baba';


explain
SELECT gender, emp_id
from EMP
WHERE gender = 'M'
AND LAST_NAME = 'Baba'
UNION ALL
SELECT gender, emp_id
FROM EMP
WHERE gender = 'F'
AND last_name = 'Baba';

-- OOO를 생각하지 않고 작성한 나쁜 SQL
-- 인덱스를 생각하지 않고 작성한 나쁜 SQL
EXPLAIN
SELECT LAST_NAME, GENDER, COUNT(1) as count
from EMP e 
group by last_name, gender;


EXPLAIN
SELECT LAST_NAME, GENDER, COUNT(1) as count
from EMP e 
group by gender, last_name;

-- 엉뚱한 OOO를 사용하는 나쁜 SQL
-- 엉뚱한 인덱스를 사용하는 나쁜 SQL
-- index full scan
EXPLAIN
SELECT emp_id
FROM EMP
WHERE HIRE_DATE LIKE '1989%'
AND EMP_ID > 10000;

-- 개선
-- 데이터의 수량을 비교하고, hire_date로 필터링 하면 좋겠다는 생각
-- date 형식의 hire_date를 like 구문에서 between으로 변경하여 index range scan이 가능하게 변경
EXPLAIN 
SELECT emp_id
FROM EMP
WHERE HIRE_DATE BETWEEN '1989-01-01' AND '1989-12-31'
AND EMP_ID > 10000;

-- hire_date 조건이 더 많은 storage filter를 가능하게 함
SELECT count(*)
from EMP
WHERE hire_date like '1989%'; -- 2,800건 (10%)

SELECT COUNT(*)
From EMP e 
WHERE EMP_ID > 10000; -- 300,024 만건 (70%)

-- 잘못된 OOOO 테이블로 수행되는 나쁜 SQL
-- 잘못된 드라이빙 테이블로 수행되는 나쁜 SQL
-- 예상: DE 테이블이 ALL(table full scan) 진행
EXPLAIN
SELECT de.emp_id, d.dept_id
from DEPT_EMP_MAPPING de,
	 DEPT d
WHERE de.dept_id = d.dept_id
and de.start_date >= '2002-03-01';

SELECT de.emp_id, d.dept_id
from DEPT_EMP_MAPPING de,
	 DEPT d
WHERE de.dept_id = d.dept_id
and de.start_date >= '2002-03-01';

-- 개선
EXPLAIN
SELECT STRAIGHT_JOIN de.emp_id, d.dept_id
from DEPT_EMP_MAPPING de,
	 DEPT d
WHERE de.dept_id = d.dept_id
and de.start_date >= '2002-03-01';

SELECT STRAIGHT_JOIN de.emp_id, d.dept_id
from DEPT_EMP_MAPPING de,
	 DEPT d
WHERE de.dept_id = d.dept_id
and de.start_date >= '2002-03-01';

-- 악성 SQL문 튜닝으로 전문가 되기
-- 불필요한 OO을 수행하는 나쁜 SQL
-- 불필요한 조인을 수행하는 나쁜 SQL
EXPLAIN
SELECT COUNT(DISTINCT e.emp_id) as COUNT
FROM EMP e,
	(SELECT emp_id
	   FROM ENTRY_RECORD
	  WHERE gate = 'A') record
WHERE e.emp_id = record.emp_id; -- 523ms

EXPLAIN
SELECT COUNT(*) as COUNT
FROM EMP e
WHERE exists (
	 SELECT 1
	   FROM ENTRY_RECORD
	  WHERE gate = 'A'
	    AND e.emp_id = emp_id
); -- 375ms

-- OO 절로 추가적 필터를 수행하는 나쁜 SQL
-- Having 절로 추가적 필터를 수행하는 나쁜 SQL
EXPLAIN
SELECT e.emp_id, e.first_name, e.last_name
FROM EMP e,
	 SALARY s 
WHERE e.emp_id > 450000
  AND e.emp_id = s.emp_id
GROUP BY s.emp_id
having max(s.ANNUAL_SALARY) > 100000;

EXPLAIN
SELECT e.emp_id, e.first_name, e.last_name
FROM EMP e
WHERE e.emp_id > 450000
  AND (SELECT MAX(annual_salary)
         FROM SALARY s 
        WHERE s.emp_id = e.emp_id) > 100000;

-- 유사한 OOO문을 여러 개 나열한 나쁜 SQL
-- 유사한 select 문을 여러 개 나열한 나쁜 SQL
EXPLAIN
SELECT 'BOSS' grade_name, COUNT(*) cnt
  FROM GRADE
 WHERE grade_name = 'Manager' and end_date = '9999-01-01'
UNION ALL
SELECT 'TL' grade_name, COUNT(*) cnt
  FROM GRADE
 WHERE grade_name = 'Technique Leader' and end_date = '9999-01-01'
UNION ALL 
SELECT 'AE' grade_name, COUNT(*) cnt
  FROM GRADE
 WHERE grade_name = 'Assistant Engineer' and end_date = '9999-01-01'; -- 309ms

EXPLAIN
SELECT 
	case when grade_name = 'Manager' then 'BOSS'
		 when grade_name = 'Technique Leader' then 'TL'
		 when grade_name = 'Assistant Engineer' then 'AE'
		 END AS grade_name, count(*) cnt
FROM GRADE
WHERE GRADE_NAME IN ('Manager', 'Technique Leader', 'Assistant Engineer')
  AND end_date = '9999-01-01'
GROUP BY grade_name; -- 106ms

-- 소계/통계를 위한 쿼리를 OO하는 나쁜 쿼리
-- 소계/통계를 위한 쿼리를 반복하는 나쁜 쿼리
-- EXPLAIN
SELECT REGION, null GATE, count(*) cnt
FROM ENTRY_RECORD
WHERE REGION <> ''
GROUP BY REGION 
UNION ALL
SELECT REGION, GATE, count(*) cnt
FROM ENTRY_RECORD
WHERE REGION <> ''
GROUP BY REGION, GATE  
UNION ALL
SELECT null region, null GATE, count(*) cnt
FROM ENTRY_RECORD
WHERE REGION <> ''; -- 1.243s

-- 개선점: 쓸데 없는 세번의 테이블 접근인거 같은데 ...
-- 구하려고 하는 것은? region 별로 count 하는 것
-- 첫번째 쿼리는 region이 비어있지 않고 region만 구하는?
-- 두번째 쿼리는 region이 비어있지 않고 region, gate로 묶어서 개수 카운트
-- 세번째 쿼리는 total 개수 카운트하는 쿼리

-- ROLLUP 이라는 함수를 활용해보자
-- 점점 확대해가는 방식
-- mariadb 문법이 살짝 다름
-- mysql group by rollup(region, gate)
-- mariadb group by region, gate with rollup
SELECT REGION, GATE, COUNT(*) AS cnt
FROM ENTRY_RECORD
WHERE REGION <> ''
GROUP BY REGION, GATE WITH ROLLUP; -- 414ms

SELECT VERSION();

RESET QUERY CACHE;
FLUSH TABLES;

-- 처음부터 OO OOO를 가져오는 나쁜 SQL
-- 처음부터 모든 데이터를 가져오는 나쁜 SQL
EXPLAIN
SELECT e.emp_id
	 , s.avg_salary
	 , s.max_salary
	 , s.min_salary
FROM EMP e,
	(select emp_id,
			ROUND(AVG(annual_salary), 0) avg_salary,
			ROUND(MAX(annual_salary), 0) max_salary,
			ROUND(MIN(annual_salary), 0) min_salary
	   from SALARY
	  GROUP BY emp_id) s
WHERE e.emp_id = s.emp_id
  AND e.emp_id BETWEEN 10001 and 10100; -- 26ms
  
-- 개선 쿼리 예상
-- 이게 제일 빠른데?
EXPLAIN
SELECT e.emp_id
	 , s.avg_salary
	 , s.max_salary
	 , s.min_salary
FROM EMP e,
	(select emp_id,
			ROUND(AVG(annual_salary), 0) avg_salary,
			ROUND(MAX(annual_salary), 0) max_salary,
			ROUND(MIN(annual_salary), 0) min_salary
	   from SALARY
	  where emp_id BETWEEN 10001 and 10100
	  GROUP BY emp_id) s
WHERE e.emp_id = s.emp_id; -- 3ms

EXPLAIN
SELECT e.emp_id
	 , (SELECT ROUND(AVG(annual_salary), 0)
	 	  FROM SALARY as s1
	 	 WHERE emp_id = e.emp_id
	   ) AS avg_salary 
	 , (SELECT ROUND(MAX(annual_salary), 0)
	 	  FROM SALARY as s2
	 	 WHERE emp_id = e.emp_id
	   ) AS max_salary 
	 , (SELECT ROUND(MIN(annual_salary), 0)
	 	  FROM SALARY as s3
	 	 WHERE emp_id = e.emp_id
	   ) AS min_salary 
 FROM EMP e
WHERE e.emp_id BETWEEN 10001 and 10100; -- 7ms

-- 비효율적인 OOO을 수행하는 나쁜 SQL
-- 비효율적인 페이징을 수행하는 나쁜 SQL
EXPLAIN
SELECT e.EMP_ID , e.FIRST_NAME , e.LAST_NAME , e.HIRE_DATE 
  FROM EMP e, -- 4만개
  	   SALARY s  -- 280만개
 WHERE e.EMP_ID = s.EMP_ID 
   AND e.EMP_ID BETWEEN 10001 AND 50000
 GROUP BY e.EMP_ID 
 ORDER BY SUM(s.ANNUAL_SALARY) DESC 
 LIMIT 150, 10; -- 394ms

EXPLAIN
SELECT e.EMP_ID , e.FIRST_NAME , e.LAST_NAME , e.HIRE_DATE 
  FROM EMP e,
  	   (SELECT EMP_ID AS EMP_ID
  	      FROM SALARY
  	     WHERE EMP_ID BETWEEN 10001 AND 50000
  	     GROUP BY EMP_ID
  	     ORDER BY SUM(ANNUAL_SALARY) DESC
  	     LIMIT 150, 10) s 
 WHERE e.EMP_ID = s.EMP_ID; -- 235ms
 
-- 개선해보자
EXPLAIN
SELECT e.EMP_ID , e.FIRST_NAME , e.LAST_NAME , e.HIRE_DATE 
  FROM (SELECT EMP_ID, FIRST_NAME, LAST_NAME, HIRE_DATE
          FROM EMP
         WHERE EMP_ID BETWEEN 10001 AND 50000) e,
  	   (SELECT EMP_ID AS EMP_ID
  	      FROM SALARY
  	     WHERE EMP_ID BETWEEN 10001 AND 50000
  	     GROUP BY EMP_ID
  	     ORDER BY SUM(ANNUAL_SALARY) DESC
  	     LIMIT 150, 10) s 
 WHERE e.EMP_ID = s.EMP_ID; -- 166ms

EXPLAIN
SELECT e.EMP_ID, e.FIRST_NAME, e.LAST_NAME, e.HIRE_DATE
FROM (SELECT EMP_ID
        FROM SALARY
       WHERE EMP_ID BETWEEN 10001 AND 50000
       GROUP BY EMP_ID
       ORDER BY SUM(ANNUAL_SALARY) DESC
       LIMIT 150, 10) s
STRAIGHT_JOIN (SELECT EMP_ID, FIRST_NAME, LAST_NAME, HIRE_DATE
          FROM EMP
         WHERE EMP_ID BETWEEN 10001 AND 50000) e
ON e.EMP_ID = s.EMP_ID; -- 160ms

-- OOOO 정보를 가져오는 나쁜 SQL
-- 불필요한 정보를 가져오는 나쁜 SQL
EXPLAIN
SELECT COUNT(emp_id) AS count
  FROM (SELECT e.emp_id, m.dept_id
          FROM (SELECT *
                  FROM EMP
                 WHERE gender = 'M'
                   AND emp_id > 300000
               ) e
          LEFT JOIN MANAGER m 
                 ON e.emp_id = m.emp_id
        ) sub; -- 65ms

-- 틀린 케이스
SELECT COUNT(*)
  FROM EMP e
 WHERE gender = 'M'
   AND emp_id > 300000
   AND EXISTS (
   		SELECT 1 
   		  FROM MANAGER m 
   		 WHERE e.EMP_ID = m.EMP_ID
   ); -- 3ms

-- 주의: emp_id 가 null이면 count 되지 않음
EXPLAIN
SELECT COUNT(emp_id) AS count
  FROM EMP
 WHERE gender = 'M'
   AND emp_id > 300000;
   

-- 비효율적인 OO을 수행하는 나쁜 SQL
-- 비효율적인 조인을 수행하는 나쁜 SQL
  
-- 임시 테이블에서 sort를 진행하는 나쁜 쿼리
EXPLAIN
SELECT DISTINCT de.dept_id
  FROM MANAGER m ,
  	   DEPT_EMP_MAPPING de
 WHERE m.DEPT_ID = de.DEPT_ID 
 ORDER BY de.DEPT_ID ; -- 556ms

-- 불필요한 중복제거?
-- 인덱스(이미 정렬됨) sort진행해도 좋음
SELECT de.dept_id
  FROM MANAGER m ,
  	   DEPT_EMP_MAPPING de
 WHERE m.DEPT_ID = de.DEPT_ID 
 GROUP BY m.DEPT_ID 
 ORDER BY de.DEPT_ID ; -- 146ms

SELECT de.dept_id
  FROM DEPT_EMP_MAPPING de,
  	   MANAGER m
 WHERE m.DEPT_ID = de.DEPT_ID 
 GROUP BY m.DEPT_ID 
 ORDER BY de.DEPT_ID ; -- 146ms

EXPLAIN
SELECT de.dept_id
  FROM (select distinct dept_id from DEPT_EMP_MAPPING) de
 WHERE EXISTS (SELECT 1 FROM MANAGER m WHERE m.dept_id = de.DEPT_ID); -- 3ms


 
-- 인덱스없이 데이터를 조회하는 나쁜 SQL
EXPLAIN
SELECT * 
FROM EMP 
IGNORE INDEX (`i_last_first_name`)
WHERE FIRST_NAME = 'Georgi'
and LAST_NAME = 'Wielonsky'; -- 73ms

 
SELECT *
FROM EMP
WHERE FIRST_NAME = 'Georgi'
and LAST_NAME = 'Wielonsky'; -- 2ms

-- first_name + last_name 인덱스 생성 -> 생성을 하는 것은 select 쿼리 속도 향상 but insert, update 등등 다른 성능이 안 좋아짐
-- 인덱스에도 값을 넣어줘야 하기 때문
-- gender + last_name 인덱스 변경 -- 조심스럽게 할 것(영향도 체크해야 함)

-- 1번 방식으로 진행해보자

-- mysql: ALTER TABLE EMP ADD i_last_first_name(last_name, first_name);
-- mariadb 방법
ALTER TABLE EMP ADD INDEX i_last_first_name (last_name, first_name);
 
-- OOO를 사용하지 않는 나쁜 쿼리
-- 인덱스를 사용하지 않는 나쁜 쿼리
EXPLAIN
SELECT *
FROM EMP IGNORE INDEX (`I_FIRST_NAME`)
WHERE FIRST_NAME = 'Matt'
OR HIRE_DATE = '1987-03-31'; -- 65ms

ALTER TABLE EMP ADD INDEX I_FIRST_NAME (first_name);

EXPLAIN
SELECT * 
FROM EMP
WHERE HIRE_DATE = '1987-03-31'
   OR FIRST_NAME = 'Matt'; -- 6ms
   
   
-- 인덱스에 나쁜 영향을 주는 OOO
-- 이력용 테이블에서는 이 작업이 오래 걸릴 수 있음
-- 인덱스에 나쁜 영향을 주는 나쁜 DML
-- 인덱스를 지우고, update 후 다시 인덱스 생성하는게 더 빠를 수 있음
-- 대량의 데이터(인덱스가 걸린) 컬럼을 UPDATE 할 경우 문제 발생
SELECT @@autocommit;

SET @@autocommit = 0;

UPDATE ENTRY_RECORD 
   SET GATE = 'X'
 WHERE GATE = 'B';

ROLLBACK;
 

--   PRIMARY KEY (`NO`,`EMP_ID`) USING BTREE,
--  KEY `I_REGION` (`REGION`),
--  KEY `I_ENTRY_TIME` (`ENTRY_TIME`),
--  KEY `I_GATE` (`GATE`)

ALTER TABLE ENTRY_RECORD ADD INDEX I_REGION(REGION);
ALTER TABLE ENTRY_RECORD ADD INDEX I_ENTRY_TIME(ENTRY_TIME);
ALTER TABLE ENTRY_RECORD ADD INDEX I_GATE(GATE);


SET @@autocommit = 1;

-- 비효율적인 OOO를 사용하는 나쁜 SQL
-- 비효율적인 인덱스를 사용하는 나쁜 SQL
SELECT EMP_ID, FIRST_NAME, LAST_NAME
FROM EMP
WHERE GENDER = 'M'
  AND LAST_NAME = 'Baba';
  
SELECT COUNT(*)
FROM EMP
WHERE GENDER = 'M'; -- 179973

SELECT COUNT(*)
FROM EMP
WHERE LAST_NAME = 'Baba'; -- 226

-- 인덱스 순서 아닐까?..
-- 카디널리티가 좋은 컬럼인 LAST_NAME을 선두컬럼으로 선정하는게 더 나음

ALTER TABLE EMP
DROP INDEX I_GENDER_LAST_NAME;

ALTER TABLE EMP
ADD INDEX I_LAST_NAME_GENDER(LAST_NAME, GENDER);

-- OOOO가 섞인 데이터와 비교하는 나쁜 SQL
-- 대소문자가 섞인 데이터와 비교하는 나쁜 SQL
EXPLAIN
SELECT FIRST_NAME, LAST_NAME, GENDER, BIRTH
FROM EMP
WHERE LOWER(FIRST_NAME) = LOWER('MARY')
AND HIRE_DATE >= STR_TO_DATE('1990-01-01', '%Y-%m-%d'); 

-- 인덱스를 조작했네..
EXPLAIN
SELECT FIRST_NAME, LAST_NAME, GENDER, BIRTH
FROM EMP
WHERE FIRST_NAME = 'Mary'
AND HIRE_DATE >= '1990-01-01';

ALTER TABLE EMP
ADD COLUMN lower_first_name varchar(14) not null after first_name; 
-- collate 'utf8mb3_general_ci'

UPDATE EMP
SET lower_first_name = LOWER(first_name);

SELECT *
FROM EMP LIMIT 5;

ALTER TABLE EMP ADD INDEX I_LOWER_FIRST_NAME(LOWER_FIRST_NAME);

SHOW INDEX FROM EMP;

EXPLAIN
SELECT FIRST_NAME, LAST_NAME, GENDER, BIRTH
FROM EMP
WHERE LOWER_FIRST_NAME = 'mary'
AND HIRE_DATE >= STR_TO_DATE('1990-01-01', '%Y-%m-%d');

ALTER TABLE EMP DROP COLUMN LOWER_FIRST_NAME;

-- OO 없이 대량 데이터를 사용하는 나쁜 SQL
-- 분산 없이 대량 데이터를 사용하는 나쁜 SQL
EXPLAIN
SELECT COUNT(1)
FROM SALARY -- 2844047
WHERE START_DATE BETWEEN STR_TO_DATE('2000-01-01', '%Y-%m-%d')
					 AND STR_TO_DATE('2000-12-31', '%Y-%m-%d'); -- 255785
					 
SELECT COUNT(*) FROM SALARY;

SHOW INDEX FROM SALARY;

SELECT * FROM SALARY LIMIT 10;

SELECT YEAR(START_DATE), COUNT(1)
FROM SALARY
GROUP BY YEAR(START_DATE);

ALTER TABLE SALARY
PARTITION BY RANGE COLUMNS (start_date)
(
	PARTITION part85 values less than ('1985-12-31'),
	PARTITION part86 values less than ('1986-12-31'),
	PARTITION part87 values less than ('1987-12-31'),
	PARTITION part88 values less than ('1988-12-31'),
	PARTITION part89 values less than ('1989-12-31'),
	PARTITION part90 values less than ('1990-12-31'),
	PARTITION part91 values less than ('1991-12-31'),
	PARTITION part92 values less than ('1992-12-31'),
	PARTITION part93 values less than ('1993-12-31'),
	PARTITION part94 values less than ('1994-12-31'),
	PARTITION part95 values less than ('1995-12-31'),
	PARTITION part96 values less than ('1996-12-31'),
	PARTITION part97 values less than ('1997-12-31'),
	PARTITION part98 values less than ('1998-12-31'),
	PARTITION part99 values less than ('1999-12-31'),
	PARTITION part00 values less than ('2000-12-31'),
	PARTITION part01 values less than ('2001-12-31'),
	PARTITION part02 values less than ('2002-12-31'),
	PARTITION pMax values less than (MAXVALUE)
);

EXPLAIN
SELECT COUNT(1)
FROM SALARY -- 2844047
WHERE START_DATE BETWEEN STR_TO_DATE('2000-01-01', '%Y-%m-%d')
					 AND STR_TO_DATE('2000-12-31', '%Y-%m-%d'); 
					
EXPLAIN PARTITIONS
SELECT COUNT(1)
FROM SALARY
WHERE START_DATE BETWEEN '2000-01-01' AND '2000-12-31';

ALTER TABLE SALARY 
REMOVE partitioning;


```
