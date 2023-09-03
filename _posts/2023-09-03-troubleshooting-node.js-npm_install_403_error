---
layout: post
title: npm install - 403 Forbidden error
subtitle: 403 error를 해결하고 private registry를 이용하는 방법
categories: troubleshooting
tags: [nodejs]
---

## 이슈

node.js 설치후 필요한 모듈을 설치하려고 하는데 403 에러가 발생
```bash
$ npm install puppeteer
npm ERR! code E403
npm ERR! 403 403 Forbidden - GET https://registry.npmjs.org/puppeteer
npm ERR! 403 In most cases, you or one of your dependencies are requesting
npm ERR! 403 a package version that is forbidden by your security policy.

npm ERR! A complete log of this run can be found in:
npm ERR!     /home/****/.npm/_logs/2023-08-30T02-14_00_787Z-debug.log
```

## 해결

config에 설정된 repo에 접급권한이 없어서 실패했다.
기본 repository는 `https://registry.npmjs.org` 이곳이다.
문제가 발생한 환경은 보안문제로 공개 repo를 사용할 수 없고 지정된 repo를 사용해야 한다.
참고하는 주소를 바꿔서 문제 해결
```
$ npm config set registry http://nexus.XXXXX.com/repository/npm-group/
$ npm install --global yarn
$ yarn --version
```

npm config 내용 확인
`$ npm config list`
