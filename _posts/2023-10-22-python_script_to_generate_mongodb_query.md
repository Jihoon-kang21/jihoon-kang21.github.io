---
layout: post
title: python-MongoDB query 문 대신 써주는 스크립트
subtitle: 비슷한 query를 자주 써야하는 경우 검색할 key, value만 입력하면 query문을 완성해준다
categories: tech
tags: [python,script,mongodb]
---

## script 용도 및 배경

Python으로 query를 하든 GUI interface에서 query를 하든 수시로 query해야하는데 따옴표, 중괄호, 콜론 등을 쓰는게 너무 번거로웠다. key, value 값만 입력하면 query 문을 완성해주는 스크립트를 썼다.
regex는 굉장히 제한적이지만, 추가가 가능하다.
스크립트 작성 내용을 순서대로 설명하면 아래와 같다.
전체 스크립트는 여기 : https://github.com/Jihoon-kang21/scripts/blob/main/mongodb_query_generator.py

## 1 : 필요한 라이브러리 가져오기

스크립트에 필요한 라이브러리를 가져온다.
다음의 라이브러리들은 데이터베이스에 연결하고 Document 식별자를 관리하며 query 결과를 보기좋게 만들어 준다.

### `MongoClient`: MongoDB에 연결

`MongoClient`는 Python에서 MongoDB 서버에 연결하고 통신하기 위한 출발점 역할을 한다. `MongoClient` 객체를 생성하면 connection string에 지정된 MongoDB 서버에 연결합니다. 이 connection string에는 일반적으로 서버 주소, 포트 및 인증 정보에 대한 정보가 포함되어 있다.

아래에서 connection string은 `mongodb://your-connection-details-here` 이다. 

```python
from pymongo import MongoClient

# MongoDB에 연결
client = MongoClient("mongodb://your-connection-details-here")
```

이 스크립트를 사용하려면 `"mongodb://your-connection-details-here"`를 실제 MongoDB 서버정보로 바꿔주면 된다.

### `ObjectId`: Document 고유 식별자

`ObjectId`는 MongoDB의 기본 구성 요소이다. MongoDB는 새 Document를 삽입할 때 이 식별자를 자동으로 생성하여 특정 Document를 식별하고 query한다.
이 스크립트에서는 `bson` 라이브러리에서 `ObjectId` 클래스를 가져와 Document 식별자를 다룬다. 예를 들어 아래처럼 정의할 수 있다.

```python
from bson import ObjectId

document_id = ObjectId("5f8dd6d038f634be88a0d2e1")
```

### `pprint`: 아름답게 서식 지정된 출력

복잡한 데이터 구조, dictionary, list 등을 출력할때 보기좋게 서식을 적용해서 출력해 준다. pprint가 아닌 print를 쓰면 데이터 내용을 한줄에 모두 표현해서 읽기가 힘들다.
다음은 `pprint`의 사용법 예시:

```python
from pprint import pprint

data = {
    'name': 'John Doe',
    'age': 30,
    'address': {
        'street': '123 Main St',
        'city': 'Exampleville',
        'zipcode': '12345'
    }

}

# 데이터 출력을 위해 pprint 사용
pprint(data)
```

아래처럼 별과를 보여준다.

```plaintext
{'address': {'city': 'Exampleville',
             'street': '123 Main St',
             'zipcode': '12345'},
 'age': 30,
 'name': 'John Doe'}
```

## 2 : MongoDB 로 연결

아래는 필요한 라이브러리를 불러온 다음 MongoDB 서버 정보를 이용해 Database에 접근하는 단계이다.
```python
# Establish a connection to MongoDB
client = MongoClient("mongodb://your-connection-details-here")
db = client.get_database()  # Replace with your actual database name
```

## 3 : 필요한 함수 정의
### 1) convert_input_value : 사용자 입력을 query로 변환하는 함수

`convert_input_value()` 함수를 만들어 사용자 입력을 BSON query로 변환합니다. 

```python
def convert_input_value(input_value):
    if input_value.startswith("$gte:") and len(input_value) > len("$gte:"):
        date_str = input_value[len("$gte:"):]
        try:
            return {"$gte": datetime.fromisoformat(date_str)}
        except ValueError:
            # Handle an invalid date format
            return None
    elif input_value.startswith("$in:"):
        return {"$in": input_value[len("$in:"):].split(',')}
    elif input_value.lower() == "true":
        return True
    elif input_value.lower() == "false":
        return False
    elif ObjectId.is_valid(input_value):
        return ObjectId(input_value)
    elif input_value.isdigit():
        return int(input_value)
    else:
        return input_value

```

적용된 내용은 아래와 같다
#### 날짜 비교 처리

1. 사용자 입력이 `$gte:` (greater than equal)로 시작하는지 확인한다. `datetime.fromisoformat()` 메서드를 사용하여 파이썬 `datetime` 객체로 변환된다. 변환이 성공하면 이상 또는 같음 날짜를 나타내는 query를 생성한다. 날짜 필드가 있는 Document를 query하여 특정 날짜 이후의 기록을 찾을 때 유용하다.

   예제 입력: `$gte:2023-01-01`
   결과 query: `{"$gte": datetime.datetime(2023, 1, 1)}`

   이 경우, 스크립트는 입력을 사용하여 2023년 1월 1일 이후의 Document를 query하는 query를 생성한다.

#### 목록 포함 처리

2. 사용자 입력이 "$in:"으로 시작하면 함수는 입력한 여러 값들 중 하나만 일치해도 query 할 수 있도록 한다. 

   예제 입력: `$in:사과,바나나,오렌지`
   결과 query: `{"$in": ["사과", "바나나", "오렌지"]}`

   이 예제에서 스크립트는 "사과," "바나나," 또는 "오렌지" 중 하나인 필드 값을 가지는 Document를 찾는 query를 생성한다.

#### boolean 처리

3. 입력이 "true" 또는 "false" (대소문자 구분 없음)와 일치하는지 확인한다. 일치하는 경우 해당 값인 `True` 또는 `False`를 반환한다. 

   예제 입력: `true`
   결과: `True`

#### ObjectId 처리

4. 입력값이 ObjectId로 인식되면 함수는 ObjectId 를 query할 수 있도록 한다.

   예제 입력: `5f8dd6d038f634be88a0d2e1`
   결과: `ObjectId("5f8dd6d038f634be88a0d2e1")`

   이 경우, 스크립트는 입력을 ObjectId로 인식하고 해당 입력을 query에 사용할 수 있도록 만들어 준다.

#### int 처리

5. 입력이 숫자로 구성된 경우 (예: "123" 또는 "-456"), 정수 타입의 값으로 변환한다. 

   예제 입력: `123`
   결과: `123`

#### 기본 동작

6. 위의 어떤 조건도 일치하지 않으면 함수는 입력을 그대로 반환합니다. string 값을 별다른 조건없이 적용하는 경우이다.

   예제 입력: `사과`
   결과: `"사과"`

   이 경우, 스크립트는 입력을 텍스트 값으로 처리하고 query에 그대로 포함시킨다.

### 2) get_sort_criteria : 결과 정렬 함수

출력될 결과에 대해정렬기준 값과 정렬 방식(ascending or descending)을 입력할 수 있다.

```python
def get_sort_criteria():
    sort_key = input("Enter a sorting key (or press Enter to skip): ")
    if sort_key:
        sort_order = int(input("Enter the sorting order (1 for ascending, -1 for descending): "))
        return [(sort_key, sort_order)]
    return None
```

### 3) get_keys_to_exclude : 특정키 제외하는 함수

어떤 key가 너무 방대한 데이터를 포함하고 있는 경우, 그리고 그 key 내용을 볼 필요가 없는 경우에 제외하고 출력할 수 있다.

```python
def get_keys_to_exclude():
    keys_to exclude = input("Enter keys to exclude (comma-separated, or press Enter to skip): ")
    if keys_to_exclude:
        return keys_to_exclude.split(',')
    return None
```

## 4 : 사용자 input을 받아 함수 실행

```python
# Fetch the keys for the specified collection
sample_doc = db[collection_name].find_one()
collection_keys = list(sample_doc.keys()) if sample_doc else []

# Display the available keys for the specified collection
print(f"Available keys for the collection '{collection_name}':")
print(", ".join(collection_keys))
```
- 이 부분은 지정한 collection에서 예제 Document(첫 번째 Document)를 가져온다. 그런 다음 이 예제 Document에서 키(필드 이름)를 추출하고 이를 `collection_keys`에 저장한다.
- 스크립트를 실행했을때 사용자가 입력한 collection에 존재하는 key 리스트를 보여주는 역할을 한다.

```python
# Get user input for key-value pairs
query_dict = {}
while True:
    key = input("Enter a key (or press Enter to finish): ")
    if not key:
        break
    value = input("Enter the value: ")

    query_dict[key] = convert_input_value(value)
```
- 이 섹션에서 스크립트는 사용자가 query를 위해 여러개의 키-값 쌍을 입력할 수 있도록 반복문에 들어간다. 그래서 사용자는 query할 때 필터할 key를 여러개 입력할 수 있다.
- 사용자에게 키를 입력하라는 메시지가 표시되며, 사용자가 키를 제공하지 않고 Enter 키를 누르면 반복문이 종료된다.
- 입력한 각 키에 대해 사용자가 값을 입력하도록 메시지가 나타나며, 이 값을 `convert_input_value()` 함수를 사용하여 처리하고 `query_dict`에 저장합니다.

```python
sort_criteria = get_sort_criteria()
```
- 정렬기준을 입력할 때  `get_sort_criteria()` 함수를 사용하도록 한다.

```python
keys_to_exclude = get_keys_to_exclude()
```
- 스크립트 결과에서 특정 키를 제외하고 출력하기 위해  `get_keys_to_exclude()` 함수를 사용하도록 한다.

```python
limit_str = input("Enter limit (or press Enter to skip): ")
limit = int(limit_str) if limit_str else None
```
- 사용자에게 결과 표시 제한을 입력하라는 메시지가 표시된다. Document가 너무 많은 collection를 조회하는 경우에 대비해 추가된 부분이다. 사용자가 별도 값 입력없이 Enter 키를 누르면, 이 값은 `None`으로 설정된다.

```python
distinct_option = input("Do you want to retrieve distinct values for a specific key? (yes/no): ")

if distinct_option.lower() == "yes":
    key_to_inspect = input("Enter the key for which you want to see distinct values: ")
    # Query the database using pymongo to get distinct values for the specified key
    collection = db[collection_name]
    distinct_values = get_distinct_values(collection, key_to_inspect)

    # Display the distinct values
    if distinct_values:
        print(f"Distinct values for '{key_to_inspect}':")
        for value in distinct_values:
            print(value)
    else:
        print(f"No distinct values found for '{key_to_inspect}' in the collection '{collection_name}'.")
else:
    print("Distinct values retrieval skipped.")
```
- 스크립트는 사용자로 하여금 특정 키에 대한 고유값을 검색할 것인지 묻는다. 
- 예를 들어 genre라는 key에 fiction이라는 value를 검색했을때 fiction에 해당하는 결과가 없는 경우, 스크립트는 아무런 결과물을 주지 않는다. 이런 상황에서 fiction 이외에 어떤 value들이 존재하는지 궁금한 경우 활용할 수 있다.
- 사용자가 "yes"라고 답하면, 고유값을 볼 키를 입력하도록 한다. 그런 다음 데이터베이스를 쿼리하여 PyMongo의 `distinct` 메서드를 사용하여 결과를 표시한다.
- 사용자가 "no"라고 대답하면, skip한다.

```python
# Print the entire generated query
print("Generated Query:")
pprint(query_dict)
```
- 사용자 입력값을 고려하여 만들어진 전체 query가 어떻게 생겼는지 보여준다.

```python
# Query the database using pymongo (for constructing the query)
collection = db[collection_name]
results = collection.find(query_dict)

# Apply sorting criteria if provided
if sort_criteria:
    results = results.sort(sort_criteria)
```
- 여기서 스크립트는 사용자 입력을 기반으로 PyMongo의 `find` 메서드를 사용하여 query를 작성한다.
- 사용자가 정렬 기준을 입력한 경우, PyMongo의 `sort` 메서드를 사용하여 결과에 정렬을 적용한다.

```python
total_count = collection.count_documents(query_dict)
```
- PyMongo의 `count_documents` method를 사용해서 query 결과가 몇개의 document인지 세어서 total_count 변수에 저장한다. limit를 고려하지 않고 숫자를 센다.

```python
# Conditionally limit the results based on user input
if limit is not None:
    results = results.limit(limit)
```
- query 결과를 사용자가 입력한 limit 값만급 출력할 수 있도록 한다.

```python
# Print the results with excluded keys
for result in results:
    result_to_display = result.copy()
    if keys_to_exclude:
        for key in keys_to_exclude:
            result_to_display.pop(key, None)
    pprint(result_to_display)
```
- query 결과는 cursor 라는 object로 만들어진다. 이 object은 그냥 print할 수 없고 loop를 사용해서 출력이 가능하다.
- 사용자가 결과에서 제외할 key를 입력한 경우, 해당 키를 제외하고 출력한다.

```python
print(f"Total count of matching documents: {total_count}")
```
- 실행한 query에 해당하는 document 수를 보여준다.
## 9: 스크립트 사용 예시

아래와 같은 예시 MongoDB collection 과 document들이 있다고 할때, 스크립트를 사용 예시이다.
### Example MongoDB Collections and Documents:

**Collection:books**

```json
{
   "_id": ObjectId("5f8dd6d038f634be88a0d2e5"),
   "title": "The Great Gatsby",
   "author": "F. Scott Fitzgerald",
   "category": "fiction",
   "price": 25.99,
   "publish_date": "2022-05-15"
}
{
   "_id": ObjectId("5f8dd6d038f634be88a0d2e6"),
   "title": "To Kill a Mockingbird",
   "author": "Harper Lee",
   "category": "fiction",
   "price": 19.99,
   "publish_date": "2021-11-03"
}
{
   "_id": ObjectId("5f8dd6d038f634be88a0d2e7"),
   "title": "1984",
   "author": "George Orwell",
   "category": "fiction",
   "price": 14.99,
   "publish_date": "2022-03-20"
}
{
   "_id": ObjectId("5f8dd6d038f634be88a0d2e8"),
   "title": "Pride and Prejudice",
   "author": "Jane Austen",
   "category": "fiction",
   "price": 29.99,
   "publish_date": "2023-02-10"
}
```

### 스크립트 사용해보기

위와 같은 DB에서 장르가 소설이고,  날짜가 22년 1월 1일 이후인 책들을 이름 순대로 나열하고 싶다고 하자.
스크립트를 실행하고 아래처럼 입력한다.

- Collection name: `books`
- Key 1: `category`
- Value 1: `fiction`
- Key 2: `publish_date`
- Value 2: `$gte:2022-01-01`
- Sorting key: `title`
- Sorting order: `1` (for ascending)
- Keys to exclude: None
- Limit: None
- Distinct values: No (skip)

### 스크립트 결과

결과는 아래와 같다.

```json
Generated Query:
{'category': 'fiction', 'publish_date': {'$gte': datetime.datetime(2022, 1, 1)}}

Matching documents in 'books' collection:
{
   "_id": ObjectId("5f8dd6d038f634be88a0d2e8"),
   "title": "Pride and Prejudice",
   "author": "Jane Austen",
   "category": "fiction",
   "price": 29.99,
   "publish_date": "2023-02-10"
}
{
   "_id": ObjectId("5f8dd6d038f634be88a0d2e5"),
   "title": "The Great Gatsby",
   "author": "F. Scott Fitzgerald",
   "category": "fiction",
   "price": 25.99,
   "publish_date": "2022-05-15"
}

Total count of matching documents: 2

```


