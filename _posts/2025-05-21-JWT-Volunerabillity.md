---
layout: post
title: "JWT Base64 URL Encoded 취약점"
summary: "JWT Base64 URL Encoded 취약점"
author: wHoIsDReAmer
date: '2025-05-21 15:27:00 +0900'
category: ['etc', 'learning']
tags: cve
thumbnail: /assets/img/posts/2025-05-21-0.png
keywords: ctf, cve, vulnerability
usemathjax: false
permalink: /blog/JWT-Volunerabillity/
---

## 배경
BYUCTF 2025 웹해킹 문제 해결 시도중 찾아보았습니다.

## 요약

- `JWT` 토큰 값을 비교하는 상황에서 취약점 확인

## 문제 상황

우선 문제 배경은 다음과 같습니다.

1. 특정 토큰 값이 노출되어 있는 상황이고, 해당 토큰 값에는 `admin` 필드가 `true`인 상황

2. 노출된 토큰 값이 아니고, 페이로드 내에 `admin` 필드가 `true`인 토큰 값을 출력하는 문제

다만, 현실적으로 토큰 발급 과정에서 `admin` 필드가 `true`인 토큰 값을 발급하는 것은 불가능했었습니다.

## 분석 과정

우선 해당 서비스의 백엔드는 파이썬입니다.

파이썬의 `base64` 모듈은 `urlsafe_b64decode`라는 함수를 제공합니다. 문제는 이 함수가 디코딩 시 확장성을 위해 `url based base64`든 `base64`든 모두 디코딩 시 같은 결과를 출력한다는 점입니다. `PyJWT`는 이를 이용하여 토큰 값을 디코딩합니다.

```python
import base64

print(base64.urlsafe_b64decode("Twva/sZwHPkmJbLVmvqe1g=="))
print(base64.urlsafe_b64decode("Twva_sZwHPkmJbLVmvqe1g=="))
```

위 코드를 실행해보면 똑같은 값이 출력됩니다.

이를 이용하여 토큰 값의 `-, /` 값을 base64 url safe 인코딩 값으로 변환하여 토큰 값에 지장없이 토큰을 조작할 수 있습니다.

## 알아낸 내용
- `PyJWT`는 `base64` 모듈의 `urlsafe_b64decode` 함수를 이용하여 토큰 값을 디코딩합니다.
- `base64` 모듈의 `urlsafe_b64decode` 함수는 `url based base64`든 `base64`든 모두 디코딩 시 같은 결과를 출력합니다.
- 이를 이용하여 토큰 값의 `-, /` 값을 base64 url safe 인코딩 값으로 변환하여 토큰 값에 지장없이 토큰을 조작할 수 있습니다.

## 참조

- [base64-vs-base64-url-safe-그리고-예제](https://velog.io/@dohaeng0/base64-vs-base64-url-safe-%EA%B7%B8%EB%A6%AC%EA%B3%A0-%EC%98%88%EC%A0%9C)
