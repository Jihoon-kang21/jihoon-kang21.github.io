---
layout: post
title: python script로 postgresql db에 query 날리기
subtitle: psycopg2를 이용해 table을 만들고 insert 문을 실행해본다
categories: Code Example
tags: [python,postgresql]
---

## postgresql 설치

```bash
sudo apt-get install postgresql postgresql-contrib 

# psql 실행
sudo -u postgres psql

# 초기 비밀번호
postgre=# ALTER USER postgres PASSWORD '비밀번호'

# DB 생성
postgre=# create database 'DB이름'
```

## python에서 postgres에 query를 날리기 위한 라이브러리를 설치
```
sudo apt-get install python-psycopg2
sudo apt-get install libpq-dev

pip install psycopg2
# 또는
python3 -m pip install
```

## 예시 스크립트
아래는 psql 에 접속하여 다양한 query를 날려보는 예시 스크립트 이다.
```python
import psycopg2
import base64  # binary를 decode할 용도


# postgresql db에 연결할 로그인 정보
conn = psycopg2.connect(
    database="your_database_name",
    user="your_database_user",
    password="your_database_password",
    host="your_database_host",
    port="your_database_port"
)
cursor = conn.cursor()

# 'your_table_name'이라는 테이블이 있는지 확인하는 쿼리 실행
cursor.execute("SELECT EXISTS (SELECT 1 FROM information_schema.tables WHERE table_name = 'your_table_name')")
table_exists = cursor.fetchone()[0]

if table_exists:
    # 테이블이 있는 경우 테이블의 모든 row 삭제
    cursor.execute("TRUNCATE TABLE your_table_name")
else:
    # 테이블이 존재하지 않는 경우 새로 만든다.
    cursor.execute('''CREATE TABLE your_table_name
                      (id SERIAL PRIMARY KEY,
                       key_name TEXT,
                       data_type TEXT,
                       data_count INTEGER)''')

for i in keys:
    key_name = i.decode('utf-8')
	data_type = 'set'
	data_count = master.scard(i)

    # Insert 문 실행
    cursor.execute("INSERT INTO redis_data (key_name, data_type, data_count) VALUES (%s, %s, %s)", (key_name, data_type, data_count))

# 변경내용을 저장하고 connection을 종료한다.
conn.commit()
conn.close()
```

