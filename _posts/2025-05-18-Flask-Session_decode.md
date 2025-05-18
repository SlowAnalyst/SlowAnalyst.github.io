---
layout: post
title: "Flask session decode"
summary: "러스트 에러 정보 내부 콜스택 추가하기"
author: eveheeero
date: '2025-05-18 22:25:00 +0900'
category: ['etc', 'learning']
tags: vulnerability
thumbnail: /assets/img/posts/2025-05-18-0.png
keywords: ctf, session, vulnerability
usemathjax: false
permalink: /blog/flask-session-decode/
---


## 배경

BYUCTF 2025 웹해킹 문제 해결 시도중 찾아보았습니다.

## 요약

- `jwt.io`를 이용해 복호화 가능
- `flask-unsign`을 `pip`에서 받아 복호화 가능, 단 암호화는 브루트포스를 이용한 키가 있어야 한다.

## 문제 상황

세션 내부 값을 원하는대로 조작하려 하였습니다. 브루트포스 일부 시도중 포기하였습니다.

## 분석 과정

![jwt.io 복호화 이미지](/assets/img/posts/2025-05-18-0.png){: style="max-width: 100%; height: auto;"}

세션으로 JWT가 들어와서 유명한 JWT.io를 이용해 디코드를 하였습니다. 값 확인엔 성공했지만 원하는대로 수정이 불가능했습니다.

![flask-unsign을 이용한 복호화 이미지](/assets/img/posts/2025-05-18-1.png){: style="max-width: 100%; height: auto;"}

flask-unsign이란 툴로 복호화가 되는지 진행해보았습니다.

## 알아낸 내용

- `flask-unsign`을 통해 세션 값 재생성 가능. (단 암호를 알 경우만)

## 참조

- [jwt.io](https://jwt.io/)
- [Flask-Unsign](https://github.com/Paradoxis/Flask-Unsign)
