---
layout: post
title: pymongo 와 mongoengine 으로 MongoDB 접속해서 query 하기
subtitle: pymongo, mongoengine 연결 및 query실행
categories: Tech
tags: [python,mongodb,nosql]
---

## pymongo와 mongoengine

pymongo는 Python을 위한 MongoDB의 공식적인 네이티브 드라이버이다. MongoDB Query API를 통해 MongoDB 데이터베이스에 접속하여 query할 수 있다. 

MongoEngine의 현재 버전은 PyMongo을 기반으로 만들어졌다.

간단 비교글 참고 : https://www.mongodb.com/compatibility/mongoengine-pymongo

db에 connect 하는 방법은 조금 다른데 연결 후 query 날리는 건 MongoDB Query API(https://www.mongodb.com/docs/v7.0/crud/#std-label-crud) 에 나오는 방법 그대로 쓰는게 똑같다.


## pymongo 로 mongodb 연결하기

```python
from pymongo import MongoClient

# 'my_db' database로 연결
client = MongoClient("mongodb://my_user:my_password@127.0.0.1:27017/my_db")

# DB name 출력
db = client.get_database()
print(db.name)

# collection list 출력
collection_list = db.list_collection_names()
print(collection_list)

# 'restaurant' collection의 document 하나만 출력
restaurant=db.restaurant
print(restaurant.find_one())
```


## mongoengine 로 mongodb 연결하기

```python
from mongoengine import connect, get_db

# 'my_db' database로 연결
connect(host="mongodb://my_user:my_password@127.0.0.1:27017/my_db")

# DB name 출력
db = get_db()
print(db.name)

# collection list 출력
collection_list = db.list_collection_names()
print(collection_list)

# 'restaurant' collection의 document 하나만 출력
restaurant=db.restaurant
print(restaurant.find_one())
```


