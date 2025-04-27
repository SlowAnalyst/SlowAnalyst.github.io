---
layout: post
title: "유니티 게임 파일 변조법"
summary: "정적 수정을 통한 유니티 게임 크래킹"
author: eveheeero
date: '2025-04-27 12:35:00 +0900'
category: ['reversing', 'learning']
tags: ctf
thumbnail: /assets/img/posts/2025-04-27-0.png
keywords: ctf, unity, reversing, cracking, hacking
usemathjax: false
permalink: /blog/unity-binary-modification/
---


## 배경

UMassCTF 2025 Bullet Dodge 문제를 푸는 중 `Unity`게임을 크랙하는 문제가 있어 간단하게 요약하였습니다.

## 요약

- `dnSpy` 혹은 `iLSpy`를 이용해 개발자가 입력한 코드 수정 가능

## 문제 상황

![어려운 총알 피하기 게임 화면](/assets/img/posts/2025-04-27-0.png)

게임 클리어를 위해서는 `1000000`점을 얻어야 하지만 총알이 너무 빠르고 점수 필요 량이 너무 많아 클리어하기 힘들다.

따라서 게임오버가 되지 않게 하고 점수를 많이 받도록 해야 한다.

`점수 - 310점, 푼 팀 - 70/475팀`

## 분석 과정

게임 파일을 수정하기 위해선 `dll`파일이 많이 들어있는 폴더를 찾아야 한다.

이번 경우는 `/Data/Managed/`에 들어있었지만 다른 폴더에 들어가는 경우가 있을 수 있다.

C#에서 주로 사용하는 `System.Core.dll` 파일로 검색을 하면 쉽게 찾을 수 있다.

![dll이 들어간 폴더](/assets/img/posts/2025-04-27-1.png)

이후 게임 파일을 찾아야 한다. 일반적으로 `Assembly-CSharp.dll` 파일이라고 하지만 다른 파일인 경우도 있다.

찾으면 해당 파일을 `dnSpy`로 열어서 코드를 살펴본다.

![dnSpy로 연 내용](/assets/img/posts/2025-04-27-2.png)

해당 내용에서 이름이 적힌 내용은 일반적으로 라이브러리를 사용한 부분이다. 디버깅 정보가 없어 _로 표시된 부분을 살펴봐야 한다.

![수정하기 원하는 코드](/assets/img/posts/2025-04-27-3.png)

주로 클래스들의 `Update`함수 로직을 살펴보면서, 어떤 값을 사용하는지 확인해서

![수정한 코드드](/assets/img/posts/2025-04-27-4.png)

`우클릭 -> Edit Class`를 통해 코드를 수정한 뒤

![dnSpy 저장화면](/assets/img/posts/2025-04-27-5.png)

`파일 -> Save...`를 통해 저장 후 게임 실행시 수정된 코드가 적용된다.

![게임 클리어 화면면](/assets/img/posts/2025-04-27-6.png)

잘 적용되어 죽지 않고 점수도 빠르게 올라간다.

## 알아낸 내용

- .
