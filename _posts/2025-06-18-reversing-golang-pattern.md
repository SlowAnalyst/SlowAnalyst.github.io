---
layout: post
title: "리버싱 golang 패턴 정리"
summary: "리버싱 golang 고루틴 패턴"
author: eveheeero
date: '2025-06-18 04:15:00 +0900'
category: ['reversing', 'learning']
tags: reversing
thumbnail: /assets/img/posts/default.png
keywords: reversing
usemathjax: false
permalink: /blog/reversing-golang-pattern/
---


## 배경

golang 리버싱 중 발견된 패턴에 대해 정리합니다.

## 요약

- `ghidra`는 비슷하게 출력되지만 일부 깨지는 현상이 발생합니다.
- `ida`가 `golang` 리버싱에 적합합니다.

## 문제 상황

CTF 문제를 풀던 중 고랭으로 나온 프로그램을 디컴파일하게 되었습니다.

## 분석 과정

- 고랭 객체 생성시 변수 추적이 잘 되지 않음.

```c++
/* 채널 생성시 runtime_makechan 뒤에 오는 할당식의 lvar에 채널이 들어갑니다. in, out 구분 없음 */
runtime_makechan(&RTYPE_chan_int, 0LL, v0);
v49 = v2;
runtime_makechan(&RTYPE_chan_uint8, 0LL, v3);
v48 = v4;
runtime_makechan(&RTYPE_chan_uint8, 0LL, v5);
v47 = v6;

/* 고루틴 호출 패턴 */
runtime_newobject(&stru_4B5C60, 0LL);
v24->fn = main_main_func2; // 실제 실행 함수
if ( *&runtime_writeBarrier.enabled ) // GC 설정 관련 내용, 무시
{
  runtime_gcWriteBarrier2();
  v25 = v48;
  *v27 = v48;
  v26 = v46;
  v27[1] = v46;
}
else
{
  v25 = v48;
  v26 = v46;
}
v24[1].fn = v25; // 고루틴 인자가 fn으로 들어감
v24[2].fn = v26; // 두번째 인자
runtime_newproc(v24);

/* 고루틴 인자 받는 패턴 */
// 어셈으로 볼 경우 스택 8, 16등에 넣는것이 보입니다. 8, 16, ... 순서대로 인자값으로 추정
v1 = *(v0 + 16);
c = v1;
v2 = *(v0 + 8);
v9 = v2;
```

## 작성자의 글

- 디컴파일된 코드를 편집기로 옮겨서 불필요한 내용을 삭제하며 보는게 좋을듯합니다.

## 참조

- .
