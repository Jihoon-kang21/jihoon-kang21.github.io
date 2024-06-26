---
layout: post
title: pytest 기본 사용법
subtitle: 설치 및 실행법
categories: Tech
tags: [python,pytest]
---

## pytest란?

pytest는 간단하고 확장 가능한 테스트 케이스를 쉽게 작성할 수 있게 해주는 강력한 테스트 프레임워크이다. 작은 단위 테스트부터 복잡한 기능 테스트까지 처리할 수 있다.

### pytest의 주요 기능:

- 간단하고 읽기 쉬운 문법
- 자동 테스트 발견
- 테스트 설정 및 해제를 위한 Fixture
- 매개변수화된 테스트 지원
- 확장 기능을 위한 광범위한 플러그인

## pytest 설치

테스트를 작성하기 전에 pytest를 설치해야 한다. python 패키지 설치 도구인 pip를 사용하여 pytest를 설치할 수 있다. 터미널 또는 명령 프롬프트를 열고 다음 명령을 실행한다:

```bash
pip install pytest
```

## 첫 번째 테스트 작성

pytest가 어떻게 작동하는지 이해하기 위해 간단한 예제로 시작해 본다. 두 숫자를 더하는 `add`라는 함수가 있다고 가정한다. 이 함수가 제대로 작동하는지 확인하는 테스트를 작성해 본다.

### 간단한 애플리케이션 코드 (`app.py`):

```python
def add(a, b):
    return a + b
```

### 테스트 작성 (`test_app.py`):

`app.py`와 동일한 디렉토리에 `test_app.py`라는 새 파일을 만든다. 이 파일에서 `add` 함수에 대한 테스트를 작성할 것이다.

```python
from app import add

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0
```

`test_add` 함수에서 `assert` 문을 사용하여 `add` 함수가 예상한 결과를 반환하는지 확인한다. 이러한 assert 중 하나라도 실패하면 pytest가 오류를 뱉는다..

### 테스트 실행

테스트를 실행하려면 터미널 또는 명령 프롬프트를 열고 `test_app.py`가 있는 디렉토리로 이동한 다음 다음 명령을 실행한다:

```bash
pytest
```

pytest는 자동으로 테스트 함수를 발견하고 실행할 것이다. 모든 assert가 통과하면 테스트가 성공했다는 출력이 표시된다. assert가 실패하면 pytest는 실패에 대한 자세한 정보를 보여준다.

## Fixtures

fixture는 테스트 실행 전에 필요한 준비 작업을 설정하거나, 테스트 후 정리 작업을 수행하는 데 사용한다. 이는 반복적인 코드 작성 없이 재사용 가능한 설정을 제공함으로써 코드의 중복을 줄이고 유지 보수를 용이하게 한다.

### 사용 사례: 데이터베이스 연결 테스트

예를 들어, 데이터베이스와의 연결이 필요한 여러 개의 테스트가 있다고 가정해 본다. 각 테스트에서 데이터베이스 연결을 설정하고 해제하는 코드를 반복적으로 작성하는 대신, fixture를 사용하여 이를 한 번만 설정하고 여러 테스트에서 재사용할 수 있다.

```python
# conftest.py
import pytest
from my_database import DatabaseConnection

@pytest.fixture
def db_connection():
    connection = DatabaseConnection()
    connection.connect()
    yield connection
    connection.disconnect()
```

이제 테스트에서 `db_connection` fixture를 사용할 수 있다.

```python
# test_app.py
def test_insert_data(db_connection):
    result = db_connection.insert("some_data")
    assert result == "success"

def test_query_data(db_connection):
    result = db_connection.query("some_query")
    assert result == "expected_data"
```

여기서 `db_connection` fixture는 각 테스트 함수가 실행될 때마다 자동으로 호출되어 데이터베이스 연결을 설정하고 테스트가 끝난 후에는 연결을 해제한다.

## 파라미터화 (Parametrize)

파라미터화 decorator는 하나의 테스트 함수에 대해 다양한 입력값을 제공하고 이를 반복 실행하여 코드의 여러 경우를 테스트할 수 있도록 한다. 이를 통해 동일한 테스트 로직을 여러 번 작성할 필요 없이 다양한 입력값에 대한 테스트를 쉽게 수행할 수 있다.

### 사용 사례: 여러 입력값에 대한 함수 테스트

예를 들어, 특정 함수가 여러 입력값에 대해 올바른 결과를 반환하는지 확인하고 싶다고 가정해 본다. 파라미터화 decorator를 사용하면 이 작업을 쉽게 수행할 수 있다.

```python
# app.py
def add(a, b):
    return a + b
```

이제 `add` 함수에 대해 다양한 입력값을 테스트하는 코드를 작성해 본다.

```python
# test_app.py
import pytest
from app import add

@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (2, 3, 5),
    (10, 20, 30),
    (0, 0, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

여기서 `@pytest.mark.parametrize` decorator는 `a`, `b`, `expected`라는 세 개의 매개변수를 갖는 테스트를 정의하고, 각 매개변수 조합에 대해 테스트 함수를 반복 실행한다.

## 공용 Fixture를 위한 `conftest.py` 사용

여러 테스트 파일이 있고 이들 간에 Fixture를 공유하려는 경우, `conftest.py` 파일에 Fixture를 정의할 수 있다. pytest는 이 파일에 정의된 Fixture를 자동으로 발견하고 사용한다.

### `conftest.py` 예제

여러 테스트 파일에서 `test_data` Fixture를 사용하려고 한다.

### `conftest.py`:

```python
import pytest

@pytest.fixture
def test_data():
    return {"a": 10, "b": 2, "c": 0}
```

### 테스트 파일 (`test_app1.py` 및 `test_app2.py`):

이 두 테스트 파일은 이제 `test_data` Fixture를 다시 정의하지 않고 사용할 수 있다.

### `test_app1.py`:

```python
from app import divide

def test_divide(test_data):
    assert divide(test_data["a"], test_data["b"]) == 5
    with pytest.raises(ValueError):
        divide(test_data["a"], test_data["c"])
```

### `test_app2.py`:

```python
from app import add

def test_add(test_data):
    assert add(test_data["a"], test_data["b"]) == 12
```

## 특정 테스트 실행

pytest는 다양한 명령줄 옵션을 사용하여 특정 테스트나 테스트 파일을 실행할 수 있다. 여기 몇 가지 유용한 명령이 있다:

- 특정 테스트 파일 실행:
  ```bash
  pytest test_app1.py
  ```

- 특정 테스트 함수 실행:
  ```bash
  pytest -k test_divide
  ```

- 자세한 출력 보기:
  ```bash
  pytest -v
  ```

- 첫 번째 실패 후 중지:
  ```bash
  pytest -x
  ```

## Markers

pytest marker는 특정 테스트나 테스트 그룹에 태그를 추가하여 조건부 실행, 스킵, 또는 특정 설정을 적용할 수 있도록 한다.

### 사용 사례: 특정 조건에서만 테스트 실행

예를 들어, 특정 테스트를 개발 환경에서만 실행하고 싶다면 marker를 사용할 수 있다.

```python
# test_app.py
import pytest

@pytest.mark.dev
def test_development_feature():
    assert my_dev_function() == "expected_result"
```

이제 이 테스트를 실행할 때 `-m` 옵션을 사용하여 특정 marker가 달린 테스트만 실행할 수 있다.

```sh
pytest -m dev
```

## Exception Handling Tests

특정 함수나 코드가 예외를 올바르게 처리하는지 테스트하는 것이 중요하다. pytest의 `raises` 를 사용하면 쉽게 예외 처리 테스트를 작성할 수 있다.

### 사용 사례: 예외 발생 여부 테스트

```python
# app.py
def divide(x, y):
    if y == 0:
        raise ValueError("Cannot divide by zero")
    return x / y
```

```python
# test_app.py
import pytest
from app import divide

def test_divide_by_zero():
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(1, 0)
```

## 플러그인(Plugins)

pytest는 플러그인 아키텍처를 사용하여 기능을 확장할 수 있다. 수많은 커뮤니티 제작 플러그인이 있으며, 이를 통해 보고서 작성, 코드 커버리지 측정 등 다양한 기능을 추가할 수 있다.

### 사용 사례: 코드 커버리지 측정

`pytest-cov` 플러그인을 사용하면 코드 커버리지를 쉽게 측정할 수 있다.

```sh
pip install pytest-cov
```

```sh
pytest --cov=my_project
```

이 명령은 `my_project` 폴더의 코드 커버리지를 측정하고 결과를 출력한다.

## 훅(Hooks)

pytest 훅은 테스트 실행 전후에 특정 작업을 수행할 수 있는 방법을 제공한다. 예를 들어, 테스트 시작 전에 설정을 하거나 테스트가 끝난 후 결과를 정리할 수 있다.

### 사용 사례: 테스트 전후 설정

`conftest.py` 파일에 훅을 정의할 수 있다.

```python
# conftest.py
def pytest_configure(config):
    print("테스트가 시작됩니다.")

def pytest_unconfigure(config):
    print("테스트가 종료되었습니다.")
```

## Behavior Driven Development, BDD

pytest는 BDD 스타일의 테스트를 작성할 수 있는 `pytest-bdd` 플러그인을 지원한다. 이를 통해 자연어로 작성된 시나리오를 테스트 코드로 변환할 수 있다.

### 사용 사례: BDD 스타일 테스트

```sh
pip install pytest-bdd
```

```python
# features/test_login.feature
Feature: User login

  Scenario: Successful login
    Given I am on the login page
    When I enter valid credentials
    Then I should be redirected to the dashboard
```

```python
# test_login.py
from pytest_bdd import scenarios, given, when, then

scenarios('features/test_login.feature')

@given('I am on the login page')
def on_login_page():
    # 페이지 로드 로직

@when('I enter valid credentials')
def enter_credentials():
    # 로그인 로직

@then('I should be redirected to the dashboard')
def redirected_to_dashboard():
    # 대시보드 확인 로직
```
