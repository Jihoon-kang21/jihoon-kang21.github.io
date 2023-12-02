---
layout: post
title: mssql with k8s 또는 docker - volume mount했을때 sqlservr실행시 system directory permission error 
subtitle: mssql를 k8s 또는 docker로 실행하기
categories: troubleshooting
tags: [docker,k8s,mssql,troubleshooting]
---

## Situation

- mssql 을 k8s 환경에서 실행한다
- /opt/mssql/bin/sqlservr 를 실행하고 DB, table등을 생성하는 sql문을 실행해야한다.
- 그런데 volume 을 mount해서 실행하면 system directory(.system 폴더) 관련 permission error가 발생한다.
- 서버 띄우는 명령어 실행할때 권한과 directory 권한 차이인가해서 폴더를 미리 만들어서 권한 지정한다는지, root로 pod를 실행한다든지 이것저것 해봐도 안된다.
- 이리저리 헤매다가 아래처럼 해서 일단 사용할 수 있게 만들었다.

## 1. Dockerfile

- password 관련해서 두가지 설정이 있는데 버전마다 다르다고 한다. 그냥 둘다 설정했다.
	- `SA_PASSWORD`, `MSSQL_SA_PASSWORD`
- mssql 사용하려면 EULA에 동의해야된다. 그래서 넣은게 아래줄
	- `ENV ACCEPT_EULA=Y` 
- sql server를 실행하는 command는 run.sh 파일에 넣어놓는다.
```
FROM mcr.microsoft.com/mssql/server:2022-latest

ENV TZ=Asia/Seoul
ENV ACCEPT_EULA=Y
ENV SA_PASSWORD=Passwo0d!
ENV MSSQL_SA_PASSWORD=Passwo0d!

COPY . /app
WORKDIR /app

ENTRYPOINT ["bash", "/app/run.sh"]
```

## 2. run.sh 

- 처음 실행할때 sqlservr를 바로 실행하면 permission에러가 발생한다.
- 일단 sleep을 걸어서 pod에 접속할 수 있도록 만들어 놓는다.
```
#!/bin/bash
sleep infinity
```


## 3. k8s deployment.yml

- securityContext : pod에 대한 권한 및 액세스 제어 설정이다. mssql 그룹의 GID 값이 10001이다.
- 그 외 불륨 마운트 설정, 포트번호  k8s 관련 추가 설정은 필요없다.
```yaml
spec:
  template:
    spec:
      securityContext:
        fsGroup: 10001
```

## 4. pod 진입 및 sqlservr, sql 실행

1. pod 에 진입
2. `/opt/mssql/bin/sqlservr` 실행
3. sql 문 실행

## 5. run.sh 수정 및 재배포

- volume을 마운트했다는 가정이므로 DB, Table들이 생성된 채로 volume에 저장되어 있다.
- run.sh 파일을 아래 처럼 수정한다.
```
#!/bin/bash
#sleep infinity
/opt/mssql/bin/sqlservr
```
- deployment.yml 파일을 이용해서 다시 docker image 및 pod를 생성해보면 이상없이 잘 된다.
