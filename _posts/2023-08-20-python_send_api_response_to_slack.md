---
layout: post
title: api 응답을 slack 메시지로 전달하기
subtitle: 파일을 open, read 하여 requests.post로 특정 url에 파일을 업로드하는 예시 스크립트
categories: tech
tags: [python]
---

# 상황
어떠한 application을 실행하는 서버 6대가 있다.
이 application을 실행할 수 있는 횟수는 라이선스로 제한되어있다.
라이선스를 계속 체크해서 주어진 횟수의 80%를 소진하면 slack으로 알림을 받고 싶다.
라이선스를 확인할 수 있는 api가 서버로부터 제공되고 ip주소를 통해 접속할수 있다.

# script
```python
#!/usr/bin/env python3

# https://api.slack.com/apps 에서 app 생성후 token 생성가능
# SLACK_BOT_TOKEN=xoxb-xxxxxxx

# 크론탭으로 스크립트 실행

import requests
import os
from slack_sdk import WebClient
from slack_sdk.errors import SlackApiError

CHANNEL_NAME = "#test-slack-channel"
SERVER_DIC = {'192.168.1.124': 'server-01',
              '192.168.1.125': 'server-02',
              '192.168.1.126': 'server-03',
              '192.168.1.127': 'server-04',
              '192.168.1.128': 'server-05',
              '192.168.1.129': 'server-06'}

SERVER_IP_LIST = list(SERVER_DIC.keys())

PERCENTAGE = 0.8

# Slack token 값은 스크립트 실행시 입력
slack_token = os.environ["SLACK_BOT_TOKEN"]
client = WebClient(token=slack_token)

init_message = ""


def send_alarm(server_ip_list, channel_name, percentage):

    message = init_message

    for server_ip in server_ip_list:

        # 라이센스 정보 불러오기
        response = requests.post(f"http://{server_ip}:1234/api/keys/")

        # message += "*IP : %s* \n" % SERVER_DIC[server_ip]

        # 불러온 라이센스 정보 json 형태로 저장하기
        license_info_list = response.json()

        for license_info in license_info_list:

            # product name
            product = license_info['product']

            # 현재 실행 횟수가 0인 경우 skip
            if "current_usage_count" not in license_info:
                continue

            # 현재 실행 수
            current_usage_count = license_info['current_usage_count']

            # 실행 가능 최대 장수
            limit_usage = license_info['limit_usage_count']

            # 남은 장수
            remain_count = limit_usage - current_usage_count

            # 알람 기준 값 -> limit의 80%로 할것
            alarm_threshold = limit_usage * percentage

            # 실행 수가 limit 를 초과할 때까지 하루 한번 반복 실행
            if current_usage_count > alarm_threshold:
                message += f"*IP : {SERVER_DIC[server_ip]}* \n" \
                           f"```" \
                           f"Product : {product} \n " \
                           f"limit : {limit_usage} \n " \
                           f"current usage count : {current_usage_count} (over {percentage*100} % of limit) \n " \
                           f"remain count : {remain_count}" \
                           f"``` \n" \


    if message != init_message:
        try:
            response = client.chat_postMessage(
                channel=channel_name,
                text=message
            )
        except SlackApiError as e:
            assert e.response["error"]


def main():
    send_alarm(SERVER_IP_LIST, CHANNEL_NAME, PERCENTAGE)


main()
```
