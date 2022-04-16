---
layout: post
title: 나만의 Git서버 만들기
---


## 옛날에 만들어 둔 클라우드 서버를 어떻게든 사용하기 위해 설정한 나만의 Git서버 만드는 방법이다

여러가지 방법을 사용하여 Git server를 구축해볼것이다.

해당 페이지는 [Git 홈페이지](<https://git-scm.com/book/ko/v2/Git-%EC%84%9C%EB%B2%84-%EC%84%9C%EB%B2%84%EC%97%90-Git-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0>)의 설명을 많이 참고하였다.

[추가적으로 해당 문서도 참고하였음.](<https://git-scm.com/book/ko/v2/Git-%EC%84%9C%EB%B2%84-%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C>)

----

### 서버용 깃 폴더 설정하기

서버용 깃 폴더는 다음과 같이 설정할 수 있다.

> git init --bare (???.git등의 형태로 만든 빈 볼더 안에서) (관례상으로 서버용 깃 폴더는 .git으로 끝나기 때문이다.)
>
> git clone <https://github.com/>(사용자 이름)/(레포지토리 이름) --bare

위와 같은 형태로 --bare옵션을 넣어주는 형태로 깃을 설정할 시, 작업들을 저장할 수 있는 작업 폴더가 생성된다.

----

### SSH를 이용한 깃서버 접속

전문적으로 git을 관리하기 위한 계정을 만들어준다.

> sudo useradd git(아니면 다른 이름을 사용한다) (옵션(깃 폴더 지정 등))

이후 위의 깃 폴더 설정 이후 해당 명령어를 사용해 깃을 복사할 수 있다.

> git clone ssh://git(계정명)@(주소)/(깃 폴더 절대주소)
>
> git clone git(계정명)@(주소):(홈 폴더 기준 상대주소) (계정명을 생략할 시 현재 로그인한 사용자의 계정을 사용한다고 한다.)

이때 주의해야 할 점은, 위의 방법은 홈폴더 기준이 아닌 루트폴더 기준이라는것이다.

홈 폴더에 있는 레포지토리를 클론하기 위해선 ssh://git@homepage/~/example.git 등의 방법을 사용해야한다.

> 좀 더 편하게 ssh를 이용하기 위해선 rsa키를 설정해서 이용하는것이 좋다.
>
> 그것이 아니라면 ssh에 비밀번호를 이용하여 접근할 수 있는지 확인하는것은 필수다.
>
> 다음과 같은 방법을 사용했는데 오류가 나타난다면 해당 유저가 레포지토리 폴더에 접근할 수 있는지 확인하라. (폴더의 권한이나 사용자, 그룹설정 등)

----

### Git daemon을 이용한 깃서버 접속

깃 데몬을 이용하는 방법은 생각보다 까다로웠다.

Git daemon은 기본적으로 인증 절차가 없기 때문.

[해당 문서를 참고하였다.](<https://git-scm.com/docs/git-daemon>) [설명이 더 많은 한국어버전](<https://git-scm.com/book/ko/v2/Git-%EC%84%9C%EB%B2%84-Git-%EB%8D%B0%EB%AA%AC>)

대부분의 설정은 그냥 따라하면 되지만 푸시를 할 수 없기 때문에 추가적인 설정이 필요하다.

> 다음과 같은 방법으로 /etc/systemd/system/git-daemon.service 파일을 작성한다.
>>
>> [Unit]
>>
>> Description=Start Git Daemon
>>
>> [Service]
>>
>> ExecStart=/usr/bin/git(깃 실행파일 주소) daemon --reuseaddr --base-path=/srv/git/(깃 저장소 폴더) /srv/git/(깃 저장소 폴더)
>>
>> Restart=always
>>
>> RestartSec=500ms
>>
>> StandardOutput=syslog
>>
>> StandardError=syslog
>>
>> SyslogIdentifier=git-daemon (service명령어를 볼 때 사용되는 듯 하다)
>>
>> User=git(사용자)
>>
>> Group=git(그룹)
>>
>> [Install]
>>
>> WantedBy=multi-user.target
>
> 이후 systemctl enable git-daemon명령을 통해서 활성화를 시킬 수 있다. (우분투 14.04 이전버전이면 홈페이지에 기재된 방법 사용)

위에 적힌 스크립트에서 ( )부분은 사용자의 설정에 따라 바뀔 수 있다.

> 깃 데몬은 인증이 없기 때문에 push도 못하며 클론도 기본적으로 못한다 (예상치 못한 파일을 모르는 사람이 가져다 쓸 수 있기 때문)
>
> 그렇기 때문에 각각의 레포지토리에 git-daemon-export-ok 이름의 파일을 생성해야 깃 데몬이 접근을 허용해주며,
>
> 깃 데몬은 하나의 포트를 사용하기 때문에 9418포트의 방화벽을 확인하여야 한다. (grep 9418 /etc/services명령을 사용해 포트를 제대로 사용하는지 확인 가능하다.)
>
> 또한 Git daemon은 인증이 없기 때문에 푸시도 기본적으로 못하는데, 푸시를 할 수 있도록 하려면 --enable=receive-pack 옵션을 [Service]의 ExecStart라인 안에 넣어주면 된다.

[옵션에 대한 자세한 설명은 다음을 참고하면 된다.](<https://git-scm.com/docs/git-daemon>)

이후 위의 깃 폴더 설정 이후 해당 명령어를 사용해 깃을 복사할 수 있다.

> git clone git://(주소)/(깃 폴더 상대주소)

----

이외에도 nginx를 이용한 http프로토콜에 대한 깃 클론을 사용해보려 하였으나, 검색해보면 나오는것들은 대부분 기존의 웹서버가 있을때를 기준으로 작성되었기 때문에 포기하였다.

----

### 기존의 깃 폴더에 새로운 저장소 추가

기존의 깃 폴더에 새로운 remote를 추가하는 방법이다.

> git remote add (리모트 이름) (위에서 사용한 여러 주소중 하나)

위와 같은 명령어를 사용해 remote를 추가할 수 있다.

- remote 삭제 - git remote remove (리모트 이름)
- remote 이름 변경 - git remote rename (리모트 이름) (목표 이름
- remote 목록 보기 - git remote
- remote 주소까지 보기 - git remote -v

또한 다음과 같은 방법으로 해당 리모트에 푸시할 수 있다.

> git push (리모트 이름) [브랜치 이름 추가가능]

그 외에도 (리모트 이름)/(브랜치 이름) 등의 방법으로 이용 가능하니 입맞에 맞는 방법으로 사용하면 된다.

> 처음엔 gitlab 커뮤니티 에디션을 오라클 무료 클라우드서버에 설치하려고 하였다.
>
> 하지만 gitlab-ctl reconfigure에서 오류가 자주 일어나서 서비스도 없애고 메모리 스왑도 사용하고 초기화도 하고 포트도 바꾸고 그랬지만 나아지지 않아서 보니, gitlab에서 사용하는 puma 서비스는 여러 스레드를 사용하는 도구인데, 오라클 무료 클라우드 서버(1스레드)에서는 돌리기가 마땅치 않았던 것으로 보인다. (설정에서 1스레드만 사용하도록 변경하여도 오류가 일어났다.)
>
> 해당 정보는 gitlab-ctl tail puma를 이용하여 보았다. 다른 서비스는 정상 작동하지만 puma가 서비스종료코드로 계속 종료되어서 502-Whoops, GitLab is taking too much time to respond가 떴었다.
>
> 알고보니 puma는 4코어 cpu, 4기가 램의 최소사양을 가지고 있었다.
>
> Gitlab 내부만 분리해서 사용하는 방법이 있다는것같은데 기본설치는 은근 무거우니 시도해보지 말 것.
