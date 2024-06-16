---
layout: post
title: git - master branch와 rebase하고 conflict 해결하기
subtitle: git rebase 기본 사용법
categories: Tech
tags: [git]
---

## 문제

GitHub을 사용하다 보면, 다른 branch에서 작업하는 동안 동료와 같은 파일을 수정하는 상황이 발생할 수 있다. 이런 경우, conflict이 발생할 수 있으며, 이를 해결하기 위해 rebase 과정을 거쳐야 한다. 이 블로그에서는 실제 상황을 예로 들어, master branch와 rebase하는 방법과 conflict을 해결하는 방법을 단계별로 안내한다.

## 상황

- **현재 branch**: `jihoon/new_feature`
- **commit한 파일**: `feature.txt`
- **동료의 변경사항**: `master` branch에 push됨

최신 `master` branch의 변경사항을 가져오고, 동료의 기능이 포함되도록 내 branch를 rebase해야 한다. 이 과정에서 `feature.txt` 파일에 conflict이 발생했다.

## 단계별 가이드

### 1단계: 기능 branch를 checkout하기

먼저, 현재 위치가 branch `jihoon/new_feature`가 맞는지 확인한다.

```bash
git checkout jihoon/new_feature
```

출력:
```
Switched to branch 'jihoon/new_feature'
```

### 2단계: 원격 저장소에서 최신 변경사항 가져오기

다음으로, 원격 저장소에서 최신 변경사항을 가져와서 `master` branch가 최신 상태인지 확인한다.

```bash
git fetch origin
```

출력:
```
From https://github.com/your-repo
 * branch            master     -> FETCH_HEAD
 * [new branch]      master     -> origin/master
```

### 3단계: 기능 branch를 최신 master로 rebase하기

이제, `jihoon/new_feature` branch를 최신 `master` branch로 rebase한다. 이를 통해 내 변경사항이 `master`의 commit 위에 다시 적용됩니다.

```bash
git rebase origin/master
```

출력:
```
First, rewinding head to replay your work on top of it...
Applying: Your commit message
Using index info to reconstruct a base tree...
M       feature.txt
Falling back to patching base and 3-way merge...
Auto-merging feature.txt
CONFLICT (content): Merge conflict in feature.txt
Automatic merge failed; fix conflicts and then commit the result.
```

### 4단계: conflict 해결하기

위 메시지에서 `feature.txt`에 conflict이 있음을 확인했다. conflict이 있는 파일을 열어보면 master가 가지고 있는 내용과 내 branch에서 작업한 내용이 둘다 보인다. 둘중 하나는 삭제하도록 한다. `<<<< HEAD`, `>>>> Your commit message`등도 삭제한다.

**해결 전:**
```plaintext
# feature.txt
Line 1
<<<<<<< HEAD
Line from master branch
=======
Line from your feature branch
>>>>>>> Your commit message
Line 3
```

불필요한 내용을 삭제하면 다음과 같다.

**해결 후:**
```plaintext
# feature.txt
Line 1
Line from master branch
Line from your feature branch
Line 3
```

파일을 수정한 후, conflict이 해결되었음을 표시하기 위해 스테이징 영역에 추가한다.

```bash
git add feature.txt
```

rebase 과정을 계속 진행한다:

```bash
git rebase --continue
```

출력:
```
Applying: Your commit message
```

### 5단계: rebase 확인하기

rebase가 성공했는지 확인하기 위해 commit 히스토리를 확인한다.

```bash
git log --oneline --graph --all
```

출력:
```
* abc1234 (HEAD -> jihoon/new_feature) - Your commit message
* def5678 - 동료의 commit message
* ghi9012 (origin/master) - Previous master commit
```

### 6단계: 변경사항 강제 push하기

commit 히스토리를 재작성했기 때문에, 변경사항을 원격 branch에 강제로 push해야 한다.

```bash
git push origin jihoon/new_feature --force
```

출력:
```
Total 0 (delta 0), reused 0 (delta 0)
To https://github.com/your-repo
 + d4e5f6g...abc1234 jihoon/new_feature -> jihoon/new_feature (forced update)
```

## 다이어그램

### rebase 전

```
master:
* c3d2e1c (origin/master) - Master commit 3
* a1b2c3d - Master commit 2
* 9f8e7d6 - Master commit 1

jihoon/new_feature:
* e5f6g7h (jihoon/new_feature) - Feature commit 2
* d4e5f6g - Feature commit 1
```

### rebase 중

```
(master remains unchanged)

origin/master:
* c3d2e1c - Master commit 3
* a1b2c3d - Master commit 2
* 9f8e7d6 - Master commit 1

Rebasing jihoon/new_feature:
pick d4e5f6g Feature commit 1
pick e5f6g7h Feature commit 2
```

### conflict 해결 중

```
(master remains unchanged)

origin/master:
* c3d2e1c - Master commit 3
* a1b2c3d - Master commit 2
* 9f8e7d6 - Master commit 1

jihoon/new_feature (conflict resolution):
pick d4e5f6g Feature commit 1 (conflict in feature.txt)
pick e5f6g7h Feature commit 2 (conflict resolved)
```

### rebase 후

```
(master remains unchanged)

origin/master:
* c3d2e1c - Master commit 3
* a1b2c3d - Master commit 2
* 9f8e7d6 - Master commit 1

jihoon/new_feature:
* f6g7h8i (HEAD -> jihoon/new_feature) - Feature commit 2 (rebased)
* e5f6g7h - Feature commit 1 (rebased)
* c3d2e1c - Master commit 3
* a1b2c3d - Master commit 2
* 9f8e7d6 - Master commit 1
```

## 결론

이 단계들을 따라하면 기능 branch를 최신 master branch와 성공적으로 rebase하고, 발생한 conflict을 해결할 수 있다. 이렇게 하면 내 branch에 동료의 최신 변경사항이 포함되고 commit 히스토리를 깔끔하게 유지할 수 있다.