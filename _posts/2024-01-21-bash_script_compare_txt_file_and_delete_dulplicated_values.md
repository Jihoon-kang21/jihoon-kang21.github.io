---
layout: post
title: bash - 텍스트 파일 두개 비교해서 중복값을 제외하고 저장하기
subtitle: array, IFS, read 활용
categories: CodeExample
tags: [bash]
---

## Problem

a.txt 내용은 다음과 같다
```
a
b
c
```
b.txt 내용은 다음과 같다
```
a
b
c
d
q
o
p
```
b.txt 내용 중에서 a.txt 에 없는 값만 남기고 싶다.

## 구글링 통해 작성한 Script

아래는 구글링 통해 만든 스크립트

```bash
#!/bin/bash

# a.txt, b.txt 파일 내용을 아래 각 array에 저장할 예정
array1=()
array2=()

file1='a.txt'
file2='b.txt'

# a.txt 내용은 array1에 저장한다.
for i in 'cat $file1'
do
	array1+=($i)
done

# b.txt 내용은 array2에 저장한다.
for j in 'cat $file2'
do
	array2+=($j)
done

# array1에 있는 모든 값에 대해 array2에 존재하는지 확인하고, 있으면 array2에서 제외 한다.
for i in `seq 0 ${#array2[@]}`
do
	for j in `seq 0 ${#array1[@]}`
	do
		if [[ ${array2[$i]} == ${array1[$j]} ]]
		then
			unset 'array2[$i]'
		fi
	done
done

# array1에 존재하는 값을 뺀 array2 내용을 result.txt에 저장한다.
for line in "${array2[@]}"
do
	echo $line >> result.txt
done

```

## Chat GPT가 고쳐준 Script

위 코드를 Chat GPT한테 보여줬더니 아래 처럼 고쳐쓰라고 한다.

```bash
#!/bin/bash
array1=()
array2=()

file1='a.txt'
file2='b.txt'

# Reading elements from file1 into array1
while IFS= read -r line; do
    array1+=("$line")
done < "$file1"

# Reading elements from file2 into array2
while IFS= read -r line; do
    array2+=("$line")
done < "$file2"

# Removing common elements from array2
for i in "${array1[@]}"; do
    array2=("${array2[@]/$i}")
done

# array2에서 array1에 있는 값을 뺀 항은 null로 존재한다. array2에서 값이 존재하는 항만 array3으로 저장한다. array3을 출력하면 빈줄이 없다.
array3=()
for i in "${array2[@]}"; do
    [[ -n "$i" ]] && array3+=("$i")
done

# Writing the remaining elements from array2 to result.txt
for line in "${array3[@]}"; do
    echo "$line" >> result.txt
done
```

### 덧.  IFS= read -r line 가 뭘까

- IFS 는 텍스트 구분자(delimiter)
- IFS= 빈칸으로 정의하면 텍스트를 끊지 않고 한 줄 전체를 하나의 값으로 고려
- read는 파일 또는 사용자 입력 내용을 읽는 기능
- -r 은 `\` 가 있는 경우 문자그대로 인식 (줄바꿈 기능으로 동작하지 않음)
- line은 read가 읽은 line 을 저장하기 위한 변수
