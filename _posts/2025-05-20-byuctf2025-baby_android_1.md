---
layout: post
title: "[byuctf2025] Baby Android 1"
summary: "DEX 압축 된 안드로이드 리버싱"
author: eveheeero
date: '2025-05-20 15:18:00 +0900'
category: ['reversing', 'learning']
tags: ctf
thumbnail: /assets/img/posts/2025-05-20-4.png
keywords: ctf, reversing, android
usemathjax: false
permalink: /blog/flask-session-decode/
---


## 배경

BYUCTF 2025 리버싱 `Baby Android 1` 문제 풀이입니다.

## 요약

- 디컴파일은 `압축 해제 된` dex파일을 기드라에 넣어서 가능, 혹은 `apktool`로 디컴파일 후 `smail2java`등의 툴을 이용해 가능

## 문제 상황

![apk 파일 내부 구조](/assets/img/posts/2025-05-20-0.png){: style="max-width: 100%; height: auto;"}

주어진 apk파일 내부 구조는 위와 같았습니다.

`점수 - 241점, 푼 팀 - 191/894팀`

## 분석 과정

`classes.dex`나 `classes2.dex`의 파일 크기가 너무 커 `안드로이드 기본 클래스` 및 `코틀린 기본 클래스`라고 생각하고 `classes3.dex`를 보았습니다.

![기드라로 연 내용](/assets/img/posts/2025-05-20-1.png){: style="max-width: 100%; height: auto;"}

기드라로 디컴파일 후 `export` 혹은 `classes`, `namespace`를 통해 사용자가 작성한 코드를 볼 수 있습니다.

![Main 컴포넌트 동작](/assets/img/posts/2025-05-20-2.png){: style="max-width: 100%; height: auto;"}

![cleanup 코드](/assets/img/posts/2025-05-20-3.png){: style="max-width: 100%; height: auto;"}

각각 메인 동작 코드와 `cleanup` 코드입니다.

`cleanup`코드에서 내부 텍스트를 전부 지우는것으로 보이는데, 어떤 텍스트인지는 보이지 않습니다. xml파일에 기본으로 있을 것으로 예상됩니다.

`resources.arsc` 파일 복호화가 필요해 `apktool`을 사용해 다시 디컴파일한 뒤 `grep -rnw . -e "flagPart1"`을 통해 컴포넌트 xml을 찾았습니다.

![grep 실행 내용](/assets/img/posts/2025-05-20-4.png){: style="max-width: 100%; height: auto;"}

![xml 파일 내부](/assets/img/posts/2025-05-20-5.png){: style="max-width: 100%; height: auto;"}

xml 파일을 열고 순서대로 입력해보았으나 틀렸습니다. 레이아웃을 이용해 정렬해야 할 듯합니다.

![결과](/assets/img/posts/2025-05-20-6.png){: style="max-width: 100%; height: auto;"}

여러가지 온라인 툴을 찾아보았으나 없었고, 계산하기 귀찮아 안드로이드 스튜디오로 직접 그려보았습니다.

## 작성자의 글

- 기드라는 압축파일 내부 dex파일을 인식하지 못했습니다.
- `Baby Android 2`문제보다 푼 사람이 적어서 의외였습니다. 안드로이드 스튜디오 설치하기가 귀찮았나봅니다.

## 참조

- .
