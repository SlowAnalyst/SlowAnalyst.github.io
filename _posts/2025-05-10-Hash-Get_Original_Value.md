---
layout: post
title: "해시 원본 값 구하기"
summary: "해시 결과값을 통해 원본 값을 구하는 법"
author: eveheeero
date: '2025-05-10 20:48:00 +0900'
category: ['etc', 'ctf', 'learning']
tags: ctf
thumbnail: /assets/img/posts/default.png
keywords: ctf, hash, md5
usemathjax: false
permalink: /blog/hash-original-value/
---


## 배경

DevSecOps 2025 Password Cracking 문제를 푸는 중 해시 값의 원본 값을 구하는 문제가 있어 요약하였습니다.

## 요약

- `Hashcat`을 이용해 여러 해시 방법, 암호화 방법에 대한 브루트포싱을 간단하게 할 수 있으며, Gpu 및 Charset, Masking 등의 여러 도구가 들어 있어 브루트포싱할때 유용함
- Salt가 없는 `md5`등의 해시 기법은 인터넷의 `데이터베이스 기반 탐색` 및 `레인보우 테이블`을 살펴보자.

## 문제 상황

주어진 해시에 대한 평문 값을 구하시오.

## 분석 과정 (뻘짓 과정)

1. 간단하게 `md5` attck을 키워드로 검색하여 `충돌 공격` 및 `충돌 공격 기법들`에 대해 알아보았습니다.
   - 해시 충돌에 대해 알게 되었지만, 문제에 도움되지 않고 정말 많은 시간이 흘렀습니다.
2. `충돌 공격`은 원문을 찾는 데 도움이 되지 않는다는 것을 확인하고 `데이터베이스 기반 탐색`사이트를 찾아다니며 해당 해시를 넣어보았습니다.
3. `Hashcat`을 이용해 브루트포싱을 시도하였습니다.
   - `hashcat -m 0 -a 3 hash.txt -1 ?u?d_ "punk_{?1?1?1?1?1?1?1}" -O` 을 인자로, 기본 플래그 틀을 이용해 마스킹한 후 진행하였으나, 잘못된 선택이었습니다.
4. 동시에 `레인보우 테이블`을 통해 해당 해시를 검색하였습니다.

## 알아낸 내용

- 암호문이나 해시 값이 주어질 경우 `레인보우 테이블`을 먼저 찾아보면 좋다.
- `Hashcat`을 이용해 간단하게 브루트포싱이 가능하다.
- <https://github.com/corkami/collisions> - 해시 충돌 관련 문서
  - 위 내용을 이용해 같은 해시를 가진 서로 다른 파일을 만들 수 있다.
  - `exe`파일의 경우, 같은 해시를 가졌지만 전혀 다른 작동을 하는 프로그램을 만들 수 있다.
