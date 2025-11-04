---
layout: post
title: "Android 패킷 캡처중 tls 관련 오류 해결"
summary: "Android 패킷 캡처중 tls 관련 오류 해결"
author: eveheeero
date: '2025-11-04 21:50:00 +0900'
category: ['reversing', 'learning']
tags: reversing
thumbnail: /assets/img/posts/2025-11-04-3.png
keywords: reversing
usemathjax: false
permalink: /blog/android-packet-inspect-tls/
---

## 배경

안드로이드 앱의 패킷을 `PCAPdroid`를 통해 캡처하려 했으나, 캡처를 실행했더니 앱 내부 인터넷 통신 오류가 발생해 작성하게 되었습니다.

## 요약

- 앱의 네트워크 설정에 따라 `유저 설치 인증서`가 비허용되면서 `tls`통신이 실패함
- `apktool`을 통해 `targetSdkVersion`을 `23`으로 변경하거나
- `apktool`을 통해 안드로이드 매니페스트 내부 `<application>`에 `<application android:networkSecurityConfig="@xml/network_security_config"...`태그 추가하고 `res/xml/network_security_config.xml`경로에 일부 설정하면 됨

## 문제 상황

`PCAPdroid`를 이용해 안드로이드 앱의 패킷을 캡쳐하려 했으나, 패킷 캡처를 켜니 앱이 정상 동작하지 않음

![_](/assets/img/posts/2025-11-04-0.png){: style="max-width: 100%; height: auto;"}

| 오류 메세지가 이번에는 `tlsv1 alert access denied`라고 떴지만, 메세지는 매번 달라졌었습니다.

## 분석 과정

[이 글이 많이 도움되었습니다.](https://velog.io/@doobykihun/%EC%88%AD%EC%8B%A4%EB%8C%80-%EB%AA%A8%EB%B0%94%EC%9D%BC%ED%95%99%EC%83%9D%EC%A6%9D-%EC%95%B1-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EB%B6%84%EC%84%9D)

위 블로그를 통해 어떤 문제인지 알게 되었으며, [관련 설명이 담긴 공식 사이트를 찾을 수 있었습니다.](https://developer.android.com/privacy-and-security/security-config?hl=ko)

![_](/assets/img/posts/2025-11-04-1.png){: style="max-width: 100%; height: auto;"}

홈페이지를 살펴보니 안드로이드 23버전까지는 사용자가 추가한 인증서를 신뢰하고, 그 이후부터는 신뢰하지 않는다고 합니다.

![_](/assets/img/posts/2025-11-04-2.png){: style="max-width: 100%; height: auto;"}

위 블로그 설명처럼 `apktool`을 이용해 디컴파일 후 `apktool.yml`파일에서 `targetSdkVersion`을 `23`으로 조정한 후 컴파일 및 [서명을 해 설치](/blog/android-recompile-signing/)하면 정상적으로 `tls`패킷 확인이 가능하다.

![_](/assets/img/posts/2025-11-04-3.png){: style="max-width: 100%; height: auto;"}

| `minSdkVersion`이 `23`보다 클 경우 `23`으로 조정해주시면 됩니다.
| 버전을 `23`보다 낮게 설정할 경우 [문제](https://developer.android.com/google/play/requirements/target-sdk?hl=ko)가 생길 수 있습니다.
![_](/assets/img/posts/2025-11-04-4.png){: style="max-width: 100%; height: auto;"}

---

디폴트 설정이 `유저 인증서 신뢰 안함`이라면 설정을 변경할 수도 있다고 생각했습니다.

네트워킹 보안 설정 페이지 아래를 살펴보니 [관련된 설정이 보였습니다.](https://developer.android.com/privacy-and-security/security-config?hl=ko#network-security-config)

![_](/assets/img/posts/2025-11-04-5.png){: style="max-width: 100%; height: auto;"}

확인해보니 디폴트 설정은 다음과 같답니다. 즉 `<certificates src="user" />`을 추가하는 것으로 유저 인증서를 사용할 수 있는것처럼 보입니다.

![_](/assets/img/posts/2025-11-04-6.png){: style="max-width: 100%; height: auto;"}

`AndroidManifest.xml`파일 내부 여러군데에 위의 `certificates`설정을 넣어봤는데 잘 안됩니다. 너무 대충 읽은듯합니다.

![_](/assets/img/posts/2025-11-04-7.png){: style="max-width: 100%; height: auto;"}

확인해보니 `AndroidManifest.xml`에는 설정을 `ON`해줘야 하고, 설정파일은 또 따로 만들어야 하는 듯 합니다.

![_](/assets/img/posts/2025-11-04-8.png){: style="max-width: 100%; height: auto;"}

이곳에 넣어주면 되는 듯 합니다.

![_](/assets/img/posts/2025-11-04-9.png){: style="max-width: 100%; height: auto;"}

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

해당 파일을 기본 설정대로 두고 빌드하니 제대로 빌드됩니다.

패킷도 제대로 보입니다.

![_](/assets/img/posts/2025-11-04-10.png){: style="max-width: 100%; height: auto;"}


## 작성자의 글

- 안드로이드에서 `ssl` 복호화에 사용되는 키를 저장하는 옵션인 `sslkeylog`를 찾을 수 없었습니다.
- 프록시를 통해 패킷을 살펴보는건 좀 귀찮았습니다. 나중에 해봐야겠습니다.
- `apktool`을 통해 앱을 디컴파일하고 컴파일하니 앱 시그니처가 달라져서 문제를 겪었습니다.
- 참고한 블로그에 잘 적혀있었지만, sdk버전 수정중 오류가 발생할 경우를 대비해 실습해보았습니다.

## 참조

- [doobykihun님의 velog 블로그](https://velog.io/@doobykihun/%EC%88%AD%EC%8B%A4%EB%8C%80-%EB%AA%A8%EB%B0%94%EC%9D%BC%ED%95%99%EC%83%9D%EC%A6%9D-%EC%95%B1-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EB%B6%84%EC%84%9D)
- [네트워킹 보안 설정](https://developer.android.com/privacy-and-security/security-config?hl=ko)
- [sdk 버전 제약사항](https://developer.android.com/google/play/requirements/target-sdk?hl=ko)
