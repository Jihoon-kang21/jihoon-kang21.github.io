---
layout: post
title: python - Log Rotation 으로 Log 사이즈 제한하기
subtitle: RatatingFileHandler 활용하기
categories: Tech
tags: [python]
---

## Log Rotation이란?
Log Rotation은 Log File이 특정 크기에 도달하면 새로운 Log File로 교체하는 방법이다. 이 방법을 통해 Log File이 무한정 커지는 것을 방지하고, 디스크 공간을 효율적으로 관리할 수 있다. 새로운 Log File이 생성될 때, 이전 Log File은 백업 File로 저장된다.

## Log Rotation이 필요한 경우

### 디스크 공간 관리:
Log File이 너무 커지면 디스크 공간을 많이 차지하게 되어 시스템의 다른 작업에 영향을 줄 수 있다.

### Log File 읽기 및 관리:
매우 큰 Log File은 읽고 분석하기 어렵다. 작은 크기의 여러 File로 나누면 관리가 더 쉬워진다.

### 데이터 보존 및 보안:
오래된 Log File을 자동으로 삭제하거나 백업하여 보존 정책을 관리할 수 있다.

### RotatingFileHandler의 특징
RotatingFileHandler는 Python의 logging 모듈에서 제공하는 Handler 중 하나로, Log File Rotation을 자동으로 관리한다. 주요 특징은 다음과 같다:

1. **File 크기 제한**:
- maxBytes 매개변수를 통해 Log File의 최대 크기를 설정할 수 있다. File이 이 크기를 초과하면 Rotation이 발생한다.

2. **백업 File 수 관리**:
- backupCount 매개변수를 통해 유지할 백업 File의 최대 수를 설정할 수 있다. 설정된 수를 초과하는 백업 File은 삭제된다.

3. **자동 Rotation**:
- Log File이 최대 크기에 도달하면 자동으로 새로운 Log File을 생성하고, 이전 File을 백업한다.

4. **손쉬운 설정**:
- 코드에서 몇 가지 설정만으로 Log File Rotation을 구현할 수 있다.

## 상세 예제

다음은 Python에서 `RotatingFileHandler`가 어떻게 작동하는지에 대한 상세 예제이다.

### 단계별 설정

1. **Import**:
   ```python
   import logging
   from logging.handlers import RotatingFileHandler
   ```

2. **Logger 생성**:
   ```python
   logger = logging.getLogger('my_logger')
   logger.setLevel(logging.DEBUG)  # Logging 레벨을 DEBUG로 설정
   ```

3. **RotatingFileHandler 생성**:
   ```python
   output_path = 'my_log.log'  # Log File 경로
   file_handler = RotatingFileHandler(output_path, maxBytes=5 * 1024 * 1024, backupCount=10)
   file_handler.setLevel(logging.DEBUG)  # Handler 레벨을 DEBUG로 설정
   ```

4. **Formatter 생성 및 Handler에 설정**:
   ```python
   formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
   file_handler.setFormatter(formatter)
   ```

5. **Handler를 Logger에 추가**:
   ```python
   logger.addHandler(file_handler)
   ```

6. **Log 메시지 기록**:
   ```python
   for i in range(10000):
       logger.debug(f'This is debug message {i}')
   ```

### 전체 코드 예제
다음은 모든 코드를 하나로 합친 것이다:
```python
import logging
from logging.handlers import RotatingFileHandler

# Logger 생성
logger = logging.getLogger('my_logger')
logger.setLevel(logging.DEBUG)  # Logging 레벨을 DEBUG로 설정

# RotatingFileHandler 생성
output_path = 'my_log.log'  # Log File 경로
file_handler = RotatingFileHandler(output_path, maxBytes=5 * 1024 * 1024, backupCount=10)
file_handler.setLevel(logging.DEBUG)  # Handler 레벨을 DEBUG로 설정

# Formatter 생성 및 Handler에 설정
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
file_handler.setFormatter(formatter)

# Handler를 Logger에 추가
logger.addHandler(file_handler)

# Log 메시지 기록
for i in range(10000):
    logger.debug(f'This is debug message {i}')
```

## Log Rotation의 상세 설명

### 초기 상태:
- `my_log.log` (현재 Log File)

### 최대 크기를 초과한 후:

1. **첫 번째 Rotation**:
   - `my_log.log`가 5MB를 초과한다.
   - `my_log.log`는 `my_log.log.1`로 이름이 변경된다.
   - 새 `my_log.log`가 생성된다.
   - **File**:
     - `my_log.log` (새 File)
     - `my_log.log.1` (이전 File)

2. **두 번째 Rotation**:
   - `my_log.log`가 5MB를 초과한다.
   - `my_log.log.1`이 `my_log.log.2`로 이름이 변경된다.
   - `my_log`가 `my_log.log.1`로 이름이 변경된다.
   - 새 `my_log.log`가 생성된다.
   - **File**:
     - `my_log.log` (새 File)
     - `my_log.log.1`
     - `my_log.log.2`

3. **세 번째 Rotation**:
   - `my_log.log`가 5MB를 초과한다.
   - `my_log.log.2`가 `my_log.log.3`로 이름이 변경된다.
   - `my_log.log.1`이 `my_log.log.2`로 이름이 변경된다.
   - `my_log`가 `my_log.log.1`로 이름이 변경된다.
   - 새 `my_log.log`가 생성된다.
   - **File**:
     - `my_log.log` (새 File)
     - `my_log.log.1`
     - `my_log.log.2`
     - `my_log.log.3`

4. **네 번째 Rotation** (그리고 그 이후):
   - `my_log.log`가 5MB를 초과한다.
   - `my_log.log.3`가 삭제된다.
   - `my_log.log.2`가 `my_log.log.3`로 이름이 변경된다.
   - `my_log.log.1`이 `my_log.log.2`로 이름이 변경된다.
   - `my_log`가 `my_log.log.1`로 이름이 변경된다.
   - 새 `my_log.log`가 생성된다.
   - **File**:
     - `my_log.log` (새 File)
     - `my_log.log.1`
     - `my_log.log.2`
     - `my_log.log.3`

## 요약

- `RotatingFileHandler`는 Log File 크기를 관리하고, 현재 Log File이 지정된 크기를 초과할 때 백업 File을 생성한다.
- 지정된 백업 File 수 (`backupCount`)를 초과하지 않도록 가장 오래된 File을 삭제한다.