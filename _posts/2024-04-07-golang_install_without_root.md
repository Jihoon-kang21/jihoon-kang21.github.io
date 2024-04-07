---
layout: post
title: Golang - root 권한없이 사용할 수 있게 설치(Linux)
subtitle: home 폴더에 설치하고 환경 변수 수정해서 root 권한없이 golang 실행
categories: Tech
tags: [golang]
---

## 문제

공식 홈페이지에 나와있는 설치 명령어는 아래와 같다
```bash
 rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.1.linux-amd64.tar.gz
 ```
/usr/local 폴더는 root 권한이 있어야 폴더를 만들 수 있다. 그래서 그대로 설치하면 명령어에 따라 root 권한을 요구하게 된다.
root 권한을 쓸 수 없는 경우 go 명령어를 사용할 수 없게된다.

## 해결 

home 폴더 위치가 /home/jihoon 이라고 한다면, go file을 다운받아 놓은 위치에서 다음 명령어를 실행하여 설치한다. 홈페이지에 나와있는 명령어에서 경로를 /usr/local/ 이 아닌 홈 폴더에 설치한다.
```
rm -rf ~/local/go && tar -C /home/jihoon/local -xzf go1.22.1.linux-amd64.tar.gz
```

go 가 /home/jihoon/local/go/bin 에서 실행할 수 있게 환경변수를 추가해준다.
```
export PATH=$PATH:/home/jihoon/local/go/bin
```

## 환경 변수 입력 위치 (ubuntu 18 버전기준)

영구적으로 환경 변수를 추가위해 아래 파일 중 하나에 위 export 명령어를 추가하면 된다.
아래 파일은 현재 유저에 한해 적용된다.
```
~/.bashrc
~/.bash_profile
```

sudo bash 명령어로 root 권한 획득이 가능하다면 아래파일 중 하나를 편집하여 서버전체에 적용할 수 있다.
```
/etc/profile
/etc/bash.bashrc
```

파일을 수정하고 나면 수정한 파일을 실행하거나 exit 로 터미널을 종료하고 재접속하면 적용된다.
```
source ~/.bashrc
```
