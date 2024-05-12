---
layout: post
title: Python - fastapi - response로 zip 파일 보내기
subtitle: BytesIo, ZipFile 활용하기
categories: Tech
tags: [python,fastapi]
---

## 배경

- Fastapi로 api 기능을 제공한다.
- data를 여러개의 파일들로 저장하여 zip 파일로 압축한 형태로 저장할 수 있게 한다.
- 사용자는 request를 보내면 zip 파일을 다운 받을 수 있다.

## script

zip 파일을 다운받을 수 있는 api script
```python
import zipfile
from io import BytesIO

app = FastAPI()

@app.get("/download-zip")
async def download_zip_file():
    # Create a temporary in-memory buffer
    buffer = BytesIO()
    
    # Create a zip file in the buffer
    with zipfile.ZipFile(buffer, "w", zipfile.ZIP_DEFLATED) as z:
        # Write simple text files directly into the ZIP
        z.writestr("hello.txt", "Hello, this is the first text file.")
        z.writestr("world.txt", "And this is the second text file.")

    # Rewind the buffer
    buffer.seek(0)

    # Set the headers to inform the browser about the download nature of the response
    headers = {"Content-Disposition": 'attachment; filename="example_texts.zip"'}

    # Return the buffer content as a zip file response
    return Response(content=buffer.read(), media_type="application/zip", headers=headers)

# This function is to run the server
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000)
```

## request

위 스크립트를 `main.py`로 저장하고 `python main.py`명령어를 실행한 다음, 아래 명령어로 request를 보내면 hello.txt, world.txt 파일이 압축되어있는 zip 파일을 다운받을 수 있다.

```bash
curl http://localhost:8000/download-zip -o test.zip
```

## BytesIo와 ZipFile

### BytesIo

- BytesIO는 파이썬의 io 모듈에서 제공하는 클래스이다. 
- 파일처럼 사용할 수 있는 메모리 블록(버퍼)을 관리하는 방법을 제공한다.
- 실제 디스크에 읽기 또는 쓰기를 수행할 필요 없이 파일처럼 데이터를 처리해야 할 때 유용하다.

간단한 예제:

```python
from io import BytesIO

# 버퍼 생성
buffer = BytesIO()

# 버퍼에 데이터 쓰기
buffer.write(b"Hello world!")

# 버퍼에서 데이터 읽기
buffer.seek(0)  # 읽기 커서를 버퍼의 시작 부분으로 이동
print(buffer.read())  # 출력: b'Hello world!'
```

- 이 예제에서 b"Hello world!"는 이진 데이터를 나타낸다.
- write 메서드는 이 데이터를 버퍼에 추가하고, seek(0)은 데이터를 다시 읽을 수 있도록 커서를 시작 부분으로 리셋한다.

### zipfile 

- zipfile은 ZIP 파일을 작업할 수 있도록 하는 파이썬 모듈이다.
- 새 ZIP 파일을 생성하고, 파일을 추가하고, 파일을 추출하는 등의 작업을 수행할 수 있다.
- 예시 스크립트에서 zipfile은 디스크가 아닌 메모리에 ZIP 파일을 생성하는 데 사용된다.

간단한 예제:

```python
from zipfile import ZipFile
from io import BytesIO

# ZIP 파일을 저장할 버퍼 생성
buffer = BytesIO()

# 버퍼에 ZIP 파일 생성
with ZipFile(buffer, 'w') as zip_file:
    # ZIP에 간단한 텍스트 파일 생성
    zip_file.writestr('sample.txt', 'This is some sample text.')

# ZIP 내용에 접근하려면 버퍼를 리와인드해야 함
buffer.seek(0)
with ZipFile(buffer, 'r') as zip_file:
    print(zip_file.read('sample.txt'))  # 출력: b'This is some sample text.'
```

- 이 예제에서 ZipFile은 writestr을 사용하여 ZIP에 직접 텍스트 파일을 생성한다.
- 디스크에 임시 파일을 만들지 않고 ZIP 파일을 생성하는 데 유용하다.

### seek()

- Python에서 파일처럼 작동하는 객체와 작업할 때 seek() 메서드는 매우 중요하다.
- 파일 커서의 현재 위치를 변경하는 데 사용되며, 특정 지점에서 파일 읽기 또는 쓰기를 시작해야 할 때 중요하다.

간단한 예제:

```python
from io import BytesIO

# 버퍼 생성 및 데이터 작성
buffer = BytesIO(b"Hello world!")
buffer.write(b" More text.")

# 데이터의 시작지점으로 이동하여 데이터를 읽는다
buffer.seek(0)
print(buffer.read())  # 결과: b'Hello world! More text.'

# 데이터의 6번째 byte로 이동하여 데이터를 읽는다
buffer.seek(5)
print(buffer.read())  # 결과: b' world! More text.'
````
