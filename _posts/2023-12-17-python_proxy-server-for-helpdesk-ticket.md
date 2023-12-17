---
layout: post
title: webhook을 전달받아 포멧을 변경하여 API request 날리기
subtitle: fastAPI와 httpx 활용하기
categories: Code Example
tags: [python,fastapi]
---


## Introduction

helpdesk 에서 ticket을 만들때, technician이 할당되었을때, ticket이 closed 됐을때 webhook을 쏴주고,
아래 파이썬 코드는 이 webhook을 받아서
ITSM 시스템이 요구하는 API 에 맞춰서 자동으로 request를 날려준다.
## Setting Up the Environment

fastapi, httpx, psycopg가 사용되었다. 미리 설치 한다.

```bash
pip install fastapi httpx psycopg2-binary
```

## Code Breakdown

### 1. Database Initialization

```python
import datetime
import json
import logging
import httpx
import fastapi
import psycopg2
```

postgresDB를 쓴 이유는 기존에 설치된 DB가 있었기 때문.
새 ticket이 helpdesk에서 만들어질 때 응답으로 ITSM의 request id를 받게된다. 그 후 technician을 assign하거나 complete 처리 시킬때 ticket id와 request id를 맵핑하기 위해 DB를 썼다.
아래는 DB에 연결하고, table을 만들고, 날짜 설정하는 부분.
expected date는 ITSM에서 요구하는 값이고, 담당자를 할당할때 기본으로 7일 후에는 처리한다는 뜻이다.
```python
conn = psycopg2.connect(
    database="monitoring",
    user="postgres",
    password="postgres",
    host="postgresserveraddress",
    port="5000",
    connect_timeout=5
)
cursor = conn.cursor()

cursor.execute(
    '''CREATE TABLE IF NOT EXISTS ticket_mapping (
    ticket_id TEXT PRIMARY KEY,
    request_id TEXT,
    timestamp TIMESTAMPTZ
    )'''
)
conn.commit()

now = datetime.datetime.now()
today = now.strftime('%Y-%m-%d')

expected_date_raw = now + datetime.timedelta(days=7)
expected_date = expected_date_raw.strftime('%Y-%m-%d')

# ... (remaining code)
```

### 2. Initializing FastAPI

```python
app = fastapi.FastAPI()
_logger = logging.getLogger("uvicorn.helpdesk")
```

fastapi는 API 서버를 만들기 위한 라이브러리.
logger는 각 액션에 대해 로그를 남기기 위해 사용된다.

### 3. ITSM API URL and Header Function

```python
itsmurl = "http://itsm.com/api"

def itsm_header() -> dict:
    res = dict()
    res["Authorization"] = "Basic c25wqkasd093-base64token"
    return res
```

### 4. Functions for Creating ITSM Tickets

ITSM에 요구하는 스펙에 값을 끼워넣는 함수들이다.
newticket에서 writer, category, subject, contents는 helpdesk에서 webhook으로 날려준 값이고,
code, item_id, service, req_user_id 등은 ITSM에서 요구하는 값들이다.

```python
def itsm_newticket(writer: str, category: str, subject: str, contents: str) -> dict:
    res = dict()

    res["code"] = "000"
    res["item_id"] = "카탈로그 아이템 번호"
    res["service"] = "서비스 번호"
    res["req_user_id"] = writer
    res["reg_user_id"] = writer
    res["request_purpose"] = category
    res["priority"] = 2
    res["end_hop_date"] = expected_date
    res["title"] = subject
    res["contents"] = contents
    res["after_improvement"] = "없음"
    return res

def itsm_assign(ticket_id: str, technician: str) -> dict:
    
    cursor.execute("SELECT request_id FROM ticket_mapping WHERE ticket_id= ?", (ticket_id,))
    result_id_tuple = cursor.fetchone()
    request_id = result_id_tuple[0]
    
    res = dict()
    res["code"] = "020"
    res["request_id"] = request_id
    res["rcv_user_id"] = technician
    res["trt_user_id"] = technician
    res["end_expect_date"] = expected_date
    res["comment"] = "요청이 접수되었습니다."
    return res

def itsm_complete(ticket_id: str, date: str) -> dict:
    
    cursor.execute("SELECT request_id FROM ticket_mapping WHERE ticket_id= ?", (ticket_id,))
    result_id_tuple = cursor.fetchone()
    request_id = result_id_tuple[0]

    open = datetime.datetime.strptime(date, "%m%d%Y %I:%N:%S %p")
    timeDifference = now - open
    hoursFloat = timeDifference.total_seconds() / 3600
    hoursInt = int(hoursFloat)
    hours = str(hoursInt)

    res = dict()
    res["code"] = "030"
    res["request_id"] = request_id
    res["work_qty"] = hours
    res["comment"] = "처리가 완료되었습니다."
    return res

# ... (remaining code)
```


### 5. FastAPI Endpoints

helpdesk에서 /itsm_new, /itsm_ass, /itsm_com에 각각 webhook을 날리게 되면 httpx가 위에서 정의한 itsm_newticket, itsm_assign, itsm_complete 함수로 body를 만들고 itsmurl로 API request를 보낸다.

```python
@app.post("/itsm_new")
async def itsm_new(req: Request):
    try:
        hd_data = await req.json()
        
    except Exception as e:
        _logger.exception("Fetching a request failed:")
        return JSONResponse(str(e), status_code=401)

    _logger.info("API call for /itsm_new: Incoming=%s", hd_data)
    ticket_id = hd_data["ticket_id"]
    writer = hd_data["writer"]
    category = hd_data["category"]
    subject = hd_data["subject"]
    contents = hd_data["contents"]
    
    async with httpx.AsyncClient(timeout=60.0) as client:
        itsm_headers = itsm_header()
        itsm_body = itsm_newticket(writer, category, subject, contents)
        _logger.debug("Try to send ITSM new request: url=%s, header=%s, body=%s", itsmurl, itsm_headers, itsm_body)
        res = await client.post(itsmurl, headers=itsm_header(), json=itsm_body)
        if res.is_client_error:
            _logger.error("Failed to send ITSM new ticket API request due to client error: %s", res.text)
            return {}
        elif res.is_server_error:
            _logger.warning("Failed to send ITSM new ticket API request due to server error: %s", res.text)
            return {}
        else:
            _logger.info("ITSM new ticket API request is sent successfully.")
            response_data = res.json()
            _logger.info(f"Response: {response_data}")
            request_id = response_data.get("result", {}).get("request_id", "")
            _logger.info(f"Request ID: {request_id}")

            cursor.execute('''
                INSERT OR REPLACE INTO ticket_mapping (ticket_id, request_id, timestamp) VALUES (%s, %s, %s)
            ''', (ticket_id, request_id, now))
            conn.commit()
            
    return JSONResponse(hd_data)

@app.post("/itsm_ass")
async def itsm_ass(req: Request):
    try:
        hd_data = await req.json()
        
    except Exception as e:
        _logger.exception("Fetching a request failed:")
        _logger.error(f"hd_data: {req}")
        _logger.error

(f"Error in itsm_ass: {str(e)}")
        return JSONResponse(str(e), status_code=401)

    _logger.info("API call for /itsm_ass: incoming=%s", hd_data)
    ticket_id = hd_data["ticket_id"]
    technician = hd_data["technician"]
    
    async with httpx.AsyncClient(timeout=60.0) as client:
        itsm_headers = itsm_header()
        itsm_body = itsm_assign(ticket_id, technician)
        _logger.debug("Try to send ITSM assign request: url=%s, header=%s, body=%s", itsmurl, itsm_headers, itsm_body)

        res = await client.post(itsmurl, headers=itsm_headers, json=itsm_body)
        if res.is_client_error:
            _logger.error("Failed to send ITSM assign API request due to client error: %s", res.text)
            return {}
        elif res.is_server_error:
            _logger.warning("Failed to send ITSM assign API request due to server error: %s", res.text)
            return {}
        else:
            _logger.info("ITSM Assigning Technician API request is sent successfully.")
            response_data = res.json()
            _logger.info(f"Response: {response_data}")
            
    return JSONResponse(hd_data)

@app.post("/itsm_com")
async def itsm_com(req: Request):
    try:
        hd_data = await req.json()
        
    except Exception as e:
        _logger.exception("Fetching a request failed:")
        return JSONResponse(str(e), status_code=401)
    _logger.debug("ITSM Complete Issue API call for /itsm_com: incoming=%s", hd_data)

    ticket_id = hd_data["ticket_id"]
    date = hd_data["data"]
    
    async with httpx.AsyncClient(timeout=60.0) as client:
        itsm_headers = itsm_header()
        itsm_body = itsm_complete(ticket_id,date)
        _logger.debug("Try to send ITSM complete request: url=%s, header=%s, body=%s", itsmurl, itsm_headers, itsm_body)
        res = await client.post(itsmurl, headers=itsm_headers, json=itsm_body)
        if res.is_client_error:
            _logger.error("Failed to send ITSM complete API request due to client error: %s", res.text)
            return {}
        elif res.is_server_error:
            _logger.warning("Failed to send ITSM complete API request due to server error: %s", res.text)
            return {}
        else:
            _logger.info("ITSM Complete Issue API request is sent successfully")
            response_data = res.json()
            _logger.info(f"Response: {response_data}")
            
    return JSONResponse(hd_data)

@app.on_event("startup")
async def start_up():
    _logger.setLevel(logging.DEBUG)
```


## Logging in Python

### Basics of Logging

#### 1. **Importing the Logging Module**

   ```python
   import logging
   ```

#### 2. **Setting Up a Logger**

   ```python
   logger = logging.getLogger("my_logger")
   ```

#### 3. **Configuring Logging Level**

   ```python
   logger.setLevel(logging.DEBUG)
   ```

레벨에는 `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` 이 있다.
Debug로 설정하면 디버그를 포함한 상위 단계 로그를 모두 logging한다.

#### 4. **Creating Log Handlers**

handler는 로그메시지를 어디에 출력할지 정의한다. StreamHandler는 console에, FileHandler는 파일에 남기기위에 사용한다.
   ```python
   stream_handler = logging.StreamHandler()
   file_handler = logging.FileHandler("my_log.log")
   ```

#### 5. **Setting Up Log Formatting**

로그 메시지마다 메시지 앞에 시간, 로그 레벨을 함께 출력하는 등의 포멧을 결정한다.
   ```python
   formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')
   stream_handler.setFormatter(formatter)
   file_handler.setFormatter(formatter)
   ```

#### 6. **Adding Handlers to the Logger**

logger에 handler를 추가한다.
   ```python
   logger.addHandler(stream_handler)
   logger.addHandler(file_handler)
   ```

### Log Rotation

한개 로그 파일에 무한정 저장하면 파일이 너무 커지고 관리하기 힘들어진다. 5메가 마다 새로운 파일을 만들고, 20개까지 저장하기 위한 옵션을 추가한다.

```python
rotating_file_handler = logging.handlers.RotatingFileHandler('./my_log.log', maxBytes=1024*1024*5, backupCount=20)
logger.addHandler(rotating_file_handler)
```

## Using psycopg2 (PostgreSQL Database)

### Installation

```bash
pip install psycopg2-binary
```

### Connection to a PostgreSQL Database

PostgresDB가 설치되어있다면 그 정보를 통해 DB에 접속할 수 있다.
```python
import psycopg2

conn = psycopg2.connect(
    database="your_database_name",
    user="your_username",
    password="your_password",
    host="your_host",
    port="your_port"
)
cursor = conn.cursor()
```

### Executing SQL Queries

연결정보를 담은 cursor를 통해 query를 날린다.
select query 결과는 fetch 함수를 이용해 변수로 저장가능하다.
```python
cursor.execute("SELECT * FROM your_table;")
records = cursor.fetchall()

for record in records:
    print(record)
```

### Committing Changes and Closing Connection

insert, update등의 쿼리를 쓰는 경우, 쿼리 후 commit를 실행해야한다.
close는 연결을 종료하는데 사용한다.
```python
conn.commit()
conn.close()
```

## Using FastAPI

### Installation

```bash
pip install fastapi uvicorn
```

### Creating a Simple FastAPI App

아래는 간단한 FastAPI APP을 만드는 코드이다.
/와 /items/{itsm_id} 두 가지 endpoint를 생성한다. 로컬에 서버 띄우면 http://localhost/, http://localhost/items/1 형태로 api를 이용할 수 있게 된다.
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

@app.get("/items/{item_id}")
def read_item(item_id: int, query_param: str = None):
    return {"item_id": item_id, "query_param": query_param}
```

main.py 라는 파일로 위 코드를 저장했다면 아래 명령어로 API 서버를 실행할 수 있다.
```bash
uvicorn main:app --reload
```
위 명령어로 실행하면 `http://127.0.0.1:8000/docs` 에서 Swagger 문서 확인이 가능하고 `http://127.0.0.1:8000/redoc` 에서 ReDoc 을 조회할 수 있다.

## Using httpx

httpx는 client 역할을 한다. 다른 api 서버에 request를 날릴때 쓴다.
### Installation

```bash
pip install httpx
```

### Making a Simple HTTP GET Request

아래는 url에 get request를 날리는 예시이다.
```python
import httpx

async def fetch_data():
    url = "https://jsonplaceholder.typicode.com/posts/1"
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
    print(response.status_code)
    print(response.json())

# Run the event loop
import asyncio
asyncio.run(fetch_data())

```


