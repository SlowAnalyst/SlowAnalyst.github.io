---
layout: post
title: "[byuctf2025] Baby Android 2"
summary: "동적 라이브러리 형태 안드로이드 리버싱"
author: eveheeero
date: '2025-05-20 15:31:00 +0900'
category: ['reversing', 'learning']
tags: ctf
thumbnail: /assets/img/posts/2025-05-20-9.png
keywords: ctf, reversing, android
usemathjax: false
permalink: /blog/byuctf2025-baby-android2/
---


## 배경

BYUCTF 2025 리버싱 `Baby Android 2` 문제 풀이입니다.

## 요약

- 단순 연산 문제입니다.

## 문제 상황

![apk 파일 내부 구조](/assets/img/posts/2025-05-20-7.png){: style="max-width: 100%; height: auto;"}

주어진 apk파일 내부 구조는 위와 같았습니다.

`점수 - 61점, 푼 팀 - 248/894팀`

## 분석 과정

![기드라로 연 내용](/assets/img/posts/2025-05-20-8.png){: style="max-width: 100%; height: auto;"}

기드라로 연 후 export를 보니 다음과 같은 함수가 있었습니다.

파이썬으로 만들어 풀었습니다.

![파이썬 결과 이미지](/assets/img/posts/2025-05-20-9.png){: style="max-width: 100%; height: auto;"}

## 작성자의 글

- 전 문제와 달리 이번 문제는 dex 복호화가 필요 없어서 푼 사람이 많았나봅니다.

## 참조

- .
