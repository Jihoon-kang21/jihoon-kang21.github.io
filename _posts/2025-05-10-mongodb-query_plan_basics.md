---
layout: post
title: MongoDB - Query Plans 기초
subtitle: Query plan 확인 방법과 Index 추가 후 차이 확인
categories: Tech
tags: [mongodb]
---

## 🔍 Query Plan이란?

MongoDB에서 Query Plan은 데이터베이스 엔진이 특정 쿼리를 실행하는 방법을 상세히 설명한다. 이 정보를 얻으려면 `find()` 쿼리에 `explain()` 메서드를 추가하면 된다.

```
db.cookbook.find({ "type": "Dinner" }).explain()
```

이 명령은 쿼리 실행에 대한 다양한 정보를 포함한 결과를 반환한다.

```json
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'cookbook.cookbook',
    indexFilterSet: false,
    parsedQuery: { type: { '$eq': 'Dinner' } },
    queryHash: '3D98D089',
    planCacheKey: '3D98D089',
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    winningPlan: {
      stage: 'COLLSCAN',
      filter: { type: { '$eq': 'Dinner' } },
      direction: 'forward'
    },
    rejectedPlans: []
  },
  command: { find: 'cookbook', filter: { type: 'Dinner' }, '$db': 'cookbook' },
  serverInfo: {
    host: 'myserver',
    port: 27017,
    version: '6.0.6',
    gitVersion: '26b4851a412cc8b9b4a18cdb6cd0f9f642e06aa7'
  },
  serverParameters: {
    internalQueryFacetBufferSizeBytes: 104857600,
    internalQueryFacetMaxOutputDocSizeBytes: 104857600,
    internalLookupStageIntermediateDocumentMaxSizeBytes: 104857600,
    internalDocumentSourceGroupMaxMemoryBytes: 104857600,
    internalQueryMaxBlockingSortMemoryUsageBytes: 104857600,
    internalQueryProhibitBlockingMergeOnMongoS: 0,
    internalQueryMaxAddToSetBytes: 104857600,
    internalDocumentSourceSetWindowFieldsMaxMemoryBytes: 104857600
  },
  ok: 1
}
```

우리가 주목해야 할 부분은 `winningPlan`이다. 이 부분은 데이터베이스가 선택한 가장 효율적인 Query Plan을 보여준다.

```
db.cookbook.find({ "type": "Dinner" }).explain().queryPlanner.winningPlan
```

이 명령은 다음과 같은 결과를 반환할 수 있다:

```json
{
  "stage": "COLLSCAN",
  "filter": { "type": { "$eq": "Dinner" } },
  "direction": "forward"
}
```

여기서 `COLLSCAN`은 컬렉션 스캔을 의미하며, 이는 데이터베이스가 컬렉션의 모든 문서를 검사하여 조건에 맞는 문서를 찾았다는 것을 나타낸다. 이 방법은 인덱스가 없을 때 사용되며, 대규모 데이터셋에서는 비효율적일 수 있다.

---

## ⚙️ 인덱스를 사용한 최적화

쿼리 성능을 향상시키기 위해 인덱스를 사용할 수 있다. 예를 들어, `type` 필드에 인덱스를 추가하려면 다음 명령을 사용한다:

```
db.cookbook.createIndex({ "type": 1 }, { "name": "ix_type" })
```

이제 쿼리와 `explain()`을 다시 실행하면, 결과는 다음과 같이 변경된다.

```json
{
  "stage": "FETCH",
  "inputStage": {
    "stage": "IXSCAN",
    "keyPattern": { "type": 1 },
    "indexName": "ix_type",
    "isMultiKey": false,
    "multiKeyPaths": { "type": [] },
    "isUnique": false,
    "isSparse": false,
    "isPartial": false,
    "indexVersion": 2,
    "direction": "forward",
    "indexBounds": { "type": [ "[\"Dinner\", \"Dinner\"]" ] }
  }
}
```

여기서 `IXSCAN`은 인덱스 스캔을 의미하며, MongoDB가 `type` 필드에 대한 인덱스를 사용하여 쿼리를 실행했음을 나타낸다. 이는 전체 컬렉션을 스캔하는 것보다 훨씬 효율적이다.


## 📊 `executionStats` 모드로 실행 통계 확인하기

`executionStats` 모드를 사용하면 실제로 쿼리를 실행하고 각 단계의 성능 통계를 확인할 수 있다.

```
db.collection.find({ 조건 }).explain("executionStats")
```

이 명령은 다음과 같은 정보를 제공한다:

- `executionTimeMillis`: 전체 쿼리 실행 시간 (밀리초)
- `totalKeysExamined`: 검사한 인덱스 키의 수
- `totalDocsExamined`: 검사한 문서의 수

이러한 통계를 통해 쿼리의 효율성을 평가하고, 인덱스 사용 여부나 불필요한 문서 스캔을 확인할 수 있다.

---

## 🔄 `rejectedPlans`를 통해 대체 Query Plan 분석하기

MongoDB는 쿼리를 실행하기 전에 여러 Query Plan을 평가하고 최적의 Plan을 선택한다. 선택되지 않은 다른 Plan들은 `queryPlanner.rejectedPlans` 배열에 저장된다.

이 정보를 통해 MongoDB가 어떤 Query Plan을 고려했는지, 그리고 왜 특정 Plan이 선택되지 않았는지를 이해할 수 있다.

---

## 🧱 Query Plan 단계 이해하기

`explain()` 출력에는 여러 단계가 포함될 수 있으며, 각 단계는 특정 작업을 수행한다. 예를 들어:

- `IXSCAN`: 인덱스를 사용하여 문서를 검색
- `FETCH`: 인덱스로 찾은 문서에서 실제 데이터를 가져옴
- `SORT`: 정렬 작업 수행
- `PROJECTION`: 필요한 필드만 선택

이러한 단계를 이해하면 쿼리의 실행 흐름을 파악하고, 성능 병목 지점을 식별하는 데 도움이 된다.

---

## 🛠️ `explain()`을 활용한 쿼리 튜닝 예시

다음은 `explain()`을 사용하여 쿼리 성능을 개선하는 예시이다:

1. 기존 Query Plan 확인:

```
db.collection.find({ 필터 조건 }).explain("executionStats")
```

2. 출력된 Query Plan에서 `COLLSCAN`이나 `SORT` 단계가 있는지 확인한다.

3. 필터 조건과 정렬 필드를 포함하는 복합 인덱스를 생성한다:

```
db.collection.createIndex({ 필터 필드: 1, 정렬 필드: 1 })
```

4. 다시 `explain("executionStats")`를 실행하여 개선된 Query Plan과 성능 통계를 확인한다.

이러한 과정을 통해 불필요한 문서 스캔과 정렬 작업을 줄이고, 쿼리 성능을 향상시킬 수 있다.