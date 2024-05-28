---
layout: post
title: python - 프로젝트 관리:poetry와 pre-commit 사용해보기
subtitle: dependency 관리 및 코드 품질 유지 방법
categories: Tech
tags: [python]
---

## python 프로젝트를 위한 Poetry와 Pre-commit 초보자 가이드

Poetry는 python 프로젝트의 dependency을 관리하고 패키징을 쉽게 해주는 도구이다. 이를 통해 프로젝트의 dependency 충돌을 방지하고 일관된 개발 환경을 유지할 수 있다. Pre-commit은 코드 커밋 전에 다양한 코드 품질 검사를 자동으로 수행하는 프레임워크이다. 이를 통해 코드 스타일, 린팅, 보안 검사 등을 자동화하여 코드 품질을 보장한다. 두 도구를 함께 사용하면 프로젝트 관리와 코드 품질 유지가 훨씬 수월해진다.

### 목차
1. [Poetry 설치 방법](#poetry-설치-방법)
2. [Poetry로 프로젝트 생성하기](#poetry로-프로젝트-생성하기)
3. [Poetry 명령어로 dependency 파일 설정하기](#poetry-명령어로-dependency-파일-설정하기)
4. [Pre-commit 설치 방법](#pre-commit-설치-방법)
5. [유용한 Pre-commit Hook](#유용한-pre-commit-Hook)
6. [각 Hook의 예시](#각-Hook의-예시)

---

## Poetry 설치 방법

Poetry는 python 프로젝트 dependency 관리와 패키징을 도와주는 도구이다. 다음은 Poetry를 설치하는 방법이다:

1. **공식 설치 프로그램 사용**:
   터미널을 열고 다음 명령어를 실행한다:
   ```bash
   curl -sSL https://install.python-poetry.org | python3 -
   ```

2. **설치 확인**:
   Poetry가 제대로 설치되었는지 확인하려면 다음 명령어를 실행한다:
   ```bash
   poetry --version
   ```

## Poetry로 프로젝트 생성하기

Poetry가 설치되었으면, 새로운 프로젝트를 생성하는 것은 간단한다.

1. **새 프로젝트 생성**:
   터미널에서 프로젝트를 생성할 디렉토리로 이동한 후 다음 명령어를 실행한다:
   ```bash
   poetry new my_project
   ```
   그러면 다음과 같은 구조로 `my_project`라는 새 디렉토리가 생성된다:
   ```
   my_project
   ├── pyproject.toml
   ├── README.rst
   └── my_project
       └── __init__.py
   ```

## Poetry 명령어로 dependency 설정하기

Poetry는 프로젝트의 dependency을 관리하기 쉽게 만들어준다.

1. **dependency 추가**:
   새로운 dependency을 추가하려면 `add` 명령어를 사용한다:
   ```bash
   poetry add requests
   ```
   이 명령어는 다음을 수행합니다:

   - requests 패키지를 pyproject.toml 파일의 [tool.poetry.dependencies] 섹션에 추가한다.
   - 해당 패키지를 설치하고 poetry.lock 파일을 업데이트한다.

2. **특정 버전의 패키지 추가**:
   특정 버전의 패키지를 추가하려면 다음과 같이 실행한다:
   ```bash
   poetry add requests@2.25.1
   ```
   이 명령어는 requests 패키지의 2.25.1 버전을 설치한다.

3. **모든 dependency 설치**:
   `pyproject.toml`에 나열된 모든 dependency을 설치하려면 다음 명령어를 실행한다:
   ```bash
   poetry install
   ```
   poetry를 쓰지 않고 requirements.txt 를 사용했다면 `pip install -r requirements.txt`와 같은 역할이다.

## Pre-commit 설치 방법

Pre-commit은 다중 언어 pre-commit Hook를 관리하고 유지하는 프레임워크이다.

1. **Pre-commit 설치**:
   먼저 pre-commit 패키지를 설치해야 한다. Poetry를 사용 중이라면 개발용 dependency으로 추가할 수 있다:
   ```bash
   poetry add --dev pre-commit
   ```
   이렇게하면 pre-commit은 dev그룹에 속하게 된다. 배포할때는 pre-commit을 포함하지 않고 개발활동을 할때에만 이 그룹을 포함하여 설치, pre-commit을 사용할 수 있게 한다.

2. **구성 파일 생성**:
   프로젝트의 루트 디렉토리에 `.pre-commit-config.yaml` 파일을 생성하고 다음 내용을 추가한다:
   ```yaml
   repos:
   - repo: https://github.com/pre-commit/pre-commit-hooks
     rev: v3.2.0
     hooks:
       - id: check-merge-conflict
       - id: check-added-large-files
       - id: detect-private-key
   - repo: https://github.com/python-poetry/poetry
     rev: '1.6.1'
     hooks:
       - id: poetry-check
   - repo: https://github.com/floatingpurr/sync_with_poetry
     rev: "1.1.0"
     hooks:
       - id: sync_with_poetry
         args: []
   - repo: https://github.com/PyCQA/autoflake
     rev: v2.2.1
     hooks:
       - id: autoflake
         args: [--remove-all-unused-imports, --in-place]
   - repo: https://github.com/pycqa/isort
     rev: 5.13.2
     hooks:
       - id: isort
         args: ["--profile", "black", "--filter-files"]
   - repo: https://github.com/psf/black-pre-commit-mirror
     rev: 23.12.1
     hooks:
       - id: black
         exclude: mongodbio.py
   - repo: https://github.com/astral-sh/ruff-pre-commit
     rev: v0.3.2
     hooks:
       - id: ruff
         args: [ --fix ]
   ```

3. **Hook 설치**:
   다음 명령어를 실행하여 pre-commit Hook를 설치한다:
   ```bash
   pre-commit install
   ```
   `.pre-commit-config.yaml` 에 있는 hook을 모두 설치하게 된다.

## 유용한 Pre-commit hook

Pre-commit Hook는 커밋하기 전에 실행되는 스크립트이다. 코드 품질을 보장하고 일반적인 실수를 방지하는 데 도움을 준다.

1. **check-merge-conflict**: 해결되지 않은 병합 충돌을 감지한다.
2. **check-added-large-files**: 큰 파일이 추가될 때 경고를 표시한다.
3. **detect-private-key**: 개인 키 커밋을 방지한다.
4. **poetry-check**: `pyproject.toml` 파일의 유효성을 검사한다.
5. **sync_with_poetry**: `poetry.lock` 파일이 `pyproject.toml`과 동기화되었는지 확인한다.
6. **autoflake**: 사용되지 않는 import와 변수를 제거한다.
7. **isort**: import를 지정된 프로필에 따라 정렬한다.
8. **black**: 코드를 Black 스타일로 포맷한다.
9. **ruff**: Ruff 린터를 실행하고 수정한다.

## 각 Hook의 예시

1. **check-merge-conflict**:
   - **목적**: 해결되지 않은 병합 충돌을 감지한다.
   - **예시**: 파일에 `<<<<<<`, `======`, `>>>>>>`가 포함되어 있으면 커밋이 방지된다.

2. **check-added-large-files**:
   - **목적**: 큰 파일이 추가될 때 경고를 표시한다.
   - **예시**: 500 KB보다 큰 파일을 추가하면 경고가 표시된다.

3. **detect-private-key**:
   - **목적**: 개인 키 커밋을 방지한다.
   - **예시**: `-----BEGIN RSA PRIVATE KEY-----`가 포함된 파일을 추가하면 커밋이 차단된다.

4. **poetry-check**:
   - **목적**: `pyproject.toml` 파일의 유효성을 검사한다.
   - **예시**: `pyproject.toml`에 문법 오류가 있으면 이 Hook가 이를 감지한다.
   - **실행 방법**: 이 Hook는 각 커밋 전에 자동으로 실행된다.

5. **sync_with_poetry**:
   - **목적**: `poetry.lock` 파일이 `pyproject.toml`과 동기화되었는지 확인한다.
   - **예시**: dependency을 추가했지만 잠금 파일을 업데이트하지 않은 경우, 이 Hook가 `poetry lock`을 실행하도록 알린다.
   - **실행 방법**: 이 Hook는 각 커밋 전에 자동으로 실행된다.

6. **autoflake**:
   - **목적**: 사용되지 않는 import와 변수를 제거한다.
   - **예시**: 코드에 사용되지 않는 import가 있으면 이 Hook가 이를 제거한다.
   - **실행 방법**: 이 Hook는 각 커밋 전에 자동으로 실행된다.

7. **isort**:
   - **목적**: import를 정렬한다.
   - **예시**: import가 정렬되지 않은 경우, 이 Hook가 지정된 프로필에 따라 이를 정렬한다.
   - **실행 방법**: 이 Hook는 각 커밋 전에 자동으로 실행된다.

8. **black**:
   - **목적**: 코드를 Black 스타일로 포맷한다.
   - **예시**: 코드가 Black 스타일을 따르지 않는 경우, 이 Hook가 이를 재포맷한다.
   - **실행 방법**: 이 Hook는 각 커밋 전에 자동으로 실행된다.

9. **ruff**:
   - **목적**: Ruff 린터를 실행한다.
   - **예시**: 코드에 린팅 문제가 있으면 이 Hook가 이를 수정한다.
   - **실행 방법**: 이 Hook는 각 커밋 전에 자동으로 실행된다.

## 결론

Poetry와 pre-commit을 함께 사용하면 python 프로젝트의 dependency 관리와 코드 품질을 크게 향상시킬 수 있다. 이 가이드를 따라 이러한 도구를 설정하고 프로젝트를 잘 유지하며 일반적인 오류를 방지할 수 있다.
