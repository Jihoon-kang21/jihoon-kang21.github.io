---
layout: post
title: kafka docker로 설치하고 data 전송해보기
subtitle: docker-compose를 이용해 zookeeper, kafka를 설치하고, consumer와 producer를 실행해본다.
categories: tech
tags: [kafka]
---

# intro

- kafka를 설치하고 간단히 테스트하기
- conduktor에서 제공하는 docker compose 파일을 이용하면 간단히 설치하고 실행 가능
- conductor.io 는 kafka desktop client 툴을 제공하는 곳
- 초보 사용자들을 위해 kafka 설치 연습해볼수 있는 repo가 있음 : https://github.com/conduktor/kafka-stack-docker-compose
	- 사용 방법도 repo에 나와있다.

# 설치하기

- git clone 후에 zk-single-kafka-single.yml 파일을 실행하면 Single Zookeeper / Single Kafka 옵션으로 테스트 해볼 수 있다.
	- repo에 가면 Zookeeper 또는 Kafka를 multi로 실행할 수 있는 옵션도 있다.
```bash
git clone https://github.com/conduktor/kafka-stack-docker-compose.git
cd kafka-stack-docker-compose
docker compose -f zk-single-kafka-single.yml up -d
```

- 실행했던 docker를 모두 종료하려면,
```bash
docker compose -f zk-single-kafka-single.yml down
```

* multipass 로 Ubuntu 18.04에서 실행했을때 : 메모리 335메가, consumer 실행하면 460메가까지 사용량이 상승했다

# topic을 생성하고 메시지 전송해보기

- topic을 생성하고 consumer 및 producer 실행할 수 있는 명령어가 kafka container에 있다.
- 해서, 컨테이너에 진입한다.
```bash
docker exec -it kafka1 bash
```

- topic을 생성한다
```
kafka-topics --create --topic my-topic --bootstrap-server kafka:9092 --replication-factor 1 --partitions 1
```

- topic이 잘 생성되었는지 확인한다
```bash
kafka-topics --describe --topic my-topic --bootstrap-server kafka:9092
```

- topic의 data를 구독하는 consumer를 실행한다.
- consumer를 실행하면 topic의 data를 받아오기위해 대기 상태가 된다.
```
kafka-console-consumer --topic my-topic --bootstrap-server kafka:9092
```

- 위에서 consumer를 실행한 터미널을 그대로 두고 새로운 터미널을 연다.
- data를 topic으로 전달하는 producer를 실행한다.
```bash
kafka-console-producer --topic my-topic --broker-list kafka:9092
```
- producer에서 hello 등의 메시지를 입력하면 consumer를 실행한 터미널 창에서 같은 내용을 출력하는 걸 볼수 있다.
