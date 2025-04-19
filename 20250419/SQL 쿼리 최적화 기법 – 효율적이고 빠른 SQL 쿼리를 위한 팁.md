논문 링크: https://andrewrgoss.com/pdf/sql_query_optimization_techniques.pdf

SQL은 데이터베이스에서 데이터를 검색하는 데 사용될 수 있다. 데이터베이스에서 일정 시간 동안 데이터를 검색해 본 경험이 있다면, 느리게 실행되는 쿼리에 직면해 본 적이 있을 것이다. 느린 응답 속도의 원인은 시스템 부하일 수도 있지만, 더 일반적인 이유는 SQL이 가능한 한 효율적으로 작성되지 않았기 때문이다. 더 나은 성능을 위해서는 가장 효율적이고 빠른 쿼리를 작성하는 방법을 아는 것이 필요하다. 이 논문에서는 성능 향상을 위한 SQL 쿼리 최적화 방법을 다룬다. 본 논문에서는 데이터베이스 구조의 심층적인 분석보다는, 단순한 쿼리 튜닝 팁과 트릭을 통해 즉각적인 성능 향상을 얻는 데 중점을 둔다.


## 서론(Introduction)
쿼리 최적화는 SQL 개발자와 DBA(데이터베이스 관리자)에게 중요한 기술이다. SQL 쿼리의 성능을 향상시키기 위해, 개발자와 DBA는 쿼리 옵티마이저(query optimizer)와 액세스 경로(access path)를 선택하고 실행 계획을 준비하는 데 사용되는 기법을 이해해야 한다.

쿼리 튜닝은 비용 기반(cost-based)과 휴리스틱 기반(heuristic-based) 옵티마이저 같은 기술들에 대한 지식을 포함한다. SQL 플랫폼에서 제공되는 툴은 실행 계획을 설명하는 데 도움이 된다. 성능을 향상시키는 가장 좋은 방법은 쿼리를 다양한 방식으로 작성하고, 읽기 및 실행 계획을 비교하는 것이다.

이 논문에서는 데이터베이스 쿼리를 최적화하는 데 사용할 수 있는 다양한 기술들을 제안한다.

## 쿼리 최적화를 위한 일반적인 팁 (General Tips for Query Optimization)
각 팁은 Oracle 11g 샘플 데이터베이스(Sales 스키마)를 이용해 원본 쿼리와 개선된 쿼리를 실행하여 테스트했으며, 효율적인 쿼리를 사용했을 때의 속도 향상을 평균 실행 시간으로 측정하여 보여줍니다.

## Tip #1:SELECT 문에서 * 대신 컬럼명을 사용하라
테이블에서 일부 컬럼만 필요할 경우, `SELECT *`를 사용할 필요가 없다. `SELECT *`는 작성하기는 더 쉬우나, 데이터베이스가 전체 컬럼을 불러와야 하기 때문에 시간이 더 소요된다. 필요한 컬럼만 선택하면 결과 테이블의 크기를 줄일 수 있고, 네트워크 트래픽도 줄어들어 전반적인 성능이 향상된다.
```sql
[원본 쿼리]
SELECT * FROM SH.Sales;
```

```sql
[개선된 쿼리]
SELECT s.prod_id FROM SH.sales s;
```
![Pasted image 20250326100401](https://github.com/user-attachments/assets/c1b99f45-9c6e-4650-ab5c-b14bac575ec8)


SELECT * 쿼리는해당 페이지에 있는 모든 컬럼의 데이터를 전송해야한다. 반면 SELECT s.prod_id쿼리는 해당 페이지에서 prod_id컬럼의 데이터만 전송한다.

 이때 s.prod_id가 PK이라면 데이터 접근할 필요 없이 클러스터형 인덱스만 읽어서 바로 결과를 출력해 더 효율적이게 된다.(커버링 인덱스)

## Tip #2: SELECT 문에서 HAVING 절 사용을 피하라
`HAVING` 절은 모든 행을 선택한 후 필터링하는 방식으로 작동하기 때문에, 마치 필터처럼 작동한다. 이 절은 `SELECT` 문에서 거의 쓸모없다. `HAVING`은 쿼리 결과 테이블을 만든 후 조건에 맞지 않는 행을 제거하므로, 효율적이지 않다.
반면, 동일한 조건을 `WHERE` 절에 포함시키면 쿼리 성능이 훨씬 향상된다.

```sql
[원본 쿼리]
SELECT s.cust_id, count(s.cust_id)
FROM SH.sales s
GROUP BY s.cust_id
HAVING s.cust_id != '1660' AND s.cust_id != '2';
```

```sql
[개선된 쿼리]
SELECT s.cust_id, count(cust_id)
FROM SH.sales s
WHERE s.cust_id != '1660' AND s.cust_id != '2'
GROUP BY s.cust_id;
```
![Pasted image 20250326104335](https://github.com/user-attachments/assets/e9ed2f74-67a4-401c-b2fb-07b5a488cfe4)


원본 쿼리는 쿼리 수행 순서상 `GROUP BY s.cust_id`가 먼저 실행되어 **전체 테이블**의 데이터를 `cust_id`로 그룹화한다.
그 후 HAVING 조건에 따라 필터링한다.

개선된 쿼리는 WHERE 조건에서 먼저 필터링 된 후 만족한 레코드에 대해서만 그룹화를 수행한다. WHERE 조건절이기 때문에 인덱스가 존재할 경우 인덱스를 통해 빠르게 필터링할 수 있다. 또한 조건에 만족하는 데이터만 그룹화 되기 때문에 연산 데이터 양이 줄어 성능향상된다.

## Tip #3: 불필요한 `DISTINCT` 조건을 제거하라
다음 예제처럼, `DISTINCT` 키워드는 **불필요하게 사용되는 경우**가 많다.
예시의 경우, 결과 집합에 포함된 테이블의 `p.ID`가 기본 키이기 때문에, 중복이 발생하지 않으므로 `DISTINCT`가 필요 없다.
```sql
[원본 쿼리]
SELECT DISTINCT * 
FROM SH.sales s  
JOIN SH.customers c  
ON s.cust_id = c.cust_id  
WHERE c.cust_marital_status = 'single';
```

```sql
[개선된 쿼리]
SELECT *  
FROM SH.sales s  
JOIN SH.customers c  
ON s.cust_id = c.cust_id  
WHERE c.cust_marital_status = 'single';
```
![Pasted image 20250326105223](https://github.com/user-attachments/assets/8bfb38dc-66b1-4248-8c68-909145bcc653)



## Tip #4: 서브쿼리 풀어내기 (Un-nest sub queries)
중첩된 서브쿼리를 조인으로 다시 작성하면 **더 효율적인 실행과 최적화**가 가능해진다.  
일반적으로 상관 서브쿼리는 `FROM`절에서 최대 한 번만 사용되며, `ANY`, `ALL`, `EXISTS` 같은 술어에 사용된다. 
서브쿼리를 조인 형태로 바꾸는 것만으로도 실행 속도가 빨라진다.
```sql
[원본 쿼리]
SELECT *  
FROM SH.products p  
WHERE p.prod_id = (  
  SELECT s.prod_id  
  FROM SH.sales s  
  WHERE s.cust_id = 100996  
  AND s.quantity_sold = 1  
);
```

```sql
[개선된 쿼리]
SELECT p.*  
FROM SH.products p, sales s
WHERE p.prod_id = s.prod_id
AND s.cust_id = 100996  
AND s.quantity_sold = 1;
```
![Pasted image 20250326105758](https://github.com/user-attachments/assets/b7d85d48-ec61-4961-a4f6-13ab552c60d0)


JOIN 구문을 명시적으로 사용하지 않으면 카테시안 곱(Cross Join)이 발생하지만, Where절에 p.prod_id = s.prod_id라는 조인 조건이 있기 때문에 INNER JOIN으로 동작한다. 더욱 가독성 있게 작성하면 다음과 같다.
```sql
SELECT p.* 
FROM SH.products p 
JOIN SH.sales s ON p.prod_id = s.prod_id 
WHERE s.cust_id = 100996 
  AND s.quantity_sold = 1;
```


## Tip #5: 인덱스 컬럼을 대상으로 `IN` 조건 사용 고려하기
`IN` 조건은 **인덱스 검색을 이용하여 성능을 향상**시킬 수 있다.  
또한 옵티마이저는 `IN` 목록을 정렬하여 인덱스 정렬 순서와 맞추어 더 효율적인 검색을 유도할 수 있다.
단, `IN` 목록에는 **상수** 또는 **한 쿼리 내에서 고정된 값**만 포함되어야 한다.
```sql
SELECT s.*  
FROM SH.sales s  
WHERE s.prod_id = 14  
  OR s.prod_id = 17;
```

```sql
SELECT s.*  
FROM SH.sales s  
WHERE s.prod_id IN (14, 17);
```

![Pasted image 20250326112300](https://github.com/user-attachments/assets/03ee15b9-70a7-4666-93d3-1921eea2e2b4)


IN과 OR의 인덱스 스캔 차이점
IN은 옵티마이저 관점에서 prod_id를 한 번만 타고 14, 16을 찾는 방식이다. 즉, 하나의 인덱스 스캔으로 처리가능하다.
OR은 옵티마이저에서 여러 개의 조건 블록으로 보고, 각 조건마다 따로 인덱스 스캔을 수행하거나 조건이 복잡하면 풀 테이블 스캔을 선택할 수 있다.


## Tip #6: 일대다(1:N) 관계 조인 시 `DISTINCT` 대신 `EXISTS` 사용하라
`DISTINCT`는 테이블의 모든 열을 선택한 뒤 중복을 제거하는 방식이다. 하지만 `EXISTS`를 사용하면 전체 테이블 조회를 피하면서 조건에 맞는 행이 존재하는지만 확인할 수 있다.
```sql
[원본 쿼리 - 해당 국가의 고객이 있는지 확인]
SELECT DISTINCT c.country_id, c.country_name  
FROM SH.countries c, SH.customers e  
WHERE e.country_id = c.country_id;
```

```sql
[개선된 쿼리]
SELECT c.country_id, c.country_name  
FROM SH.countries c  
WHERE EXISTS (  
  SELECT 'X' FROM SH.customers e  
  WHERE e.country_id = c.country_id  
);
```
![Pasted image 20250326121620](https://github.com/user-attachments/assets/9d5d4067-9bf5-4fdc-bad8-7c3f3864e2ab)


**EXISTS를 사용하면 최적화되는 이유**는 DISTINCT가 불필요한 중복 제거 작업을 수행하는 반면, EXISTS는 조기 종료(Early Termination)로 불필요한 연산을 줄이기 때문이다.

기존 방식은 INNER JOIN을 수행하여 `countries`와 `customers`를 매칭하고, 조인된 결과에서 DISTINCT를 사용하여 중복을 제거한다.

`customers`테이블이 일대다 관계(한 국가에 여러고객 존재)이므로, 같은 country_id가 여러 번 포함될 가능성이 높다.
`DISTINCT`가 모든 행을 처리한 후 중복을 제거하므로, 불필요한 데이터 스캔이 발생한다.

`EXISTS`는 특정 country_id가 customers 테이블에 존재하는지 여부만 검사한다. 매칭되는 첫 번째 행을 찾으면 즉시 종료(Early Termination)하게 된다.

`SELECT 'X'`는 **불필요한 데이터를 가져오지 않도록 최적화**(일반적으로 성능에 영향을 주지 않음)

## Tip #8: 조인 조건에 `OR` 사용을 피하라
조인 조건에 `OR`을 사용하면 쿼리 성능이 **두 배 이상 느려질 수 있습니다**.  대신 `UNION ALL`을 사용해 각각의 조건을 분리된 쿼리로 작성하면 성능이 향상됩니다.
```sql
[원본 쿼리]
SELECT *
FROM SH.costs c
INNER JOIN SH.products p
  ON c.unit_price = p.prod_min_price  
  OR c.unit_price = p.prod_list_price;
```

```sql
[개선된 쿼리]
SELECT *  
FROM SH.costs c  
INNER JOIN SH.products p  
  ON c.unit_price = p.prod_min_price  
UNION ALL  
SELECT *  
FROM SH.costs c  
INNER JOIN SH.products p  
  ON c.unit_price = p.prod_list_price;
```
![Pasted image 20250326122505](https://github.com/user-attachments/assets/81ad7213-c505-42f0-8f1a-2e3a393618fb)


해당 예시를 보면, products테이블과 costs테이블이 나눠져있어 이를 통합해야하는 경우이다. 엔티티(테이블)모델링 관점에서 적절하지 않아 보이나, 모종의 이유에서 분리했다고 하자.

원본 쿼리는 INNER JOIN으로 OR을 포함하는 쿼리로 작성되어있다. OR을 사용하는 쿼리는 옵티마이저가 효율적인 실행 계획을 세우기 어렵다. 어떤 인덱스를 사용해야할지 모호하기 때문이다. 때문에 옵티마이저가 하나의 인덱스만 사용할 수도, 인덱스를 아예 사용하지 않고 풀 스캔을 수행할 가능성이 있다.

개선된 쿼리에는 UNION ALL을 사용하여 개선한다. 이 경우 각 쿼리에 대해 하나의 조건(개별적인 조건)을 가지고 있어 옵티마이저가 각 쿼리에 대해 별도 실행 계획을 세울 수 있다.
그리고 `prod_min_price`, `prod_list_price` 각각에 인덱스가 있다면, 각 쿼리는 인덱스를 사용할 가능성이 높다.

## Tip #9: 연산자의 오른쪽에서 함수 사용을 피하라
SQL 쿼리에서 **함수 또는 메서드를 연산자의 오른쪽에 사용하는 경우**가 매우 흔하지만, 이 방식은 **인덱스 최적화를 방해하며 성능 저하**를 유발합니다.
```sql
[원본 쿼리]
SELECT *
FROM SH.sales
WHERE EXTRACT(YEAR FROM TO_DATE(time_id, 'DD-MON-RR')) = 2001
  AND EXTRACT(MONTH FROM TO_DATE(time_id, 'DD-MON-RR')) = 12;
```

```sql
[논문속 - 개선된 쿼리]
SELECT * FROM SH.sales
WHERE TRUNC(time_id) BETWEEN
      TRUNC(TO_DATE('12/01/2001', 'mm/dd/yyyy'))
  AND TRUNC(TO_DATE('12/30/2001', 'mm/dd/yyyy'));

[좋은 쿼리]
-- time_id가 varchar가 아닌 DATE일 경우
WHERE time_id BETWEEN TO_DATE('2001-12-01', 'YYYY-MM-DD')
 AND TO_DATE('2001-12-31 23:59:59', 'YYYY-MM-DD HH24:MI:SS')
```
![Pasted image 20250326152502](https://github.com/user-attachments/assets/c2572237-04c8-4dd4-b24e-7b9df5aaddaf)


원본 쿼리는 time_id를 TO_DATE()로 날짜로 변경 후 EXTRACT(YEAR), EXTRACT(MONTH)로 특정 년,월로 필터링한다.

옵티마이저 관점에서 동작을 확인하면, `time_id` 전 행에 대해 TO_DATE()를 수행하고 각각 EXTRACT(YEAR), EXTRACT(MONTH) 연산 수행할 것이다.
그리고 나서 위 연산 결과를 메모리에 올려 =2001, =12조건을 비교할 것이다.
이 과정은 인덱스를 완전히 무시하고 테이블 전체에 대해 수행된다.

논문에 개선된 쿼리 또한 TRUNC도 결국time_id에 변형을 가하는 것이기 때문에 인덱스를 못탈 가능성이 있다. 내생각에는 논문 설명을 참고한 의도는 아래의 좋은 쿼리일 것이다. 그러나 성능 비교란 일관된 환경에서 비교를 진행해야하기 때문에 다음 같은 예제로 첨부된 것 같다.

아래의 좋은 쿼리 또한 time_id가 VARCHAR타입이면 Index Range Scan이 불가능하다. TO_DATE()와 비교하려면 암묵적인 형변환이 발생하기 때문이다. 따라서 time_id가 DATE타입이고, time_id에 대해 인덱스가 있다면 Index Range Scan이 가능할 것이다.

## Tip #10: 불필요한 수학 연산을 제거하라
SQL 쿼리에서 수학적 연산을 사용하는 경우, 해당 연산이 매번 실행될 때마다 계산 비용이 발생한다. 특히, 불필요한 계산은 성능에 큰 영향을 줄 수 있으므로, 간단한 수학도 제거하거나 미리 계산해서 조건을 단순화하는 것이 좋다.
```sql
[원본 쿼리]
SELECT *
FROM SH.sales s  
WHERE s.cust_id + 10000 < 35000;
```

```sql
[개선된 쿼리]
SELECT *
FROM SH.sales s
WHERE s.cust_id < 25000;
```
![Pasted image 20250326160740](https://github.com/user-attachments/assets/38846922-ad30-46d0-b68a-2be8c7902625)


원본 쿼리에서는 s.cust_id + 10000로 변형된 값으로 비교를 한다. 때문에 옵티마이저는 `cust_id`에 인덱스가 있더라도, 해당 연산은 컬럼 변형이므로 인덱스를 사용할 수 없다. 옵티마이저는 Full Table Scan 선택하여 모든 레코드에 대해 +10000연산을 수행하고 조건 비교를 할 가능성이 높다.

개선된 쿼리에서는 컬럼에 직접적인 변형이 없으므로 인덱스가 존재한다면 Index Range Scan이 가능하여 성능상 이점이 있다. 인덱스가 없어 Full Table Scan 수행하더라도 덧셈 연산이 없고 비교 연산만 수행하면 되기 때문에 성능상 이점이 있다.
