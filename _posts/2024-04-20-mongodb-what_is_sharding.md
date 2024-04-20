---
layout: post
title: MongoDB - Sharding 이란 무엇인가
subtitle: MongoDB 수평 스케일링, 필수 단위 Chunk 이해하기
categories: Tech
tags: [mongodb]
---

## Sharding 이란

MongoDB에서 Sharding은 대규모 데이터를 처리하는 데이터베이스의 성능과 Scalability을 향상시키기 위해 데이터를 여러 machine에 분산시키는 방법이다. 하나의 서버로 감당하기 어려울 정도로 데이터 세트가 크거나, read/write 요청량이 많은 경우에 적용할 수 있다.

### 왜 Sharding을 사용할까

1. **Scalability**: Sharding을 통해 데이터베이스 시스템이 수평적으로 확장될 수 있다. 이는 부하 증가에 대응하기 위해 더 많은 machine를 추가하는 것을 의미하며, 단일 서버를 업그레이드하는 것(Vertical Scaling)보다는 낫다.
2. **Performance**: 데이터를 여러 서버에 분산시키면 단일 서버에 대한 부하를 줄일 수 있으며, 이는 읽기 및 쓰기 성능을 향상시킬 수 있다.
3. **High Availability**: Sharding은 중복을 통해 가용성을 향상시킬 수 있다. 한 Shard가 실패하면 다른 Shard가 대신 작업을 수행할 수 있다.

### MongoDB에서 Sharding이 작동하는 방식

**주요 구성 요소:**

- **Shard**: 각 Shard는 Sharding된 데이터를 가지고 있다. 각 Shard는 Replica Set으로 배포될 수 있다.
- **Mongos**: 쿼리 라우터이다. 클라이언트는 Mongos 인스턴스에 연결하며, Mongos는 쿼리를 적절한 Shard로 라우팅한다.
- **Config Servers**: MongoDB는 클러스터의 메타데이터를 저장하기 위해 Config Server를 사용한다. 이 메타데이터는 Mongos가 쿼리를 올바른 Shard로 라우팅하는 데 도움을 준다.

**Sharding 과정:**

1. **Shard 키 선택**: Shard 키는 데이터를 Chunk로 나누기 위해 선택된 필드 또는 필드이다. Shard 키의 선택은 Shard 간 데이터 분배에 영향을 미치므로 중요하다.
2. **Chunk 분할**: 데이터는 Shard 키를 기반으로 Chunk로 나누어진다. MongoDB는 데이터 분배를 자동으로 관리한다.
3. **Balancing**: MongoDB는 데이터의 균등한 분포를 목표로 Shard 간에 Chunk를 균형 있게 조정한다. 이 자동 균형 조정은 단일 Shard가 병목 현상이 되는 것을 방지하는 데 도움이 된다.

### Sharding의 예

예를 들어, 전 세계 수백만 명의 사용자를 보유한 소셜 미디어 플랫폼에 대한 정보를 저장하는 데이터베이스가 있다고 하자. 운영자는 작업 부하를 분산시키기 위해 "사용자(Users)" Collection을 Sharding하기로 결정한다.

1. **Shard 키 선택**: 사용자 데이터가 사용자 ID를 통해 접근된다고 하면, Shard 키로 `user_id`를 선택할 수 있다. 이 키는 데이터를 고르게 분산시킬 것이다.
2. **Setup**: 운영자는 데이터를 저장할 수 있는 여러 Shard를 설정할 것이다. 각 Shard는 데이터의 일부를 저장한다.
3. **Routing Queries**: 예를 들어 사용자 ID로 사용자를 찾기 위한 쿼리가 만들어질 때, Mongos 라우터는 Shard 키를 기반으로 적절한 Shard로 쿼리를 라우팅할 것이다.

### Sharding 구현 단계

1. **MongoDB 설치**: MongoDB가 설치되어 있고 Sharding된 클러스터를 생성하기 위한 필요한 권한과 자원이 있는지 확인한다.
2. **Shard 구성**: Shard 역할을 할 여러 서버를 설정한다.
3. **Mongos 및 Config Server 구성**: 쿼리 라우터 및 Config Server를 설정한다.
4. **데이터베이스에 대한 Sharding 활성화**: 특정 데이터베이스에서 Sharding을 활성화하기 위해 MongoDB 명령을 사용한다.
5. **Collection Sharding**: 데이터베이스에 대한 Sharding을 활성화한 후, Shard 키를 지정하여 특정 Collection에 대한 Sharding을 활성화할 수 있다.

```bash
# 데이터베이스와 Collection에 Sharding을 활성화하기 위한 예제 명령
# Mongos에 연결
mongo --host mongos-host --port 27017

# 데이터베이스에 Sharding 활성화
sh.enableSharding("database_name")

# Collection Sharding
sh.shardCollection("database_name.collection_name", { "user_id" : 1 } )
```

이 예제는 Sharding된 MongoDB 환경을 설정하는 기본 단계를 설명한다. Sharding을 계획하고 실행할 때는 Shard 키의 선택이 시스템의 성능과 Scalability에 중대한 영향을 미칠 수 있으므로 신중하게 계획하고 실행하는 것이 중요하다. 구성 요소와 과정을 이해하면 MongoDB에서 Sharding을 구현할 때와 방법에 대해 정보에 근거한 결정을 내릴 수 있다.

## MongoDB에서 Chunk란?

Sharding 아키텍처의 핵심 요소인 "Chunk"는 Sharding 클러스터에서 데이터를 관리하고 분산하는 기본 단위이다. Shard 키 값의 연속적인 범위를 나타내며, Sharding 클러스터에서 데이터 분배의 기본 단위이다. 각 Chunk는 단일 Shard에 저장되며, 이를 통해 MongoDB는 데이터셋을 여러 Shard에 효율적으로 분산시킬 수 있다. 주 목적은 대량의 데이터를 더 작고 관리하기 쉬운 부분으로 나누어 사용 가능한 서버에 고르게 분배하는 것이다.

### Chunk의 작동 방식

Chunk를 통한 데이터 관리 과정은 몇 가지 주요 단계를 포함한다:

1. **Shard Key-Based Division**: Collection이 Sharding될 때, MongoDB는 Shard 키를 사용하여 데이터를 Chunk로 나눈다. Shard 키는 필드 또는 필드의 조합으로 선택되며, 데이터를 조직하고 Shard 간에 분배하는 데 사용된다.
   
2. **Size and Limits**: 각 Chunk는 기본적으로 64메가바이트의 크기 제한을 가지며, 이는 필요에 따라 조정될 수 있다. 이 크기 제한은 쿼리 처리의 효율성과 너무 많은 작은 Chunk를 관리하는 오버헤드 사이의 균형을 맞춘다.

3. **Data Distribution**: Chunk는 Shard 부하가 균형을 이루도록 사용 가능한 Shard에 분산되어 배포된다. 데이터가 성장하고 개별 Chunk가 최대 크기에 도달하면, MongoDB는 자동으로 Chunk를 더 작은 Chunk로 분할하고 필요에 따라 Shard 간에 이동하여 부하 균형을 유지다.

4. **Balancing and Splitting**: MongoDB는 백그라운드에서 실행되는 밸런서 프로세스를 사용하여 Chunk의 크기와 분포를 모니터링한다. Chunk가 설정된 크기 제한을 초과하면 밸런서는 그것을 더 작은 Chunk로 분할한다. 또한, 클러스터 전체의 데이터 분배가 불균형을 보이면 밸런서는 Chunk를 이동시켜 부하를 균형있게 조정한다.

### 예시 시나리오: 사용자 데이터베이스 Sharding

전 세계 고객 기반을 가진 전자 상거래 플랫폼을 운영한다고 가정해보자. 이를 효과적으로 관리하기 위해 `customer_id`를 Shard 키로 사용하여 `customers` Collection을 Sharding할 수 있다. 이 시나리오에서 Chunk가 어떻게 작동하는지 살펴보자:

- **초기 Chunk 분배**: 초기에 다음과 같이 Chunk를 분배할 수 있다:
  - Chunk 1: `customer_id` 1-1000
  - Chunk 2: `customer_id` 1001-2000
  - Chunk 3: `customer_id` 2001-3000
- **Shard 할당**: 이 Chunk들은 다른 Shard에 할당되어 데이터 부하를 분산한다.
- **동적 Chunk 관리**: 거래가 발생하고 새로운 고객이 추가됨에 따라 일부 Chunk는 다른 Chunk보다 빠르게 성장할 수 있다. 예를 들어, Chunk 1이 너무 커지면 MongoDB는 자동으로 그것을 더 작은 Chunk로 분할하다(예: Chunk 1a: `customer_id` 1-500, Chunk 1b: `customer_id` 501-1000). 필요한 경우 이 새로운 Chunk를 다른 Shard로 이동하여 부하를 균형있게 조정할 수 있다.

### 결론

Chunk는 MongoDB Sharding 환경에서 데이터 관리에 필수적이다. 대규모 데이터 볼륨 및 높은 트랜잭션 속도를 효과적으로 처리할 수 있는 Sharding 클러스터를 관리하는 데 Chunk의 역할이 중요하다. Chunk 관리 및 그 분포를 효율적으로 최적화함으로써, 관리자는 MongoDB 클러스터가 성능을 개선하고 가용성 및 고장 허용도를 높일 수 있도록 보장할 수 있다.

Chunk를 활용함으로써 MongoDB는 여러 서버에 걸쳐 부하를 분산시키면서 수평적으로 확장할 수 있는 견고한 프레임워크를 제공하다. 이로 인해 대규모, 빠른 성장을 겪는 응용 프로그램을 다루는 기업들에게 선호되는 선택이 된다.