---
layout: post
title: python-mongoengine 사용해서 MongoDB query하기
subtitle: MongoDB에 접속해서 다양한 query 실행하는 방법
categories: CodeExample
tags: [python,mongodb]
---

### 1: 필요한 라이브러리 가져오기
```python
from bson import ObjectId
from time import time
from datetime import datetime, timedelta, date, time as dt_time
from mongoengine import connect, disconnect, get_db
import json
from pprint import pprint
```

### 2: 현재 연결된 MongoDB가 있다면 연결 해제
```python
disconnect()
```

### 3: MongoDB 서버에 인증 및 레플리카 설정 사용하여 연결
- 각 부분에 실제 데이터베이스 연결 세부 정보로 대체한다.
- authSource나 replicaSet은 해당되는 경우에만 입력한다.
```python
connect(
    host="mongodb://jihoon:PASSWORD@10.120.120.1:12000/database_name?authSource=mongodbxxx&replicaSet=mdb-repl10"
)
```

### 4: 어떤 collection 들이 있는지 리스트 확인
- 연결된 데이터베이스의 모든 collection 이름 출력한다.
```python
pprint(get_db().list_collection_names())
```

### 5: $in 연산자 사용하여 여러 값을 필터링하여 collection 내 document 가져오고 출력
- <collection_name>을 실제 컬렉션 이름으로 대체하고 $in 연산자를 사용하여 여러 값으로 필터링한다.
- find()를 쓰는 경우 cursor object 을 반환하므로 print하기 위해서는 for 문 사용이 필요하다.
- 출력될 데이터가 많은 경우 limit()로 갯수를 제한 할 수 있다.
```python
values = ["value1", "value2", "value3"]  # 사용자 지정 값 목록
cursor = get_db().collection_name.find({"field_name": {"$in": values}}).limit(10)
for document in cursor:
    pprint(document)
```

### 6: $gte 연산자 사용하여 날짜 필터링하여 collection 내 document 가져오고 출력
- <collection_name>을 실제 collection 이름으로 대체한다.
- $gte 연산자를 사용하여 특정 날짜 이후로 필터링 할 수 있다.
```python
start_date = datetime.fromisoformat("2023-01-01")  # 사용자 지정 시작 날짜
cursor = get_db().collection_name.find({"date_field_name": {"$gte": start_date}}).limit(10)
for document in cursor:
    pprint(document)
```

### 7: ObjectId를 사용하여 collection 내 document 가져오고 출력
- 실제 collection 이름으로 <collection_name> 대체한다.
- 출력할 필드 이름으로 <field_name> 대체한다.
- sort("날짜필드", -1) 로 최신 데이터를 가장 상위에서 확인할 수 있다. -1 대신 1을 쓰면 반대로 정렬한다.
```python
cursor = get_db().<collection_name>.find({"<field_name>": ObjectId('6515c6c1fb')}).sort("created_at", -1).limit(2)
for document in cursor:
    pprint(document)
```

### 8: collection에서 하나의 document를 가져와 출력
- 실제 collection 이름으로 <collection_name> 대체
- 특정 collection에 어떤 필드들이 존재하는지 확인이 필요한 경우 활용할 수 있다.
```python
document = get_db().<collection_name>.find_one()
if document:
    pprint(document)
```

### 9: 특정 기준에 부합하는 collection 내 document 수 세기
- 여러가지 필드를 중첩으로 필터 한다.
- count_documents()로 document의 개수를 확인할 수 있다.
```python
count = get_db().<collection_name>.count_documents({
    "<field1>": "value1",
    "<field2>": {"$gte": datetime.fromisoformat("2023-01-01T00:00:00")},
    "<field3>": "value3",
    "<field4>": {"$in": ["value4a", "value4b", "value4c"]}
})
print("Count of documents:", count)
```
