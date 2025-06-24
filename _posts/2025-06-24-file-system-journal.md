---
layout: post
title: "운영체제 별 파일 시스템 저널 읽기"
summary: "저널링 파일 시스템 리서치"
author: eveheeero
date: '2025-06-24 05:31:00 +0900'
category: ['forensic', 'learning']
tags: forensic
thumbnail: /assets/img/posts/2025-06-24-1.png
keywords: forensic
usemathjax: false
permalink: /blog/file-system-journal/
---

## 배경

파일 시스템은 각각 저널 시스템을 가지고 있어, 어떤 파일에 어떠한 접근을 했는지 등을 기록합니다.

운영체제별 파일 시스템 저널을 보는 방법에 대해 살펴보겠습니다.

## 요약

- 윈도우 : `fsutil usn readJournal C: csv`
- 리눅스 : `debugfs -R 'logdump -a' /dev/sda1`

## 문제 상황

커널 시스템의 저널링 파일 시스템에 대해 연구중 알아보게 되었습니다.

## 분석 과정

### 윈도우

#### USN (기록용 로그)

윈도우 Usn의 경우, `fsutil usn`을 통해 저널을 확인할 수 있습니다.

`$UsnJrnl`의 파일을 읽어 파싱하는 타 응용 프로그램과 동일한 데이터를 다루는 것으로 예상됩니다.

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

#### ntfs transaction (수정용 저널)

파일 수정용 저널의 경우 `$LogFile`에 기록됩니다. 해당 관련 내용은 자세하게 적혀있지 않아 참고 영역을 참조하시면 좋습니다.

| 관련 분석 툴로 `NTFS Log Tracker`, `Windows Journal Parser`, `LogFileParser`이 있다고 합니다. [windows forensic yum_yum tistory](https://yum-history.tistory.com/285)을 참조해주세요.

`$LogFile`을 보기 위해 윈도우에서 기본적으로 제공되는 툴은 아직 찾지 못했으며, `FTK Imager` 혹은 `Autospy`를 사용해 파일을 구한 후, [`NTFS Log Tracker`](https://sites.google.com/site/forensicnote/ntfs-log-tracker)을 사용하겠습니다.

![autospy logfile dump](/assets/img/posts/2025-06-24-6.png){: style="max-width: 100%; height: auto;"}

![ntfs log tracker](/assets/img/posts/2025-06-24-7.png){: style="max-width: 100%; height: auto;"}

제대로 보이고 있습니다.

### 리눅스

리눅스의 경우, 원하는 경로가 어디에 마운트 되어있는지 확인 후 저널을 확인할 수 있습니다.

`df`, `lsblk`, `blkid`등의 명령어를 통해, 원하는 경로가 어디에 마운트되어있는지 확인합시다.

![lsblk 내용](/assets/img/posts/2025-06-24-2.png){: style="max-width: 100%; height: auto;"}

이후 `debugfs /dev/sha2`를 통해 드라이브를 연 후 `logdump -S` 혹은 `logdump -O`, `logdump -a`를 통해 저널 목록을 확인할 수 있습니다.

![logdump -o 내용](/assets/img/posts/2025-06-24-3.png){: style="max-width: 100%; height: auto;"}

![logdump -a 내용](/assets/img/posts/2025-06-24-4.png){: style="max-width: 100%; height: auto;"}

`ext4`에 대해 제대로 저널 표시되고 있습니다.

이후 `logdump -a`를 통해 확인된 `FS block`을 `block_dump 1234...` 등을 통해 어던 내용이 실제로 적혔는지 확인 가능합니다.

![block_dump 내용](/assets/img/posts/2025-06-24-5.png){: style="max-width: 100%; height: auto;"}

이외에도 ext관련 파일 시스템에 대한 여러 명령어가 있어서 `man debugfs`로 확인해보면 좋을 듯 합니다.

#### 저널링 모드 및 데이터 저널

ext4는 세가지 저널링 모드가 있습니다.

- journal - 데이터까지 기록되는 저널링 (성능이 느려짐)
- journal_ordered - 데이터 제외, 메타데이터만 저널링(기본)
- journal_writeback - writeback모드

어떤 모드가 적용되었는지는 `dmesg | grep EXT4`입력시 확인 가능하며, 다음과 같은 문구가 포함됩니다.

- journal - `...with journalled data mode...`
- journal_ordered - `...with ordered data mode...`
- journal_writeback - ??

적용 및 수정은 다음처럼 할 수 있습니다.

`tune2fs -O has_journal -o journal_data /dev/sda1`

- `has_journal` - 저널 사용, `^has_journal`시 저널 미사용
- `journal_data` - 데이터 저널링 사용, `journal_data_ordered` 및 `journal_data_writeback` 사용 가능, `man tune2fs` 참조

| 재부팅 필요

저널 로그에 제대로 기록되는 모습입니다.

| 작성한 내용은 `abcdefttt\nlowpo`입니다.

| 전용 분석 툴이 있지 않을까 싶습니다.

![입력한 데이터가 기록된 모습](/assets/img/posts/2025-06-24-8.png){: style="max-width: 100%; height: auto;"}

## 작성자의 글

- 윈도우는 보기 편하게 해주는 반면 리눅스는 순정 상태의 저널 그대로를 보여주네요
- 특정 파일에 대해 수정된 기록을 보려면 프로그래밍적으로 api로 구현하거나 해야 할 듯 합니다.
- `NTFS Log Tracker`가 글 작성일로부터 2일 전에 업데이트가 됐네요, 정말 감사한 마음입니다.

## 참조

- [MSDN](https://learn.microsoft.com/ko-kr/windows-server/administration/windows-commands/fsutil-usn)
- [ntfs kci](https://www.kci.go.kr/kciportal/ci/sereArticleSearch/ciSereArtiView.kci?sereArticleSearchBean.artiId=ART001455925)
- [ntfs.com](https://www.ntfs.com/transaction.htm)
- [windows forensic yum_yum tistory](https://yum-history.tistory.com/285)
- [Linux stackoverflow](https://stackoverflow.com/questions/11114575/accessing-ext3-ext4-journals)
