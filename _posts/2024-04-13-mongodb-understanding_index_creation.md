---
layout: post
title: MongoDB - index 생성, 종료 및 관리 이해하기
subtitle: Foreground, background build, Lock 등에 대해
categories: Tech
tags: [mongodb]
---

### MongoDB index 생성(build) 방법

MongoDB는 데이터 검색을 최적화하기 위해 index 생성을 지원한다. index는 대량의 데이터를 가진 데이터베이스에서 중요하며, 생성 방식에 따라 성능에 큰 영향을 미칠 수 있다. index는 Foreground 또는 Background 방식으로 생성할 수 있다.

#### 4.2 보다 이전 버전에서 index 생성시 두가지 옵션

4.2 이전 버전에서는 index를 생성할때 두가지 옵션 중 선택할 수 있다.

- **Foreground 빌드**: 이 방식은 더 빠르며 효율적인 index 구조를 생성한다. 그러나, index를 생성하는 동안 데이터베이스를 잠궈서 읽기 및 쓰기 작업을 할 수 없도록 Lock이 걸린다. 데이터베이스 사용을 종료할 수 있는 환경에서 적합하다.
- **Background 빌드**: 이 빌드는 느리고 덜 효율적인 index를 생성하지만, index를 생성하는 동시에 읽기 및 쓰기 작업을 계속 허용한다. 서비스 가용성을 유지해야 하는 생산 환경에서 선호된다.

MongoDB 4.2 버전부터는 데이터베이스 작업에 미치는 영향을 줄이기 위해 index 빌드 프로세스를 개선했다. 그리고 Foreground, Background 옵션을 없앴다. 4.2 버전부터는 index 생성시 다음 특징을 가진다.

- **Exclusive Locks**: index 생성 과정 중에 MongoDB는 데이터 무결성과 일관성을 보장하기 위해 중요한 시점에 exclusive locks를 적용한다. exclusive locks의 작동 방식은 다음과 같다:
   - Lock 시점: index 생성 과정의 시작과 끝에 collection에 대한 독점 Lock이 적용된다. 이는 MongoDB가 중단되지 않고 메타데이터 변경을 안전하게 처리해야 하는 중요한 순간이다.
   - Lock의 목적: 이 Lock의 주요 목적은 index 생성 과정의 필수적인 부분인 meta data 변경을 안전하게 처리하는 것이다. meta data에는 index 자체에 대한 정보, 예를 들어 구조, 관련 데이터 포인트 및 쿼리 시 index 사용 방법을 안내하는 기타 필수 세부 정보가 포함된다.
   - 작업에 미치는 영향: Lock이 설정되어 있는 동안에는 잠긴 collection에 대한 다른 읽기 또는 쓰기 작업을 수행할 수 없다. index 생성을 방해할 수 있는 동시 작업을 방지하여 데이터 손상이나 일관성 없는 상태를 막는다.

- **Yielding Behavior**: 빌드 프로세스 중 MongoDB는 가능한 한 Lock을 하지 않고 다른 작업을 허용한다. 이는 Background 빌드의 행동과 유사하지만 Foreground 빌드의 효율성을 갖는다.
   - 운영 유연성: index 생성 중에 MongoDB는 가능한 경우 Lock을 푼다. 즉, 데이터베이스는 collection에 대한 Lock을 일시적으로 해제하여 다른 작업(읽기 및 쓰기)이 진행될 수 있도록 한다. 이 양보는 index 빌드의 시작과 끝을 제외하고 전체 빌드 과정 중 주기적으로 발생한다.
   - Background 빌드와 유사: 이 접근 방식은 전통적인 Background index 빌드를 연상시키며, 이러한 과정은 느리고 덜 효율적이었지만 데이터베이스가 작동 상태를 유지할 수 있었다. 동시 작업을 허용함으로써 MongoDB는 시스템이 사용자에게 반응하고 접근 가능하도록 보장한다.
   - Foreground 빌드의 효율성: 주기적인 양보에도 불구하고, MongoDB의 현대적인 index 빌드 프로세스는 전통적인 Foreground 빌드만큼 효율적으로 설계되었다. 효율성은 빌드 중 데이터에 접근하고 구성하는 방법을 관리하는 개선된 알고리즘 및 기술에서 비롯된다. 이를 통해 MongoDB는 양보를 하면서도 이전보다 더 빠르게 작업을 처리할 수 있다.

## index 생성 중 강제 종료
MongoDB 4.2 이후 버전
MongoDB 4.2 버전부터는 잠재적인 중단과 데이터 무결성 문제를 최소화하기 위해 진행 중인 인덱스 빌드를 처리하는 프로세스가 개선되었다. 이 버전들에서는 인덱스 빌드를 더 안전하게 중단할 수 있는 기능을 도입했다:

복제 세트 및 샤디드 클러스터: 이러한 배포에서는 killOp 명령어를 사용하여 인덱스 빌드를 종료하는 것이 권장되지 않다. 이는 killOp가 클러스터를 회복하기 어려운 상태로 남겨둘 수 있기 때문이며, 특히 인덱스 빌드가 여러 노드에 걸쳐 조정된 작업의 일부인 경우에 그렇다.

대신, MongoDB는 배포의 데이터 일관성을 위험에 빠뜨리지 않고 작업을 안전하게 중단할 수 있도록 설계된 데이터베이스의 관리 기능을 통해 인덱스 빌드 중단을 관리할 것을 권장한다.

관리 명령어 사용: MongoDB는 분산 환경의 복잡성을 처리할 수 있도록 설계된 db.adminCommand({abortIndexBuild: <indexName>})와 같은 명령어를 제공하여 안전하게 인덱스 빌드를 중단힌다. 이 명령어는 모든 노드에서 조정된 방식으로 인덱스 빌드를 중단하도록 보장하여, 어떤 노드도 다른 노드와 다른 상태를 가지지 않도록 한다.

## 4.2 이전 버전에서 Index 생성 방법 예시

### 예제 1: Foreground index 생성
`users`라는 MongoDB 컬렉션에 'email' 필드에 index를 생성하여 이메일 필터링 쿼리의 성능을 향상시키고자 한다면, 다음과 같이 Foreground index를 생성할 수 있다:
```javascript
db.users.createIndex({ email: 1 }, { background: false });
```
이 작업은 index가 완료될 때까지 `users` 데이터베이스의 모든 다른 작업을 차단한다.

### 예제 2: Background index 생성
데이터베이스가 index 생성 중에도 사용 가능해야 한다면 Background 빌드를 선택할 수 있다:
```javascript
db.users.createIndex({ email: 1 }, { background: true });
```
이렇게 하면 index가 구축되는 동안에도 다른 데이터베이스 작업을 계속할 수 있다.
