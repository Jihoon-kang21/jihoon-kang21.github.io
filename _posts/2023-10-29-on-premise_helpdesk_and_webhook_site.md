---
layout: post
title: Jitbit을 통한 On-premise 고객 지원
subtitle: Jitbit을 설치하고 webhook을 테스트해본다
categories: Tech
tags: [helpdesk,on-premise,webhook]
---

## 소개

고객 지원은 계속 해서 발전하고, 고객의 특수한 요구 사항을 충족하는 적절한 도구를 찾는 것은 중요하고도 어렵다. 흔한 요구사항 중 하나인 데이터에 대한 보안과 고객 지원 방법이 서로 상충 될때가 있는게 어려운 이유 중 하나이다. 이러한 이유를 배경으로 이 블로그 게시물에서는 Jitbit를 선택하고, Docker를 사용하여 간단히 설치하고, webhook.site를 사용하여 웹훅 통합을 테스트하는 내용을 소개한다.
## 1. Helpdesk Tool로써 Jitbit 선택의 상황 및 이유

### 우리의 문제 상황

이번 경우는 사용자가 어떠한 내용도 외부로 반출되는 것을 원하지 않는 경우다. 사용자인 고객사 엔지니어들은 블로그, 외부 메일 등을 허가 없이 쓸 수 없다는 것이다. 고객지원을 위해 Helpdesk 툴을 사용해야하지만 SaaS 툴은 사용할 수 없는 것이다. SaaS 툴은 클라우드(인터넷)에 설치되어 있고 보통의 Helpdesk 툴은 SaaS로 운영된다. 그래서 On-premise 형태로 설치할 수 있는 Helpdesk 툴을 찾았고 그 중에 가성비가 좋은 Jitbit을 선택하기로 했다.

### Jitbit의 선택 이유

- On-premise 설치 가능 - 고객의 가장 강력한 요구 사항이었음
- Webhook 지원 - webhook을 통해 고객사 자체 서비스에도 API를 통해 문의 답변 내용을 업데이트 해주어야 했음
-  SSO 지원 - 사용자들이 별도 회원가입없이 기존 계정을 이용할 수 있게 지원
-  가성비 - 업데이트가 필요없다면 설치할 때 한번만 지불하면 됨

## 2. Docker를 사용한 Jitbit 설치

고객사 서버가 Kubernates로 운영되고 있어서 docker image를 전달하는 형태로 설치를 진행해야 했다.

설치 방법 (메뉴얼 : https://www.jitbit.com/docs/helpdesk/!!!helpdesk-software-readme.htm)
1. https://www.jitbit.com/helpdesk/ 에서 Download
2. 다운받은 파일을 압축해제하면 HelpDeskTrial/Helpdesk 폴더아래 Dockerfile과 docker-compose.yml 파일이 있다.
3. docker가 설치된 상태라면 터미널을 이용해 위 경로에서 `docker-compose up -d` 명령어를 실행
4. 브라우저를 열고 http://localhost:80 에 접속하면 Jibit 에 연결된다.
5. 초기 아이디와 비번은 admin / admin dlek.

## 3. webhook.site를 이용한 Webhook 테스트

Jitbit에 ticket이 만들어지거나 답변이 등록된 경우 webhook을 생성하여 타 서비스 API 요청을 할 수 있어야 했다.
타 서비스 API 내용을 들여다 보기전에 Jibit에서 원하는 데로 webhook이 잘 생성되는지 확인한다.

1. https://webhook.site 에 접속하면 webhook를 받아서 내용을 확인할 수 있는 url 이 제공된다. 
   ![webhook.site](/assets/images/posts/webhooksite.png)
2. Jitbit - admin 계정으로 로그인 - Administration 메뉴 - Automation rules에서 rule을 생성한다
3. Webhook을 생성한 조건을 선택하고, POST method를 선택, 제공받은 url을 입력한다.
4. body에는 넣고 싶은 정보를 입력한다. 아래는 예시이다.
   ```
   subject: #subject#
   url : #url#
   from : #from#
   from-email : #fromEmail#
   body : "#body#"
   technician : #technician#
   custom field : #cf_1#
   date : "#date#"
   ticketid : #ticketId#
   priority : #priority#
   status : #statusId#"
   category : #category#
   most recent message : #most_recent_message#
   ```
   ![jitbit-automation](/assets/images/posts/jitbit-automation.png)
5. 아래 처럼 ticket 내용을 포함한 webhook을 확인할 수 있다.
   ![webhooksite-result](/assets/images/posts/webhooksite-result.png)


