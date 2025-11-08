---
layout: post
title: "Example Of HLS Packets"
summary: ""
author: david232818
date: '2025-11-09 15:47:00 +0900'
category: ['learning']
tags: HLS
thumbnail: /assets/img/posts/default.png
keywords: HLS
usemathjax: false
permalink: /blog/Example-Of-HLS-Packets
---


Contents
--------
1. Introduction
2. Playlist and EXT-X-STREAM-INF Tag
3. EXT-X-KEY and EXT-X-MEDIA-SEQUENCE Tag
0. References



## Introduction

비디오 스트리밍 사이트 중에는 m3u8을 확장자로 하는 파일을 사용하여
서비스를 제공하는 곳이 있다. 이는 HTTP Live Streaming이라고 불리는
것으로 멀티미디어 데이터 스트림을 전송하는데 사용되는 프로토콜이다[1].

<br>

본 글에서는 HLS 프로토콜 관련 패킷을 캡처했을 때 볼 수 있는 것들이
어떤 의미를 갖고 있는지 정리할 것이다.

## Playlist and EXT-X-STREAM-INF Tag

한 웹 사이트가 HLS를 사용하여 스트리밍을 제공할 때 그 패킷을 Burpsuite
등으로 캡처해보면 m3u8을 확장자로 하는 파일에 접근하는 것이 있을 수
있다. 그리고 그 파일의 이름은 전형적으로 playlist.m3u8이다. 이
파일에는 재생목록이 기술되어 있을 수도 있고, 재생목록의 위치가
기술되어 있을 수도 있다. 이때 전자를 Media Playlist, 후자를 Master
Playlist라고 부른다. 이러한 Master Playlist에서 목록의 위치를
상대적으로 기술할 때 기준점은 Playlist를 가지고 있는 경로가 된다[1].

<br>

Playlist 파일에서 EXT-X-STREAM-INF 태그는 비디오나 오디오 스트림의
정보를 기술하고, 그 다음 줄에 재생목록의 위치를 표시한다[1].

<br>

Playlist를 요청하는 패킷의 예시는 다음과 같다. 

    GET /a/playlist.m3u8 HTTP/2
    Host: b
    Sec-Ch-Ua-Platform: "Linux"
    Accept-Language: en-US,en;q=0.9
    Sec-Ch-Ua: "Chromium";v="139", "Not;A=Brand";v="99"
    User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36
    Sec-Ch-Ua-Mobile: ?0
    Accept: */*
    Origin: c
    Sec-Fetch-Site: cross-site
    Sec-Fetch-Mode: cors
    Sec-Fetch-Dest: empty
    Referer: c
    Accept-Encoding: gzip, deflate, br
    Priority: u=1, i

그리고 위 패킷의 응답 예시는 다음과 같다.

    HTTP/2 200 OK
    Date: Sun, 05 Oct 2025 18:03:36 GMT
    Content-Type: application/vnd.apple.mpegurl
    Server: BunnyCDN-LA1-999
    Cdn-Pullzone: 3184689
    Cdn-Uid: 7fe4b255-0146-44d1-bb9d-d592eef4ec43
    Cdn-Requestcountrycode: KR
    Vary: Accept-Encoding
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Headers: Server, x-goog-meta-frames, Content-Length, Content-Type, Range, X-Requested-With, If-Modified-Since, If-None-Match
    Access-Control-Expose-Headers: Server, x-goog-meta-frames, Content-Length, Content-Type, Range, X-Requested-With, If-Modified-Since, If-None-Match
    Cache-Control: public, max-age=30
    Stream-Mw-Version: 1.0.5
    Cdn-Bess-Version: 1.31.0
    Cdn-Proxyver: 1.37
    Cdn-Requestpullsuccess: True
    Cdn-Requestpullcode: 200
    Cdn-Cachedat: 10/05/2025 18:03:36
    Cdn-Edgestorageid: 953
    Cdn-Requestid: 38e116eab7890ba572736fdd54e9679f
    Cdn-Cache: EXPIRED
    Cdn-Status: 200
    Cdn-Requesttime: 0
    
    #EXTM3U
    #EXT-X-VERSION:3
    
    #EXT-X-STREAM-INF:BANDWIDTH=2446156,CODECS="avc1.64001f,mp4a.40.2",RESOLUTION=854x480
    480p/video.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=4738257,CODECS="avc1.64001f,mp4a.40.2",RESOLUTION=1280x720
    720p/video.m3u8

앞서 언급한 것을 위 패킷에 적용해보면 재생목록의 위치는
/a/480p/video.m3u8과 /a/720p/video.m3u8가 된다.

## EXT-X-KEY and EXT-X-MEDIA-SEQUENCE Tag

HLS는 스트림에 대한 암호화를 지원한다. 암호화 관련 정보는 EXT-X-KEY
태그에 기술된다. 암호화 방법은 METHOD 속성에 기술되고, 이를 표로
요약하면 다음과 같다[1]. 

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Method</th>
<th scope="col" class="org-left">Description</th>
</tr>
</thead>
<tbody>
<tr>
<td class="org-left">None</td>
<td class="org-left">Not encrypted.</td>
</tr>

<tr>
<td class="org-left">AES-128</td>
<td class="org-left">Media Segments are completely encrypted using the AES-128 with a 128-bit key, Cipher Block Chaining (CBC) Mode, and Public-Key Cryptography Standards #7 (PKCS7).</td>
</tr>

<tr>
<td class="org-left">SAMPLE-AES</td>
<td class="org-left">Media Segements contain media samples that are encrypted using the AES-128.</td>
</tr>
</tbody>
</table>

이때 암호화 키는 URI 속성에 그 위치가 기술되고, 초기 벡터 (Initial
Vector)는 IV 속성에 기술된다. 만약 IV 속성이 없다면, 이는 Meida 
Sequence Number가 각 세그먼트에 대한 IV로 사용된다는 것을
의미한다. Media Sequence Number는 EXT-X-MEDIA-SEQUENCE 태그에 시작
번호가 기술되며 1씩 증가한다[1].

<br>

예를 들어, /a/720p/video.m3u8를 요청했는데 다음과 같은 응답이 왔다고
생각해보자.

    HTTP/2 200 OK
    Date: Mon, 06 Oct 2025 06:52:21 GMT
    Content-Type: application/vnd.apple.mpegurl
    Server: BunnyCDN-LA1-996
    Cdn-Pullzone: 3184689
    Cdn-Uid: 7fe4b255-0146-44d1-bb9d-d592eef4ec43
    Cdn-Requestcountrycode: KR
    Vary: Accept-Encoding
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Headers: Server, x-goog-meta-frames, Content-Length, Content-Type, Range, X-Requested-With, If-Modified-Since, If-None-Match
    Access-Control-Expose-Headers: Server, x-goog-meta-frames, Content-Length, Content-Type, Range, X-Requested-With, If-Modified-Since, If-None-Match
    Cache-Control: public, max-age=30
    Stream-Mw-Version: 1.0.5
    Cdn-Bess-Version: 1.31.0
    Cdn-Proxyver: 1.37
    Cdn-Requestpullsuccess: True
    Cdn-Requestpullcode: 200
    Cdn-Cachedat: 10/06/2025 06:52:21
    Cdn-Edgestorageid: 1112
    Cdn-Requestid: 747f0c782f763ada1b1ae659db53f7a5
    Cdn-Cache: EXPIRED
    Cdn-Status: 200
    Cdn-Requesttime: 0
    
    #EXTM3U
    #EXT-X-KEY:METHOD=AES-128,URI="/x/y/z"
    #EXT-X-VERSION:3
    #EXT-X-TARGETDURATION:4
    #EXT-X-MEDIA-SEQUENCE:0
    #EXT-X-PLAYLIST-TYPE:VOD
    #EXTINF:4.004000,
    video0.dts
    #EXTINF:4.004011,
    video1.dts
    #EXTINF:4.004000,
    video2.dts
    #EXTINF:4.004000,
    video3.dts
    #EXTINF:4.004011,
    video4.dts
    ...

위 패킷을 보면 EXT-X-KEY 태그의 METHOD 속성으로부터 AES-128 CBC 모드로
암호화 되어있음을, URI 속성으로부터 키의 위치가 /x/y/z임을 알 수
있다. 그리고 IV 속성이 없고 EXT-X-MEDIA-SEQUENCE 태그에 기술된 번호가
0이므로 0부터 시작하여 세그먼트마다 1씩 증가한 번호가 초기 벡터로
사용됨을 알 수 있다. 즉, 위 패킷에서 video0.dts는 0이 초기 벡터가
되고, video1.dts는 1이 초기 벡터가 되는 것이다.

## References

1.  R. Pantos and W. May, "HTTP Live Streaming," IETF, Rep. RFC8216.,
    Aug. 2017. Accessed: Oct. 06, 2025. [Online]. Available:
    <https://datatracker.ietf.org/doc/html/rfc8216>

