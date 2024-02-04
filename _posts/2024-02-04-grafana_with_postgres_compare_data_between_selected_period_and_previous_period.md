---
layout: post
title: Grafana - (/w PostgresDB) Timepicker 선택한 기간과 그 이전 기간 데이터 비교 계산하기
subtitle: 실제 query 방법
categories: CodeExample
tags: [grafana,postgresql]
---

그라파나에서 timepicker 에서 특정 기간을 선택하고 WHERE 구문에 $__timeFilter(열이름) 을 입력하면 Time 값이 포함된 열을 기준으로 filter가 적용된다. 다음 query 예시는 timepicker를 선택한 기간에 대해 그 이전 기간의 데이터 값과 비교하기 위함이다.

예를들어 timepicker에서는 2021-01-02 00시 ~ 2021-01-02 23:59:59시 를 선택했는데 1월 1일 하루동안의 데이터와 비교하고 싶은것.

### 활용한 기능

$__timeFilter : timepicker에서 선택한 기간 적용
$__timeTo() : timepicker에서 시작 시점
$__timeFROM() : timepicker에서 마지막 시점
::TIMESTAMP : datatype을 timestamp로 변환. 변환 안하니까 기간계산이 안되서 넣었다.

### 1월 2일 24시간을 선택했지만 1월1일 데이터를 select하는 방법

$__timeTo(), $__timeFROM()를 WHERE 구문에서 활용하여 기간을 계산하면 된다.

1. timepicker에서 시작 시점 - 마지막 시점 = 기간이 나온다
2. timepicker 시작 시점 - 1번에서 계산한 기간 = timepicker선택한 기간의 그 이전 기간의 시작점이 된다
3. timepicker 종료 시점 - 1번에서 계산한 기간 = timepicker선택한 기간의 그 이전 기간의 마지막 시점이 된다.
4. time값 ≤ 2번 : 이전 기간의 시작점 이후 데이터
5. time값 ≥ 3번 : 이전 기간의 시작점 이전 데이터
6. 4번과 5번 조건을 적용하면 timepicker 선택한 기간 보다 이전기간이 select 된다.

### 코드로 다시 설명한다

db에서 time값이 있는 열 이름이 created_at이라 하자. TIMESTAMP 변환안했더니 계산이 안되서 각 값을 변환했다(::TIMESTAMP)

```
-- 1. 기간
$__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP

-- 2. 1월 1일 00시
$__timeTo()::TIMESTAMP - ($__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP)

-- 3. 1월 1일 23:59:59
$__timeFROM()::TIMESTAMP - ($__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP)

-- 4. 1월 1일 00시 이후 
created_at <= ($__timeTo()::TIMESTAMP - ($__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP))

-- 5. 1월 1일 23:59:59 이전
created_at >= ($__timeFROM()::TIMESTAMP - ($__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP))

-- 6. query에 넣어야할 WHERE
WHERE created_at <= ($__timeTo()::TIMESTAMP - ($__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP)) 
  AND created_at >= ($__timeFROM()::TIMESTAMP - ($__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP)) 
```

### 실제 계산 사례

아래 표가 있다. grafana time picker 에서 1월 2일 24시간을 선택한 상황.

|id|created_at|score|threshold|score|
|---|---|---|---|---|
|1|2021-01-01 1:00:00|10|55|SUCCESS|
|2|2021-01-01 6:00:00|80|55|FAIL|
|3|2021-01-01 12:00:00|30|55|SUCCESS|
|4|2021-01-01 18:00:00|90|55|SUCCESS|
|5|2021-01-02 1:00:00|70|55|SUCCESS|
|6|2021-01-02 6:00:00|30|55|SUCCESS|
|7|2021-01-02 12:00:00|90|55|FAIL|
|8|2021-01-02 18:00:00|60|55|SUCCESS|

### 계산해야 하는 값은 다음과 같다

1. status가 SUCCESS 인 것들 중
2. score가 threshold를 넘는 id 수 / 특정 기간에 대해 id 수 → ratio
3. 1월 2일의 ratio 와 1월 1일의 ratio 차이

### 추가 활용 기능

WITH : view와 같은 기능

WINDOW FUNTION : COUNT() FILTER (WHERE ~) : aggregate 를 서로 다른 조건으로 계산 가능

::NUMERIC : data type를 numeric 으로 변환. 사칙연산할때 유효숫자 문제 때문에 numeric으로 변환해서 사용

### 계산 과정

1. new 테이블을 만들고 timepicker에서 선택한 기간(1월 2일 00시부터 23:59:59 까지)에 대해 ratio를 계산
    → count를 두번 하게된다. 한번은 그냥 다 세는거고 한번은 threshold 넘는것만. window function 활용했다
    → status 조건 where로 걸고
    → timepicker 선택한 기간은 $__timeFilter를 where로 조건 걸었다.
2. old 테이블을 만들고 이전 기간(1월 1일 하루)에 대해 ratio를 계산
    → 위에서 구했던 이전 기간 where 조건을 넣었다.
3. new.ratio 빼기 old.ratio
    → new와 old를 조인하기 위해서 new, old 각각에 1을 select하고 joinid라 정의한 다음. 이걸 조건으로 join 했다
    

### 결과 query

```
WITH new AS(
SELECT 
1 AS joinid
,(COUNT(id) FILTER (WHERE score >= threshold)::NUMERIC) / (COUNT(id)::NUMERIC) * 100 AS ratio
FROM list_mmg
WHERE $__timeFilter(created_at) AND status='SUCCESS' 
)
, old AS(
SELECT 
1 AS joinid
,(count(id) FILTER (WHERE score >= threshold)::NUMERIC) / (COUNT(id)::NUMERIC) * 100 AS ratio
FROM list_mmg
WHERE created_at <= ($__timeTo()::TIMESTAMP - ($__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP)) 
  AND created_at >= ($__timeFROM()::TIMESTAMP - ($__timeTo()::TIMESTAMP - $__timeFROM()::TIMESTAMP)) 
  AND status='SUCCESS'
)
SELECT
 (new.ratio::NUMERIC - old.ratio::NUMERIC) AS growth_rate
FROM new
LEFT JOIN old on new.joinid=old.joinid
```

grafana변수
[https://grafana.com/docs/grafana/latest/variables/variable-types/global-variables/](https://grafana.com/docs/grafana/latest/variables/variable-types/global-variables/)

WITH 설명
[https://yahwang.github.io/posts/49](https://yahwang.github.io/posts/49)

Window function 설명
[https://www.postgresqltutorial.com/postgresql-window-function/](https://www.postgresqltutorial.com/postgresql-window-function/)
