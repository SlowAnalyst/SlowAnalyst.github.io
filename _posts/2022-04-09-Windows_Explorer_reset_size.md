---
layout: post
title: Windows Explorer reset size by Registry
---


## 완벽주의자 환자의 클릭실수를 만회하고자 하는 윈도우 파일탐색기 사이즈 리셋 방법이다.

레지스트리 폴더를 뒤자다가 Explorer 내부에 사이즈관련 값이 없어서 GG... (창 변경 및 새로고침을 통해서 찾아보는 방법을 해도 찾을 수 없었다.)

결국 구글의 힘을 빌렸다.

> \\HKEY_CURRENT_USER\\Software\\Classes\\Local Settings\\Software\\Microsoft\\Windows\\Shell\\Bags\\AllFolders\\Shell
>> (혹은 \\HKEY_CLASSES_ROOT\\Local Settings\\Software\\Microsoft\\Windows\\Shell\\Bags\\AllFolders\\Shell 과 비슷하다.)
> \\HKEY_CURRENT_USER\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Explorer\\Modules\\GlobalSettings\\Sizer

![\\HKEY_CURRENT_USER\\Software\\Classes\\Local Settings\\Software\\Microsoft\\Windows\\Shell\\Bags\\AllFolders\\Shell의 내용](<https://eveheeero-github-io.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fba65d508-0336-43ed-adbf-c66f078019eb%2FUntitled.png?table=block&id=613500e3-9c6f-4a70-b599-66136631f319&spaceId=c2eb73c4-6260-4fb7-8470-2e07bff25e55&width=2000&userId=&cache=v2> "다음과 같은 값을 지워주면 된다.")

[이곳](https://superuser.com/questions/917231/clear-file-explorer-settings-position-size-etc)과 [이곳](https://www.majorgeeks.com/content/page/how_to_reset_file_explorer_navigation_pane_width_to_default.html)을 참고하였다.

레지스트리의 \\HKEY_LOCAL_MACHINE\\SOFTWARE (혹은 \\HKEY_USERS\\(유저 UID)\\Software 과 같다) 에는 여러 소프트웨어에 대한 설정값이 존재하며, 해당 키의 내부를 살펴보면 여러 윈도우들에 대한 설정값을 찾을 수 있다.

\\HKEY_CLASSES_ROOT이나 \\HKEY_CURRENT_USER\\Software\\Classes에는 여러 열기 방식이나 앱들의 설정값, 확장자 지정이 적혀있다.

\\HKEY_CURRENT_USER\\Software\\Microsoft\\Windows\\CurrentVersion(이하 동일)에는 윈도우 프로그램들의 설정에 대해 적혀 있다.

그 외, \\HKEY_LOCAL_MACHINE\\SOFTWARE (\\HKEY_CLASSES_ROOT와 동일) 내부에는 시스템 전역적인 공통설정이 들어가있으므로 자신이 사용하는 프로그램의 사이즈가 적혀있지 않으면 참고하라.

추가적으로 \\HKEY_CURRENT_USER\\Software\\Microsoft\\Windows\\CurrentVersion의 내부에는 자동실행인 RUN이나 여러가지 정책, 윈도우 패스, 자동실행폴더 등이 지정되어 있다.

위와 같은 값들을 지우면 여러 창들의 크기를 원래대로 돌릴 수 있으니 참고하라. (가상머신에서 디폴트값을 받은 뒤, 파일->내보내기로 세팅값 추출한다음 원래 컴퓨터에서 사용하는것 추천)
