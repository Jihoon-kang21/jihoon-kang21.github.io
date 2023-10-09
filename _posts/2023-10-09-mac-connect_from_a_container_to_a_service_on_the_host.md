---
layout: post
title: Mac-container에서 host로 접근
subtitle: Mac에서 docker desktop을 설치해서 쓰는 경우 docker0 interface가 없다
categories: tech
tags: [mac,docker,network]
---

https://docs.docker.com/desktop/networking/#use-cases-and-workarounds-for-all-platforms

## 문제

host에 서비스 A가 실행되고 있고 docker container로 서비스 B가 실행되고 있다.
아래와 같은 상황에서  서비스 B에서 서비스 A 로 API request를 보내야 하는 경우

1. host의 IP 주소가 계속 바뀌는 경우
2. 인터넷 접속이 안되는 경우
3. mac에 Docker Desktop을 설치해서 쓰기 때문에 host에 docker0 interface가 없는 경우

## 해결책

host.docker.internel 이라는 도메인을 쓰면 host로 접근할 수 있다.

## 예시

alpine container에서 host에 설치한 서버로 접근하기

1. python으로 port 번호 8000에 http server를 실행한다.

```
python -m http.server 8000
```
 
2. alpine 을 docker container로 실행해서 접근하고, curl를 추가한다음
3. curl을 이용해서 host에 접근한다.
```console
$ docker run --rm -it alpine sh
# apk add curl
# curl http://host.docker.internal:8000
# exit
```

4.  `apk add curl` 에서 아래 에러 발생시
```
SSL routines:tls_post_process_server_certificate:certificate verify failed:ssl/statem/statem_clnt.c:1889:
WARNING: updating and opening https://dl-cdn.alpinelinux.org/alpine/v3.18/main: Permission denied
```
아래 명령어 실행 후 재시도
```
sed 's/https/http/g' -i /etc/apk/repositories
```

