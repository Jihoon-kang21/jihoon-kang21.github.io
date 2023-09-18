---
layout: post
title: python - sentinel redis에 연결해서 key리스트와 각 key의 length 구하기
subtitle: master를 찾아 key list를 구하고 타입별로 length 구하기
categories: tech
tags: [python,redis]
---

## Situation

### 스크립트의 용도
- Redis commander 로 GUI 환경에서 Redis에 쌓인 Queue를 수동 모니터링하다가, Redis commander 가 원인을 알수 없는 상태로 뻗음. 모니터링은 해야되고 해서 급하게 대충 짠 스크립트

### 조건
- Redis Sentinel로 구성되어있음
	- 3대 서버 : ('10.156.133.128', 10063), ('10.156.133.134', 10067), ('10.156.133.126', 10079)
	- master가 어느 서버인지 모르는 상태
- 어떤 data (key) 가 있는지도 모르는 상태

### 목표
- key의 리스트를 구하고 각 key의 length를 구해야한다.

## Solution

### master 구하기

- 주어진 서버 3개의 정보로 sentinel 이라는 이름의 Sentinel instance를 만든다.
```python
sentinel = Sentinel([('10.111.122.133', 10003), ('10.111.122.134', 10004), ('10.111.122.135', 10005)], socket_timeout=0.1)
```

- discover_master method로 현재 master Redis instance를 찾을수 있다.
- mymaster는 master의 이름이고, default가 mymaster이다.
```python
print(sentinel.discover_master("mymaster"))
```

- master Redis instance에 연결된 'master' instance를 만든다.
- master 접속에 필요한 password가 주어졌다면 password도 전달한다.
```python
master = sentinel.master_for('mymaster', socket_timeout=0.1, password='yourpassword')
```


- Redis DB의 모든 key를 'keys'에 저장한다.
```python
keys = master.keys('*')
```

- master.type으로 key의 type을 구한다.
```python
master.type()
```

- type이 set인경우 scard, hash인 경우 hlen으로 key의 length를 구한다.
```python
master.scard()
master.hlen()
```

## 스크립트 전체
```python
from redis.sentinel import Sentinel
from redis import Redis

sentinel = Sentinel([('10.111.122.133', 10003), ('10.111.122.134', 10004), ('10.111.122.135', 10005)], socket_timeout=0.1)
master = sentinel.master_for('mymaster', socket_timeout=0.1, password='yourpassword')
keys = master.keys('*')

for i in keys:
	if master.type(i) == b'set':
		print(i, master.type(i), master.scard(i))
	elif master.type(i) == b'hash':
		print(i, master.type(i), master.hlen(i))
	else:
		print(i, master.type(i), "How to count this?")
```

