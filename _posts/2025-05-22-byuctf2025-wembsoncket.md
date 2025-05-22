---
layout: post
title: "[byuctf2025] Baby Android 2"
summary: "동적 라이브러리 형태 안드로이드 리버싱"
author: eveheeero
date: '2025-05-20 15:31:00 +0900'
category: ['reversing', 'learning']
tags: ctf
thumbnail: /assets/img/posts/2025-05-22-0.png
keywords: ctf, reversing, android
usemathjax: false
permalink: /blog/byuctf2025-wembsocket/
---


## 배경

BYUCTF 2025 웹해킹 `Wembsocket` 문제 풀이입니다.

## 요약

- Websocket 연결 시, 서버측에서 어디에서 소켓이 연결되었는지 검사하지 않을 때 하이재킹이 발생할 수 있음.

## 문제 상황

![공격 대상 사이트](/assets/img/posts/2025-05-22-0.png){: style="max-width: 100%; height: auto;"}

다음과 같은 챗봇 사이트가 존재했으며

![웹서버 소스코드](/assets/img/posts/2025-05-22-1.png){: style="max-width: 100%; height: auto;"}

어드민이 `/getFlag` 명령어 전달 시 플래그를 출력하며

`http://...`로 시작하는 주소를 입력할 시

![내부 로직](/assets/img/posts/2025-05-22-2.png){: style="max-width: 100%; height: auto;"}

`puppeteer`라이브러리를 통해 해당 사이트에 접속하여 사이트가 연결되는지 확인하는 사이트였습니다.

`점수 - 446점, 푼 팀 - 88/894팀`

## 분석 과정

아래는 문제풀이를 위한 시도였습니다.

1. 세션을 공격하거나 프로그램 내부에서 사용되고 있는 `puppeteer`취약점을 이용해 공격하는것으로 접근해보았습니다.
2. 세션 공격은 `verify`에서 검증에 실패했습니다.
3. 웹서버에서 사용하던 `puppeteer`에는 특별한 취약점이 존재하지 않는것같았습니다.

---

아래는 문제 해답입니다.

![문제 해답](/assets/img/posts/2025-05-22-3.png){: style="max-width: 100%; height: auto;"}

```text
1. 사이트 접속시 챗봇에 연결해 `/getFlag`를 전송해 응답을 원하는 곳으로 전송하는 스크립트를 작성해둡니다. (A)
2. A 사이트를 외부에서 접근 가능하게 노출시켜둡니다.
3. 웹서버 챗봇에 A 사이트에 접근하라고 지시합니다.
4. 웹서버 챗봇이 A 사이트에 접근합니다.
5. 웹서버 챗봇이 A 사이트 스크립트를 실행해, `wembsoncket.chal.cyberjousting.com`사이트의 쿠키를 가지고 챗봇에 연결합니다. (쿠키는 위 소스코드 내부에 설정되어 있습니다.)
6. 어드민의 쿠키를 가지고 있는 챗봇이 `/getFlag`를 실행하여 플래그를 반환합니다.
7. 플래그가 웹훅으로 전송됩니다.
```

## 작성자의 글

- 해당 문제는 대회 당일 해결하지 못해 writeup을 찾아보던 중 풀이를 발견하여 정리하게 되었습니다.

## 참조

- [CTF 공식 문제풀이](https://github.com/BYU-CSA/BYUCTF-2025/blob/main/web/wembsoncket/README.md)
