---
layout: post
title: Prometheus 및 Grafana를 설정하여 모니터링 및 시각화 구축하기
subtitle: 간단한 python script로 metric을 만들어 본다
categories: tech
tags: [prometheus,grafana,python,script]
---

## 소개
- 간단한 python 스크립트를 작성하고 prometheus_client 라이브러리를 이용해 metric을 추가한다.
- docker를 이용해 Grafana와 Prometheus를 설치한다.
- Prometheus에서 python 스크립트에 작성된 metric을 읽어올수 있도록 설정한다.
- Grafana에서 Prometheus를 새로운 Datasource로 추가하고 metric을 query해 본다.

## 준비 사항
- Mac 또는 Linux에서 실행한다.
- docker 및 docker-compose 설치
- Python 설치
- prometheus-client 설치

## 1. Sample python script 작성
- host의 cpu 사용량과 memory 사용량을 확인하기도 하고, 10000 이라는 investment 숫자에서 +,- 5% 변동을 5초간격으로 주고, 평균값과 현재값을 저장하는 스크립트다.
- port 8000 을 open하여 prometheus가 metric을 scripe 할 수 있게 한다.
- Gauge 는 매트릭 유형 중 하나다. 예를들어 'cpu_usage'가 metric이고 CPU Usage은 description이다.
```python
import psutil
from prometheus_client import start_http_server, Gauge
import time
import random

# Start an HTTP server to expose metrics
start_http_server(8000)

# Create Prometheus metrics for system and custom application metrics
cpu_usage = Gauge('cpu_usage', 'CPU Usage')
memory_usage = Gauge('memory_usage', 'Memory Usage')
average_roi_metric = Gauge('average_roi_metric', 'Average ROI')
current_roi_metric = Gauge('current_roi', 'Current ROI')

# Initialize variables for the financial analysis
investment = 10000  # Initial investment amount
roi_values = []     # List to store ROI values

while True:
    # Get CPU usage as a percentage
    cpu_percent = psutil.cpu_percent(interval=1)
    cpu_usage.set(cpu_percent)

    # Get memory usage as a percentage
    memory_percent = psutil.virtual_memory().percent
    memory_usage.set(memory_percent)

    # Simulate a financial analysis by calculating ROI
    # Here, we calculate a random ROI value between -5% and 5%
    roi = random.uniform(-5, 5) / 100
    investment += investment * roi  # Update the investment based on ROI
    roi_values.append(roi)

    # Calculate the average ROI over the last 30 data points
    if len(roi_values) > 30:
        roi_values.pop(0)
    average_roi = sum(roi_values) / len(roi_values)

    average_roi_metric.set(average_roi)
    current_roi_metric.set(roi)

    time.sleep(5)

```

## 2. Prometheus 설정
- prometheus 설치를 위한 빈 폴더를 만든다.
-  `docker-compose.yml` 파일을 만들어서 아래 내용을 추가한다.

```yaml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - 9090:9090
    restart: unless-stopped
    volumes:
      - ./conf:/etc/prometheus
volumes:
  prom_data:
```

- 이 구성에서는 Prometheus 이미지를 지정하고 Prometheus 웹 UI를 위해 포트 9090을 매핑하며 Prometheus 구성을 위한 볼륨을 마운트한다.

## 3. Prometheus config
- `conf` 폴더를 만들고 하위에 `prometheus.yml` 파일을 만들어 아래 내용을 추가한다.

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
alerting:
  alertmanagers:
  - static_configs:
    - targets: []
    scheme: http
    timeout: 10s
    api_version: v1
scrape_configs:
- job_name: 'python_metrics'
  static_configs:
    - targets: ["host.docker.internal:8000"]
```

- 이 구성은 Prometheus에게 `host.docker.internal:8000`에서 Metric을 scraping하도록 지시한다. 
- Python 스크립트가 8000 port를 open할 예정이다. `host.docker.internal:8000` 는 python 스크립트가 실행되었을때 접속하기 위한 주소이다.
- config 설명 참고 : https://prometheus.io/docs/prometheus/latest/configuration/configuration/

## 4. Prometheus 시작
- docker-compose.yml 파일이 있는 위치에서 다음 명령을 실행하여 Prometheus를 시작한다.

```bash
docker-compose up -d
```

- Prometheus가 실행되고 메트릭을 수집을 시작한다.

## 5. Grafana 설치
- 또다른 빈 폴더에 `docker-compose.yml` 파일을 만들고 아래 내용을 추가한다.

```yaml
version: "3.8"
services:
  grafana:
    image: grafana/grafana-enterprise
    container_name: grafana
    restart: unless-stopped
    ports:
      - 3000:3000
    volumes:
     - grafana-storage:/var/lib/grafana
volumes:
  grafana-storage: {}
```

- 이 구성은 Grafana 이미지를 지정하고 Grafana 웹 UI를 위한 포트 3000을 매핑하며 Grafana 데이터 저장을 위한 볼륨을 설정한다.

## 6. Grafana 시작
- 다음 명령을 실행하여 Grafana를 시작한다.

```bash
docker-compose up -d
```

- 이제 Grafana를 `http://localhost:3000`에서 접속할 수 있다. 기본 계정정보 (admin/admin)로 로그인하고 비밀번호를 변경한다.

## 7.  Prometheus Data source 추가
- Grafana에서 "Configuration" > "Data Sources"로 이동하고 "Add data source"를 클릭한다.
- "Prometheus"를 선택하고 다음과 같은 설정으로 구성한다.
	- 이름: Prometheus
	- URL: http://host.docker.internal:9090

- 연결을 확인하려면 "Save & Test"를 클릭한다.

## 8. Metric 시각화
이제 Metric을 시각화하기 위해 Grafana에서 대시보드를 만들 수 있다. 
Python 애플리케이션에서 `cpu_usage` 메트릭을 표시하려면 새 대시보드를 만들고 Prometheus 쿼리를 사용하여 `cpu_usage`를 표시하는 패널을 추가한다.

![webhooksite-result](/assets/images/posts/231104/231104-grafana_prometheus_datshboard.png)
