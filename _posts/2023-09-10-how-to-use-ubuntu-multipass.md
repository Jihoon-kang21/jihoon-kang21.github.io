---
layout: post
title: Mac에서 multipass 이용해 간단한 Ubuntu VM 만들기
subtitle: 간단한 multipass 명령어 리스트
categories: tech
tags: [mac,vm,ubuntu]
---

### Multipass 설치
```bash
brew install --cask multipass
```
### Multipass 사용

- 인스턴스 생성
```bash
multipass launch
```
사용할 버전을 명시해줄 수도 있다.  
```bash
multipass launch --name instance_name 18.04 
```

multipass launch 실패시
```
[2023-08-22T21:56:42.728] [error] [client] Failed to set up autostart prerequisites: failed to link file '/Users/gausslabs/Library/LaunchAgents/com.canonical.multipass.gui.autostart.plist' to '/Library/Application Support/com.canonical.multipass/Resources/com.canonical.multipass.gui.autostart.plist'
launch failed: Downloaded image hash does not match
```
--> System settings - General -   Login Items - Open at Login에 multipass 추가
-->
```
sudo cp /Library/Application\ Support/com.canonical.multipass/Resources/com.canonical.multipass.gui.autostart.plist /Users/gausslabs/Library/LaunchAgents/
```


- 옵션 값을 통해 인스턴스의 스펙을 조절해줄 수 있다.
```bash
multipass launch --cpus <cpus> --disk <disk> --memory <mem> --name <name>
```
`-c, --cpus <cpus>`
    - 할당할 CPU의 개수
    - 최소값 : 1, 기본값 : 1
`-d, --disk <disk>`
    - 할당할 저장공간
    - 기본적으로 byte 단위이며, `K`, `M`, `G` 접미사를 붙여서 단위를 지정할 수 있다.
`-m, --mem <mem>`
    - 할당할 메모리
    - 기본적으로 byte 단위이며, `K`, `M`, `G` 접미사를 붙여서 단위를 지정할 수 있다.
`-n, --name <name>`
    - 인스턴스의 이름을 지정해준다.

### multipass 사용

- 인스턴스 목록 조회
```bash
multipass ls
```

- 인스턴스 Shell 접속
```bash
multipass shell <instance name>
```

- 명령 실행
```bash
multipass exec <instance name> -- <명령어>
```

- host에서 파일 복사
```bash
multipass transfer local_file.txt <instance name>:.
```

- 인스턴스 정지 및 시작

```bash
multipass stop <instance name>
multipass start <instance name>
```

- 인스턴스 삭제

```bash
multipass delete <instance name>
```

- 인스턴스 복구

```bash
multipass recover <instance name>
```

- deleted 상태인 인스턴스 영구 삭제

```bash
multipass purge
```

`purge` 명령어를 통해 _deleted_ 상태인 인스턴스를 영구 삭제한다.

### note.

M2 에서 테스트했을때 인터넷 연결이 되지 않는다. 해결방법 아직 찾지 못했음.
Virtual box 는 아예 M2에서 VM 만들어지지가 않아서 그 대체로 사용.

