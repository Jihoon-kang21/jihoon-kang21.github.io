---
layout: post
title: Git - giclone SSL certification problem 에러 해결
subtitle: Git ssl verify option 끄기
categories: Troubleshooting
tags: [git]
---

## 0. 배경

gitclone 하려는데 다음 에러 발생

```
* SSL certificate problem: self signed certificate
* Closing connection 0
fatal: unable to access 'https://server.name/git/testing.git/': SSL certificate problem: self signed certificate
```

## 1. 해결

git config 명령어로 sslVerify 옵션을 끈다
아래 명령은 모든 git project 에 적용
```
git config --global http.sslVerify false
```

아래 명령은 gitclone을 실행할때 한번 적용
```
git -c http.sslVerify=false clone https://example.com/path/to/git
```


## 2. 원인

git 은 https repository 연결시 curl 을 사용하여 연결하는데 curl 의 SSL 인증서 검증 옵션때문에 오류가 발생
curl 은 기본적으로 https 사이트의 SSL 인증서를 검증한다. 인증 기관의 인증서 목록이 없거나 모르는 기관에서 발급한 인증서일 경우 오류 발생

```
static CURL *get_curl_handle(void)
{
        CURL *result = curl_easy_init();
        if (!curl_ssl_verify) {
                curl_easy_setopt(result, CURLOPT_SSL_VERIFYPEER, 0);
                curl_easy_setopt(result, CURLOPT_SSL_VERIFYHOST, 0);
        } else {
                /* Verify authenticity of the peer's certificate */
                curl_easy_setopt(result, CURLOPT_SSL_VERIFYPEER, 1);
                /* The name in the cert must match whom we tried to connect */
                curl_easy_setopt(result, CURLOPT_SSL_VERIFYHOST, 2);
        }
        ...
}
```

