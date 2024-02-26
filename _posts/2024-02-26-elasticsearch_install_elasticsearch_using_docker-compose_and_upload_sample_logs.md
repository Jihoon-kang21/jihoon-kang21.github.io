---
layout: post
title: Elasticsearch - docker-compose로 설치하고 sample log 만들어서 업로드하기
subtitle: python, docker를 이용해서 간단히 따라하기
categories: Tech
tags: [elasticsearch,docker,python]
---

### docker-compose로 elasticsearch 설치

- node를 하나만 사용할 것이므로 single-node 옵션을 추가한다.
- container가 종료되어도 data를 보존하려면 volume를 정의한다.
```yml
version: "3"
services:
    elasticsearch:
        ports:
            - 127.0.0.1:9200:9200
            - 127.0.0.1:9300:9300
        environment:
            - discovery.type=single-node
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.16
        volumes:
          - elasticsearch-data:/usr/share/elasticsearch/data
volumes:
    elasticsearch-data:
```

### sample log 생성

log를 만들어서 sample_logs.json 파일로 저장한다.

generate_logs.py
```python
import json
from datetime import datetime, timedelta
import random

# Sample fields for logs
fields = ["status", "service_name", "response_time"]

# Generate sample logs
logs = []
for i in range(101):  # Generate more than 100 logs
    log = {
        "timestamp": (datetime.now() - timedelta(days=random.randint(0, 100))).isoformat(),
        "status": random.choice(["success", "error"]),
        "service_name": random.choice(["auth-service", "data-service", "payment-service"]),
        "response_time": random.randint(100, 2000)  # Response time in milliseconds
    }
    logs.append(json.dumps(log))

# Output logs to a file
with open('sample_logs.json', 'w') as f:
    for log in logs:
        f.write(log + "\n")
```


### elastic search로 log 업로드

- sample_logs.json 파일에 저장된 log를 업로드 한다. index 이름은 logs-index 로 저장한다.
- port는 docker-compose.yml에서 열어둔 9200번을 사용한다.

upload_logs.py
```python
import json
import requests

# Elasticsearch URL
es_url = 'http://localhost:9200'
index_name = 'logs-index'  # Name of the index where logs will be stored

# Load logs from file
with open('sample_logs.json', 'r') as file:
    logs = file.readlines()

# Ingest logs into Elasticsearch
headers = {'Content-Type': 'application/json'}
for log in logs:
    response = requests.post(f'{es_url}/{index_name}/_doc', headers=headers, data=log)
    if response.status_code == 201:
        print("Log ingested successfully")
    else:
        print(f"Failed to ingest log: {response.content}")
```

### 브라우저에서 upload한 log 확인하기

아래 주소로 접속해서 잘 저장되었는지 확인한다.
```
# index list 확인하기
http://localhost:9200/_cat/indices

# index 에 대한 정보 확인
http://localhost:9200/{index}?pretty

# log 내용 확인
http://localhost:9200/{index}/_search
```


