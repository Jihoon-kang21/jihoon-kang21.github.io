---
layout: post
title: SQL - 중복데이터가 존재하는 데이터 찾기
subtitle: group by, having 활용하기
categories: CodeExample
tags: [sql]
---

### 문제 

DB에서 중복값이 존재하는 데이터가 어떤 것들이 있는지 찾아야할 때가 있다.
아래는 예시이다.
서점에 책 목록에 대한 DB가 있다. 그런데 여러 데이터 소스에서 데이터를 모으다 보니, DB에 중복으로 등록된 책들이 있다.

### Example Table: `books`

아래 예시 테이블이 이름을 `books` 라고 하자.

| book_id | title                 | author              | publication_year |
| ------- | --------------------- | ------------------- | ---------------- |
| 1       | The Great Gatsby      | F. Scott Fitzgerald | 1925             |
| 2       | To Kill a Mockingbird | Harper Lee          | 1960             |
| 3       | The Great Gatsby      | F. Scott Fitzgerald | 1925             |
| 4       | 1984                  | George Orwell       | 1949             |
| 5       | The Great Gatsby      | F. Scott Fitzgerald | 1925             |

여기서  "The Great Gatsby" 가 세번 중복되어있다.

### The SQL Query for Finding Duplicates

아래 쿼리를 통해 중복된 책이 어떤 것들이 있는지 확인할 수 있다.

```sql
SELECT title, author, COUNT(*)
FROM books
GROUP BY title, author
HAVING COUNT(*) > 1;
```

#### How It Works

1. `SELECT title, author, COUNT(*)`: 쿼리 결과 확인하고 싶은 데이터를 선택한다. 여깃는 title, anthor 그리고 중복 횟수를 나타내줄 `COUNT(*)` 
   
2. `FROM books`: 쿼리할 table 이름 `books`.

3. `GROUP BY title, author`: 모든 책들을 책 제목과 저자이름으로 그룹짓는다. 책 제목과 저자이름의 조합이 하나의 그룹이 된다.

4. `HAVING COUNT(*) > 1`: 데이터가 group by로 그룹지어진 상태 기준으로 각 그룹(책 제목과 저자이름의 조합)의 개수가 `COUNT(*)`의 값이되고, 그룹의 개수가 1보다 큰 경우만 결과로 보여준다.

### Query Output

쿼리결과는 다음과 같다.

| title            | author              | count |
| ---------------- | ------------------- | ----- |
| The Great Gatsby | F. Scott Fitzgerald | 3     |
중복된 값이 존재하는 "The Great Gatsby" "F. Scott Fitzgerald" 만 결과로 나타나고 count 열에는 중복 데이터 수를 보여준다.
예를 들어 "1984" "George Orwell" 은 전체 책 목록에서 하나만 존재하므로 결과에 나타나지 않는다.


