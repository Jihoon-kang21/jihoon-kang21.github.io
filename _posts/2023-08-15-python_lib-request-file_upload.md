---
layout: post
title: 파이썬 request 라이브러리를 이용해서 파일 업로드하기
subtitle: 파일을 open, read 하여 requests.post로 특정 url에 파일을 업로드하는 예시
categories: CodeExample
tags: [python]
---

아래는 조건

서버 : 10.120.1.234  
endpoint : 10.120.1.234/uploadfile  
업로드할 파일 이름 : upload_please.txt  
필요한 결과 : 업로드 요청 후 응답

### 1. 변수 정의하고 파일 읽어서 dictionary 형태로 저장하기

```
import requests

url = 'http://10.120.1.234'
txt_file = open('upload_please.txt', 'rb')
txt = {'file': txt_file}
```

open 함수 이용해서 바이너리 형태(rb 옵션)로 파읽을 읽고  
txt라는 이름으로 dictionary 형태로 저장했다.  
이때 key는 'file'로 지정한다.

### 2. 업로드 요청하고 응답받기

```

response = requests.post(f"{url}/uploadfile", files=txt)
print(response.status_code)
print(response.text)
```

post 메소드로 요청을한다.  
요청에서 txt 변수를 넘겨줄때는 'files'로 넘긴다.  
response 함수 status_code 로 에러 코드를 확인받고  
text로 응답 내용을 print한다

### 3. 전체 코드

```
import requests

url = 'http://10.120.1.234'
txt_file = open('upload_please.txt', 'rb')
txt = {'file': txt_file}

# Upload
response = requests.post(f"{url}/uploadfile", files=txt)
print(response.status_code)
print(response.text)
```

  

