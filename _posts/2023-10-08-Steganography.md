---
layout: post
title: "Deep dive of Steganography"
summary: "Summarize steganography and analyze real-world examples."
author: bradypus404
date: '2023-10-08 05:20:01 +0900'
category: ['malware']
tags: Steganography, Malware
thumbnail: /assets/img/posts/Steganography.png
keywords: malware, reversing
usemathjax: false
permalink: /blog/Malware/Steganography
---

# 스테가노그래피

## 개념

- 메시지를 숨겨서 비밀문서를 작성

## 디지털 스테가노그래피

- 텍스트 파일, 그림 파일, 오디오 파일, 동영상 파일등으로 메시지를 숨김
- 주로 그림 파일을 대상으로 함

### 오디오 파일 대상

- 인간의 청각 영역을 벗어난 주파수 대역에 노이즈 형태로 메시지를 삽입

### 이미지 파일 대상

1. Spatial domain : 픽셀을 이용
2. Transform(or Frequency) domain : Transform, Frequency에 정보를 숨김
3. Distortion : 이미지 파일을 변형시켜 정보를 숨김, 해독자는 원본 이미지 파일을 가지고 있어야함
4. Masking and filtering : 이미지의 특정 부분의 밝기나 휘도를 변형시켜 정보를 숨김

## Malware에서 스테가노그래피 사용

### 목적

- 안티 바이러스 SW의 탐지를 피하기 위함
- 장점 : 기술 구현 쉽고 탐지 어려움

### 같이 사용되는 공격

- 말버타이징(Malvertising) 공격
- Exploit-Kit 공격

### Malware 형태

1. Payload라 불리는 최종 악성코드를 설치하는 광정을 숨김
2. 악성코드를 실행하는데 필요한 Configuration 정보를 숨김
3. 해커가 운영하는 C&C 서버에서, 감염된 시스템에 전달하는 명령어를 숨김
4. 감염된 시스템에서 동작하는 Malware가 수집한 정보를, 외부 C&C서버로 보낼때, 정보를숨김

### 실제 사례

- ‘Worok’ 스테가노그래피 기법을 통해 PNG 파일에 Malware 숨김
    
    **악성코드 유형**
    
    - 컴퓨터 정보 탈취 악성코드를 위한 PNG 파일
    
    Worok’s complete infection chain
    
    **동작 방식**
    
    1. PNG 파일에 Malware 숨기기
    2. PNG에 Payload 숨기기
        1. 이미지 픽셀의 가장 중요하지 않은 비트에 악성코드의 작은 코드를 삽입하는 ‘최하위 비트(LSB) 인코딩’ 이라는 기술 사용
        2. PNGLoader가 해당 비트에서 추출한 첫 번째 Payload는 ESET이나 Avast 모두 검색할 수 없는 PowerShell script임
        3. PNG 파일에 숨어 있는 두 번째 Payload는 C2 통신, 파일 유출 등을 위해 DropBox 파일 호스팅 서비스를 남용하는 맞춤형 .NET C# 정보 스틸러(DropBoxControl)
    3. DropBox abuse
        1. ‘DropBoxsControl’ Malware는 행위자가 제어하는 DropBox 계정을 사용하여 손상된 시스템에서 데이터 및 명령을 수신하거나 파일 업로드 수행
        2. 명령은 공격자의 DropBox 리포지토리에 있는 암호화된 파일에 저장되며 Malware는 대기 중인 작업을 검색하기 위해 주기적으로 액세스
    4. 지원되는 명령
        - 주어진 매개변수로 "cmd /c" 실행
        - 주어진 매개변수로 실행 파일 실행
        - DropBox에서 장치로 데이터 다운로드
        - 장치에서 DropBox로 데이터 업로드
        - 피해자 시스템의 데이터 삭제
        - 피해자 시스템의 데이터 이름 바꾸기
        - 정의된 디렉토리에서 파일 정보 추출
        - 백도어에 대한 새 디렉토리 설정
        - 시스템 정보 유출
        - 백도어 구성 업데이트 

[Worok hackers hide new malware in PNGs using steganography](https://www.bleepingcomputer.com/news/security/worok-hackers-hide-new-malware-in-pngs-using-steganography/)