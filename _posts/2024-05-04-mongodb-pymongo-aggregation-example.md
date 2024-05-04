---
layout: post
title: MongoDB - Pymongo - Aggregation 기본 사용예시
subtitle: How to use match, lookup, unwind, project
categories: Tech
tags: [python,mongodb,pymongo]
---

## Aggregation의 장점

- 여러 collection을 중첩 쿼리하는 경우 시간도 많이 걸리고 DB 서버에 부하를 많이 줄 수 있다.
- Aggregation을 이용하면 한번에 쿼리하게 되므로 시간도 줄고 부하도 줄일 수 있다.
- 중첩 쿼리를 Aggregation 으로 바꾸면 쿼리도 훨씬 간단해져 이해하기도 쉬워진다.


## Example collections & documents

예시 collection 과 document들은 아래와 같다.

```python
from pymongo import MongoClient

# Connect to MongoDB
client = MongoClient("mongodb://localhost:27017/")
db = client['example_db']

# Insert sample data into the models collection
db.models.insert_many([
    {"_id": ObjectId("61398998ddhojkl1"), "created_at": "2024-04-25", "status": "error", "progress": 0.2, "dataset_id": ObjectId("5ldf23208098d001")},
    {"_id": ObjectId("61398998ddhojkl2"), "created_at": "2024-05-01", "status": "success", "progress": 0.9, "dataset_id": ObjectId("5ldf23208098d002")},
    {"_id": ObjectId("61398998ddhojkl3"), "created_at": "2024-04-20", "status": "error", "progress": 0.5, "dataset_id": ObjectId("5ldf23208098d003")}
])

# Insert sample data into the datasets collection
db.datasets.insert_many([
    {"_id": ObjectId("5ldf23208098d001"), "name": "base_123"},
    {"_id": ObjectId("5ldf23208098d002"), "name": "base_124"},
    {"_id": ObjectId("5ldf23208098d003"), "name": "base_125"}
])
```

## Problem to solve
status가 error 인 모델들을 나열하면서, 동시에 각 모델에 연결된 dataset의 이름까지 확인하고 싶다.

## Expected result

```python

## Result

아래와 같은 리스트를 만들고 싶다.
_id는 model의 id이고, dataset의 이름을 같이 확인할 수 있게 된다.

[
    {
        "_id": ObjectId("61398998ddhojkl1"),
        "created_at": "2024-04-25",
        "status": "error",
        "progress": 0.2,
        "name": "base_123"
    },
    {
        "_id": ObjectId("61398998ddhojkl3"),
        "created_at": "2024-04-20",
        "status": "error",
        "progress": 0.5,
        "name": "base_125"
    }
]

```

## Example script

```python
from pymongo import MongoClient
from bson.objectid import ObjectId

# Connect to MongoDB
client = MongoClient("mongodb://localhost:27017/")
db = client['example_db']

# Define the aggregation pipeline
pipeline = [
    {"$match": {"status": "error"}},  # Filter documents to only those where status is 'error'
    {"$lookup": {
        "from": "datasets",
        "localField": "dataset_id",
        "foreignField": "_id",
        "as": "dataset_info"
    }},  # Join with datasets collection
    {"$unwind": "$dataset_info"},  # Flatten the array of joined documents
    {"$project": {
        "_id": 1,  # Include the _id field of the models collection
        "created_at": 1,
        "status": 1,
        "progress": 1,
        "name": "$dataset_info.name"
    }}  # Specify which fields to include in the output
]

# Execute the aggregation pipeline
results = list(db.models.aggregate(pipeline))
print(results)

```

## Aggregation pipeline의 구성

위에 적용된 aggregation을 설명한다.

### $match

model collection에서 status가 error인 document만 filter한다

### $lookup
Excel의 vlookup과 비슷하다. 
  - $match 결과에 해당하는 document를 "from" collection에서 찾는다. 
  - localField는 match를 적용한 models collection에서의 key이고, 
  - foreignField는 대상 "from" collection에서 dataset의 id를 담고있는 _id key를 찾는다.
  - lookup이 실행되고 나면 models collection에는 "as" 를 이름으로 가지는 새 필드가 아래처럼 추가 된다.

```json
{
    "_id": ObjectId("61398998ddhojkl1"),
    "created_at": "2024-04-25",
    "status": "error",
    "progress": 0.2,
    "dataset_id": ObjectId("5ldf23208098d001"),
    "dataset_info": [
        {
            "_id": ObjectId("5ldf23208098d001"),
            "name": "base_123"
        }
    ]
}
```

### $unwind
위 결과를 보면 dataset_info가 array 형태, [ ], 를 가진다. unwind는 형태를 아래와 같이 단순화 시켜준다.

```json
{
    "_id": ObjectId("61398998ddhojkl1"),
    "created_at": "2024-04-25",
    "status": "error",
    "progress": 0.2,
    "dataset_id": ObjectId("5ldf23208098d001"),
    "dataset_info": {
        "_id": ObjectId("5ldf23208098d001"),
        "name": "base_123"
    }
}

```

### $project

어떤 필드를 포함할지 결정한다.
1이면 포함, 0이면 제외한다.
아래와 같은 document가 있다고 할때
```json
{
    "_id": ObjectId("61398998ddhojkl1"),
    "created_at": "2024-04-25",
    "status": "error",
    "progress": 0.2,
    "dataset_id": ObjectId("5ldf23208098d001"),
    "dataset_info": {
        "_id": ObjectId("5ldf23208098d001"),
        "name": "base_123"
    }
}
```

아래처럼 project를 적용하면,
```json
{"$project": {
    "_id": 0,  // Exclude the _id field
    "created_at": 1,  // Include the created_at field
    "status": 1,  // Include the status field
    "progress": 1,  // Include the progress field
    "name": "$dataset_info.name"  // Include the name field from the dataset_info
}}
```

아래와 같은 결과를 얻는다
```json
{
    "created_at": "2024-04-25",
    "status": "error",
    "progress": 0.2,
    "name": "base_123"
}
```
