---
layout: post
title: Python - PYTHONPATH 등록하기
subtitle: 다른 프로젝트 폴더에 있는 모듈 불러오기
categories: Tech
tags: [python]
---

## 배경

git으로 관리되고 있는 프로젝트가 있을 때, 프로젝트에 아직 추가하고 싶지는 않지만 프로젝트에 포함된 코드를 활용하여 스크립트를 따로 쓰고 싶을때가 있다.
그러면 스크립트에서는 해당 프로젝트의 모듈들을 import 해야하는데, PYTHONPATH에 프로젝트의 경로를 추가해놓으면 import 할때 전체 경로를 쓰지 않아도 된다.
예를들어,
- 프로젝트 경로: /project/jihoon
라고하면, 아래 경로를 등록하면,
- PYTHONPATH: /project/jihoon
/project/jihoon/module.py 모듈을 아래처럼 불러올 수 있다.
```python
import module
```

## 1. `PYTHONPATH` 추가 방법

`PYTHONPATH`는 Python 인터프리터가 표준 위치 이외에 모듈을 찾을 디렉토리를 결정하는 데 사용하는 환경 변수이다. 명령 줄에서 실행 파일을 찾는 데 사용되는 `PATH` 환경 변수와 유사하다.

### 운영 체제별 PYTHONPATH 설정 방법:

- **Windows에서:**
  - **임시 설정**: 명령 프롬프트를 열고 다음을 입력한다:
    ```cmd
    set PYTHONPATH=C:\path\to\your\project
    ```
  - **영구 설정**: '계정의 환경 변수 편집'을 검색하여 '환경 변수'를 선택한 다음 '사용자 변수' 아래에서 '새로 만들기...'를 클릭하고 이름으로 `PYTHONPATH`를 입력하고 값으로 프로젝트 디렉토리 경로를 입력하다.

- **macOS 및 Linux에서:**
  - **임시 설정**: 터미널을 열고 다음을 입력하다:
    ```bash
    export PYTHONPATH=/path/to/your/project
    ```
  - **영구 설정**: export 명령을 쉘 구성 파일(예: `.bashrc`, `.bash_profile`, `.zshrc`)에 추가하다:
    ```bash
    echo 'export PYTHONPATH=/path/to/your/project' >> ~/.bashrc
    ```

## 2. 스크립트 내에서 `os.environ['PYTHONPATH']`를 사용하면 안된다

만약, python script 에서 다음과 os 를 활용하여 PYTHONPATH를 등록하면 제대로 모듈을 불러오지 않는다.

```python
import os
os.environ['PYTHONPATH'] = '/path/to/your/project'
```

스크립트 내에서 `os.environ['PYTHONPATH']`를 설정하면 환경 변수는 수정되지만, 이 변경 사항은 현재 프로세스와 수정 후 생성된 하위 프로세스에만 영향을 미친다. 현재 실행 중인 Python 인터프리터의 `sys.path`에는 영향을 주지 않는다. 따라서 동일 스크립트에서 수행하는 모든 모듈은 이 업데이트된 경로에서 검색하지 않는다.

## 3. `sys.path.append`는 사용할 수 있다.

아래처럼 sys.path.append를 사용하면 동작한다.

```python
import sys
sys.path.append('/path/to/your/project')

import module
...
```

## 4. `sys`와 `os`의 차이점

`sys`와 `os` 모듈은 모두 Python에 내장되어 있으며 다른 기능을 제공한다:

- **`sys` 모듈**: 이 모듈은 Python 런타임 환경의 다양한 부분을 조작하는 데 사용되는 함수와 변수를 제공다. 시스템 특정 매개변수와 함수, 예를 들어 다음과 같은 것들을 다룬다:
  - `sys.path`: 디렉터리의 경로들이 기록된 문자열 리스트이다. 이 리스트에 경로를 추가하면 해당 경로에 있는 파이썬 파일을 import 문으로 불러올 수 있다.
  - `sys.argv`: Python 스크립트를 실행할때 같이 전달된 매개변수 목록이다.
  - `sys.exit()`: 스크립트를 종료시킨다.

- **`os` 모듈**: 이 모듈은 운영 체제에 종속적인 기능을 사용하는 방법을 제공한다. 운영 체제와 상호 작용하는 함수를 포함하고 있다, 예를 들어 다음과 같은 것들이 있다:
  - `os.environ`: 운영 체제에 설정되어 있는 모든 환경 변수에 접근이 가능하다. 예를 들어, `os.environ['HOME']`은 홈 디렉토리 경로를 반환하다.
  - `os.chdir()`: 현재 작업 디렉토리를 변경한다.
  - `os.system()`: 셸 명령을 실행하다.

### **차이점을 설명하는 예제들:**
```python
import sys
import os

# sys 예제: Python 검색 경로를 출력하다
print("Python Path:", sys.path)

# os 예제: 현재 작업 디렉토리를 가져옵니다
print("Current Directory:", os.getcwd())

# PYTHONPATH 수정 및 영향 확인
os.environ['PYTHONPATH'] = '/new/path'
os.system('echo $PYTHONPATH')  # Unix 계열 시스템에서 '/new/path'를 출력하다
```

## 결론

환경 변수와 같은 `PYTHONPATH`의 변경 사항은 IDE나 운영 체제에서 직접 설정하는 것이 낫기는 하지만, 여러 서버에서 스크립트를 써야할때는 위 방법이 유용하다.