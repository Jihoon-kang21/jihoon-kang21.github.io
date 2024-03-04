---
layout: post
title: SQL - ID별 주요 파라미터 값 추출 및 비교
subtitle: max, group by, having 활용
categories: CodeExample
tags: [sql]
---

## 문제 테이블

하나의 id가 para 종류 별로 여러개의 값을 가지고 있다.
어떤 id는 A,B 두개 para를 가진다.
어떤 id는 A,B 둘 중 하나의 para를 가지고 있거나 C 또는 D 만 가지고 있다.

category가 Electronics 이면서 A, B 두개 para 모두 가지고 있는 id들을 뽑아서,
A, B 의 value 값을 비교하기 위해 A열에는 A의 value, B 열에는 B의 value를 출력하고 싶다.

| id | para | value | category    | date       |
|----|------|-------|-------------|------------|
| 1  | A    | 0.12  | Electronics | 2023-01-10 |
| 1  | B    | 0.13  | Electronics | 2023-01-10 |
| 2  | A    | 0.14  | Electronics | 2023-02-15 |
| 3  | B    | 0.15  | Home Goods  | 2023-03-20 |
| 4  | A    | 0.16  | Electronics | 2023-04-25 |
| 4  | B    | 0.17  | Electronics | 2023-04-25 |
| 5  | A    | 0.18  | Home Goods  | 2023-05-30 |
| 1  | C    | 0.19  | Electronics | 2023-01-10 |
| 4  | D    | 0.20  | Electronics | 2023-04-25 |

## Query 

아래 사항들을 적용했다.

- WHERE 절에서 category가 Electronics 인 경우만 출력한다
- WHERE 절에서 para가 A 또는 B 인 경우만 출력한다.
- CASE를 이용해서 A, B의 value를 column으로 만든다.
- 하나의 para에 여러 값이 있는 경우 MAX를 이용해서 최대값 하나만 추출한다.
- HAVING으로 A, B 둘다 있는 경우만 출력하도록 한다.

```sql
SELECT
  id,
  MAX(CASE WHEN para = 'A' THEN value END) AS "A",
  MAX(CASE WHEN para = 'B' THEN value END) AS "B"
FROM
  parameters
WHERE
  (para = 'A' OR para = 'B')
  AND category = 'Electronics'
GROUP BY
  id
HAVING
  COUNT(DISTINCT para) = 2;
```
## 결과

| id | A    | B    |
|----|------|------|
| 1  | 0.12 | 0.13 |
| 4  | 0.16 | 0.17 |

MAX가 굳이 필요없다면 self join으로도 될 것 같다.

```sql
SELECT
  a.id,
  a.value AS "A",
  b.value AS "B"
FROM
  parameters a
JOIN
  parameters b ON a.id = b.id AND a.para = 'A' AND b.para = 'B'
WHERE
  a.category = 'Electronics' AND b.category = 'Electronics';
```
