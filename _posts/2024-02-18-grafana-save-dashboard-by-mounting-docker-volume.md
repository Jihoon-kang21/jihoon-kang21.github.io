---
layout: post
title: Grafana - Docker volume mount 해서 dashboard 저장내용 유지하기
subtitle: container를 다시 실행해도 dashboard 저장내용 유지 하기
categories: Tech
tags: [grafana,docker]
---

## 배경

Linux에서 Docker 이미지로 Grafana를 서버에 구성했을 때, 아무런 옵션없이 그냥 docker run 만 한 경우, 대시보드를 만들어서 저장해놓더라도 컨테이너를 삭제하고 다시 생성하면 저장내용이 사라져있다.

## 마운트 위치

컨테이너를 삭제하고 재생성하는 일이 있더라도 대시보드 작업 내용을 저장하기 위해서는 아래 경로를 볼륨 또는 호스트에 마운트해야한다. 

```
/var/lib/grafana
```

## docker-compose 이용해서 volume mount 하기
아래는 volume으로 마운트한 예시 docker-compose.yml 파일이다.

```

  grafana:
    image: grafana/grafana:7.4.3-ubuntu
    container_name: grafana
    ports:
      - 3000:3000
    volumes:
      - data-grafana:/var/lib/grafana
    restart: always
volumes:
  data-grafana:
```

data-grafana 이름으로 volume을 생성하고 grafana 컨테이너 내부 /var/lib/grafana 내용을 data-grafana에 저장한다.

이렇게 하면 volume을 지우지 않는 이상 docker 컨테이너를 재생성하더라도 대시보드를 저장한 내용이 volume에 남기때문에 다시 대시보드를 만드는 수고를 덜 수 있다.

## docker-compose 실행하기
grafana 이미지를 띄운다음 위 docker-compose.yml 파일이 있는 경로에서 아래 명령어를 실행하면 대시보드 저장 내용을 유지할 수 있는 grafana를 실행할 수 있다.

```

$ docker-compose up -d
```

## 참고 Grafana manual
아래는 docker image로 grafana 실행하기 관한 링크

[https://grafana.com/docs/grafana/latest/installation/docker/](https://grafana.com/docs/grafana/latest/installation/docker/)

아래는 /var/lib/garafana 외 다른 configuration들

[https://grafana.com/docs/grafana/latest/administration/configure-docker/](https://grafana.com/docs/grafana/latest/administration/configure-docker/)

## 덧.

Dashboard json 파일을 host에서 저장해서 직접 마운트하는 방법도 있다. provision 하는 법을 찾으면 된다.
하지만 provision으로 직접 파일을 마운트 하면 Grafana UI에서 대시보드를 수정해서 저장할 수가 없다. 
