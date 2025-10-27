---
layout: post
title: "Android 앱 리컴파일 이후 서명"
summary: "Android 앱 서명법"
author: eveheeero
date: '2025-10-27 23:42:00 +0900'
category: ['reversing', 'learning']
tags: reversing
thumbnail: /assets/img/posts/default.png
keywords: reversing
usemathjax: false
permalink: /blog/android-recompile-signing/
---

## 배경

안드로이드 `apk` 변조 및 실행하려 여러 방법을 시도한 김에 작성하였습니다.

## 요약

`apk`에 문제 있는지 확인은 `apksigner verify --print-certs -verbose --in your.apk`를 통해 가능

키 파일 생성은 `keytool -genkey -v -keystore your.keystore(혹은 jks) -alias 별명 -keyalg RSA -keysize 2048`을 통해 가능

`apk` 서명은 `apksigner sign --ks ... your.apk`를 통해 가능

## 문제 상황

안드로이드 앱 변조 후 `apktool`을 이용해 패키징을 하였으나 설치가 되지 않음

메세지가 뜨지 않아 여러가지 찾아봤는데, 서명 관련된 문제였습니다.

## 분석 과정

다음처럼 하면 됩니다.

![_](/assets/img/posts/2025-10-27-0.png){: style="max-width: 100%; height: auto;"}

리소스 갯수로 인해 `apktool`에서 빌드가 안되면 `apktool` 버전문제입니다. 최신 버전 이용하시면 웬만하면 잘 됩니다.

`jarsigner`로 서명할경우 `xapk`에서 동작하지 않고 `apksigner verify`에서 워닝뱉습니다.

`xapk`가 서명 문제로 설치되지 않은 경우, `xapk`를 압축 해제 후, 내부 모든 `apk`를 서명하면 됩니다. `apktool`로 디컴파일이 안되는 `apk`는 그대로 재서명 하면 됩니다. 이후 `zip`으로 압축 후 확장자 변경하시면 됩니다.

## 작성자의 글

- .

## 참조

- [cgy12306 Tistory](https://cgy12306.tistory.com/394)
- [apk signer](https://developer.android.com/tools/apksigner?hl=ko)
