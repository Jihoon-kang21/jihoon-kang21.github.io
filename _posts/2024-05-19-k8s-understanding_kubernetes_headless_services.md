---
layout: post
title: k8s - Headless Service 이해하기
subtitle: StatefulSet와 함께 또는 없이 Headless Service를 효과적으로 사용하는 방법
categories: Tech
tags: [k8s]
---

## Kubernetes에서의 Service란?

Kubernetes에서 **Service**는 논리적인 Pod 집합과 이들을 접근하는 정책을 정의한다. Service는 특정 Pod 집합에서 실행 중인 애플리케이션을 외부에 노출시키는 역할을 한다.

## Headless Service란?

**Headless Service**는 클러스터 IP가 없는 Service 유형이다. 이는 클라이언트가 접근할 수 있는 단일 IP 주소를 가지지 않음을 의미한다. 대신, 각 개별 Pod의 IP 주소를 직접 반환한다.

## 언제 Headless Service를 사용하는가?

Headless Service를 사용해야 할 때:
- 단일 Service IP를 통해 라우팅하는 대신 Pod에 직접 접근하고 싶을 때.
- 특정 Stateful application(예: 데이터베이스)에서 Pod의 IP를 사용해야 할 때.

## Headless Service를 어떻게 생성하는가?

Headless Service를 생성하려면 Service 정의에서 `clusterIP` 필드를 `None`으로 설정한다. 이는 Kubernetes에게 이 Service에 클러스터 IP를 할당하지 않도록 지시한다.

## Headless Service 예제

다음은 Headless Service를 정의하는 예제 YAML 파일이다:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-service
spec:
  clusterIP: None
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
```

- **apiVersion**: 사용하는 Kubernetes API 버전.
- **kind**: 리소스 유형(여기서는 `Service`).
- **metadata**: Service에 대한 메타데이터, 예를 들어 이름.
- **spec**: Service의 스펙.
  - **clusterIP: None**: 이 설정이 Service를 Headless로 만든다.
  - **selector**: 이 Service가 관리하는 Pod를 식별하는 라벨.
  - **ports**: Service가 노출되는 포트.

## Headless Service는 어떻게 작동하는가?

Headless Service를 생성하면 Kubernetes는 클러스터 IP를 할당하지 않는다. 대신 Service 이름에 대한 DNS 레코드를 생성하며, 이 DNS 이름을 쿼리할 때 개별 Pod의 IP 주소 목록을 반환한다.

## 상세 예제: Headless Service 쿼리하기

`nslookup` 같은 도구를 사용하여 Headless Service를 쿼리하면 다음과 같은 결과를 볼 수 있다:

```bash
nslookup web-app-service.default.svc.cluster.local
```

출력:

```
Name:      web-app-service.default.svc.cluster.local
Addresses: 10.244.0.3
           10.244.0.4
           10.244.0.5
```

여기서 `10.244.0.3`, `10.244.0.4`, `10.244.0.5`는 각 Pod의 IP 주소이다.

## StatefulSet 없이 Headless Service를 사용하는 경우

Headless Service는 주로 StatefulSet과 같이 사용한다. 하지만 반드시 그렇지는 않다.

### 1. **모니터링 또는 디버깅을 위한 직접 Pod 접근:**

**사용 사례:** 모니터링, 디버깅, 또는 맞춤형 라우팅을 위해 개별 Pod에 직접 접근해야 할 때.

**예제 시나리오:** 웹 Service를 실행하는 여러 Pod가 있고, 각 Pod의 성능을 직접 모니터링하고 싶을 때.

**예제:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-headless
spec:
  clusterIP: None
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

### 2. **커스텀 로드 밸런싱:**

**사용 사례:** Kubernetes의 기본 Service 로드 밸런싱 외에 커스텀 로드 밸런싱을 구현하고자 할 때.

**예제 시나리오:** 백엔드 Pod로 트래픽을 맞춤형 로직에 따라 분산시키는 로드 밸런서 Service가 있을 때.

**예제:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: custom-load-balancer-headless
spec:
  clusterIP: None
  selector:
    app: custom-backend
  ports:
  - port: 8080
    targetPort: 8080
```

### 3. **Service Discovery:**

**사용 사례:** MicroService가 각 Pod의 실제 IP 주소로 통신해야 할 때.

**예제 시나리오:** 다른 Service의 IP 주소를 발견하고 직접 통신해야 하는 MicroService 아키텍처.

**예제:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-discovery-headless
spec:
  clusterIP: None
  selector:
    app: my-service
  ports:
  - port: 5000
    targetPort: 5000
```

## StatefulSet과 Headless Service

**StatefulSet**은 Pod 집합의 배포 및 확장을 관리하는 Kubernetes 리소스이다. 일반적인 Deployment와 달리 StatefulSet은 각 Pod에 고유하고 안정적인 네트워크 ID를 제공한다.

### StatefulSet을 사용할 때:

- Pod에 안정적이고 고유한 네트워크 식별자가 필요할 때.
- Pod의 배포 및 확장을 순서대로, 예측 가능하게 해야 할 때.

예를 들면, Pod 마다 다른 PV가 연결되어있거나, 권한이 다를때.

## 예제 시나리오: StatefulSet이 있는 데이터베이스

각 인스턴스가 직접 다른 인스턴스와 통신해야 하는 데이터베이스(예: Cassandra 또는 MongoDB)가 있다고 가정해보면.

1. **StatefulSet을 위한 Headless Service 생성:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 27017
    name: db
```

2. **StatefulSet 정의:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: "db-service"
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: mongo
        ports:
        - containerPort: 27017
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

- **StatefulSet**은 안정적인 이름과 스토리지를 가진 Pod를 생성한다.
- 각 Pod는 `db-0`, `db-1`, `db-2` 등의 이름을 가집니다.
- Pod는 안정적인 DNS 이름을 사용하여 서로 통신할 수 있다.

## 상세 예제: StatefulSet DNS 이름

위 예제에서 DNS 이름을 쿼리하면 다음과 같이 나타난다:

- `db-0.db-service.default.svc.cluster.local`
- `db-1.db-service.default.svc.cluster.local`
- `db-2.db-service.default.svc.cluster.local`

각 Pod는 고유한 네트워크 ID를 가지며 직접 접근할 수 있다.

## 요약

**StatefulSet 없이 Headless Service를 사용할 때:**
- 모니터링, 디버깅, 또는 맞춤형 라우팅을 위해 개별 Pod에 직접 접근해야 할 때.
- 맞춤형 로드 밸런싱을 구현하고자 할 때.
- 마이크로Service가 서로의 IP 주소를 발견하고 직접 통신해야 할 때.

**StatefulSet과 함께 Headless Service를 사용할 때:**
- 안정적인 네트워크 ID와 안정적인 스토리지가 필요한 상태 저장 애플리케이션.
- 각 Pod에 안정적이고 예측 가능한 DNS 이름이 필요한 애플리케이션.

Headless Service를 StatefulSet과 함께 또는 없이 사용하여 안정적인 네트워크 ID, Pod 간 직접 통신, 특정 라우팅 및 모니터링 요구 사항을 관리할 수 있다.