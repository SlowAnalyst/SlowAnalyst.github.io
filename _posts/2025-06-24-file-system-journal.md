---
layout: post
title: "운영체제 별 파일 시스템 저널 읽기"
summary: "파일 시스템 저널"
author: eveheeero
date: '2025-06-24 05:31:00 +0900'
category: ['forensic', 'learning']
tags: forensic
thumbnail: /assets/img/posts/2025-06-24-1.png
keywords: forensic
usemathjax: false
permalink: /blog/file-system-journal/
---

```text
해당 내용은 저널링 파일 시스템이 아닌, 파일 시스템의 저널에 대해 다루고 있습니다.
```

## 배경

파일 시스템은 각각 저널 시스템을 가지고 있어, 어떤 파일에 어떠한 접근을 했는지 등을 기록합니다.

운영체제별 파일 시스템 저널을 보는 방법에 대해 살펴보겠습니다.

## 요약

- 윈도우 : `fsutil usn readJournal C: csv`
- 리눅스 : `debugfs -R 'logdump -a' /dev/sda1`

## 문제 상황

커널 시스템의 저널링 파일 시스템에 대해 연구하려 했으나, 비슷한 키워드를 가진 해당 내용을 찾게 되었습니다.

## 분석 과정

### 윈도우

윈도우의 경우, 윈도우에서 자체 지원해주는 `fsutil`을 통해 저널을 확인할 수 있습니다.

`$UsnJml`의 파일을 읽어 파싱하는 타 응용 프로그램과 동일한 데이터를 다루는 것으로 예상됩니다.

`fsutil usn`입력시 다음처럼 명령어들이 뜨는데, 이 중 사용 가능한 것은 `readJournal`입니다.

![fsutil usn 실행 화면](/assets/img/posts/2025-06-24-0.png){: style="max-width: 100%; height: auto;"}

```text
- `fsutil usn readData`는 파일에 대한 로우 객체를 출력합니다.
- `fsutil usn queryJournal`은 드라이브의 로우 객체를 출력합니다.
```

`fsutil usn readJournal`을 칠 경우 `fsutil usn readJournal <볼륨 경로 이름> [옵션]`과 같은 설명서가 뜨는데, 옵션들은 다음과 같습니다.

- csv : csv로 출력, 콘솔로 출력되므로 파이프라이닝 필요
- wait : 해당 명령어를 친 이후에 작성되는 저널을 계속 표시
- tail : 저널 표시 순서 반전

csv옵션을 통해 저널을 출력하고 나면 다음처럼 파일이 드랍됩니다.

![fsutil journal 내용](/assets/img/posts/2025-06-24-1.png){: style="max-width: 100%; height: auto;"}

해당 파일엔 N MB용량의 파일 수정 기록이 나오게 됩니다.

### 리눅스

리눅스의 경우, 원하는 경로가 어디에 마운트 되어있는지 확인 후 저널을 확인할 수 있습니다.

`df`, `lsblk`, `blkid`등의 명령어를 통해, 원하는 경로가 어디에 마운트되어있는지 확인합시다.

![lsblk 내용](/assets/img/posts/2025-06-24-2.png){: style="max-width: 100%; height: auto;"}

이후 `debugfs /dev/sha2`를 통해 드라이브를 연 후 `logdump -S` 혹은 `logdump -O`, `logdump -a`를 통해 저널 목록을 확인할 수 있습니다.

![logdump -o 내용](/assets/img/posts/2025-06-24-3.png){: style="max-width: 100%; height: auto;"}

![logdump -a 내용](/assets/img/posts/2025-06-24-4.png){: style="max-width: 100%; height: auto;"}

`ext4`에 대해 제대로 저널 표시되고 있습니다.

## 작성자의 글

- 윈도우는 보기 편하게 해주는 반면 리눅스는 순정 상태의 저널 그대로를 보여주네요

## 참조

- [MSDN](https://learn.microsoft.com/ko-kr/windows-server/administration/windows-commands/fsutil-usn)
- [Linux stackoverflow](https://stackoverflow.com/questions/11114575/accessing-ext3-ext4-journals)
