---
layout: post
title: "SurrealDB란 무엇인가?"
summary: "Rust로 개발된 Surreal DB가 무엇인지"
author: eveheeero
date: '2023-05-31 12:47:41 +0900'
category: ['learning']
tags: database
thumbnail: /assets/img/posts/2023-05-31-0.png
keywords: database, db, surrealdb, rust, sql
usemathjax: false
permalink: /blog/SurrealDB/
---


> 백엔드 개발자인 주제에 데이터베이스를 제대로 다뤄보지 않아, 이번 기회에 데이터베이스를 사용하는 방법을 공부해보았습니다.

### [SurrealDB](https://surrealdb.com/)

[깃허브](https://github.com/surrealdb/surrealdb)

순수 Rust로 작성된 데이터베이스로, 1.0 릴리즈 버전이 나오기 직전이라는 소문이 있어 사용해 보았습니다.

설치는 깃허브 페이지에서 실행파일을 다운로드하거나, 설치 명령어들을 실행해서 설치할 수 있습니다.

```bash
# Mac
brew install surrealdb/tap/surreal
# Linux
curl --proto '=https' --tlsv1.2 -sSf https://install.surrealdb.com | sh
# Windows
iwr https://windows.surrealdb.com -useb | iex
# Docker 
docker run --rm --pull always --name surrealdb -p 8000:8000 surrealdb/surrealdb:latest start
```

### 실행

SurrealDB 설치 후 커멘드로 실행해보았습니다.

![SurrealDB 실행 모습](/assets/img/posts/2023-05-31-0.png){: style="max-width: 100%; height: auto;"}

실행해보니 다른 디비와 마찬가지로, 실행한 창에서는 커멘드를 입력할 수 없었습니다. \(데이터베이스를 자주 다루지 않아서 미숙합니다.\)

깃허브 페이지에서 보면, 웹 클라이언트가 있어 보였는데, 아직 미출시여서 클라이언트로 접속할 수 밖에 없었습니다.

디비에 접속하여 사용하는 법을 익혀보았습니다.

개발 초기단계의 데이터베이스여서 그런지, 문서의 양이 많지 않아 원하는 내용을 쉽게 찾을 수 있었습니다.

![SurrealDB 패스워드 설정 실행](/assets/img/posts/2023-05-31-1.png "패스워드 설정 필요")

해당 설명을 보면 아이디와 패스워드를 확인 후, 해당 패스워드를 이용해 접속해야 합니다.

`surreal sql -c http://localhost:8000`

> -u <유저명> -p <패스워드> --ns <네임스페이스> --db <데이터베이스> 를 입력해야하지만, 유저명 기본값은 root, 패스워드 기본값은 root로 설정되어 있습니다.
>
> --ns와 --db는 입력하지 않아도 접속할 수 있습니다. \(데이터베이스 조회는 안됩니다.\)

### 기본 조작

[해당 링크](https://surrealdb.com/docs/surrealql/statements)를 접속하여 기본적인 헬프 명령어를 볼 수 있습니다.

```sql
-- 네임스페이스 변경 (없으면 생성 후 변경)
use ns namespace1
-- 데이터베이스 변경 (없으면 생성 후 변경)
use db db1

-- namespace1/db1 안에 들어와있습니다.
-- 테이블을 정의할 필요 없이 create 명령어로 데이터를 삽입할 수 있습니다.

-- human이라는 테이블에 hero라는 아이디를 가진 데이터를 넣어라
-- human이라는 테이블이 없으면 만들고 데이터를 넣습니다.
create human:hero
-- 만들어진 데이터 업데이트, 열을 설정하지 않아도 동작합니다.
update human:hero set region = "대한민국"
-- create human:hero set region = "대한민국" 으로도 동작합니다.

-- human 테이블 조회
select * from human
-- human 테이블 안의 hero 데이터 조회
select * from human:hero
```

![기본적인 데이터 삽입](/assets/img/posts/2023-05-31-2.png){: style="max-width: 100%; height: auto;"}

CREATE를 사용했을 때 데이터가 들어가는것으로 보아, 다른 데이터베이스를 쓰다가 오면 호환성 작업이 조금 필요할 것 같지만, 테이블을 미리 정의해 둘 필요가 없기 때문에 유연하게 데이터 사용이 가능해 보입니다.

기본 INSERT문도 지원하지만 저는 CREATE명령어가 마음에 듭니다.

추가적으로 [타입](https://surrealdb.com/docs/surrealql/datamodel)과 [내장 함수들](https://surrealdb.com/docs/surrealql/functions)도 유용한 내용들이 있어 편하게 사용 가능해 보입니다.

### 이미 존재하는 데이터 찾기

SurrealDB에는 메타데이터들이 없는 대신 `info`명령어를 통해 어떤 데이터가 저장되어있는지 찾을 수 있습니다.

```sql
-- 어떤 네임스페이스가 정의되어 있는지 조회 (데이터가 들어간 네임스페이스만 조회)
info for kv
-- > info for kv
-- { ns: { namespace1: 'DEFINE NAMESPACE namespace1' } } -- namespace1이라는 네임스페이스가 존재함
use ns "네임스페이스 명"

-- 네임스페이스 안에 존재하는 데이터베이스 조회 (데이터가 들어간 데이터베이스만 조회)
info for ns -- 설명대로는 동작해야 하는데, 데이터베이스가 설정되지 않으면 현재는 동작하지 않습니다. use db temp로 아무런 데이터베이스를 사용한다고 선언한 후 사용하시면 됩니다.
-- > use db temp
-- namespace1/temp> info for ns
-- { db: { db1: 'DEFINE DATABASE db1' }, nl: {  }, nt: {  } } -- db1이라는 데이터베이스가 존재함
use db "데이터베이스 명"

-- 데이터베이스 내부에 어떤 테이블이 있는지 조회 (데이터가 들어간 테이블만 조회)
info for db
-- namespace1/db1> info for db
-- { dl: {  }, dt: {  }, fc: {  }, pa: {  }, sc: {  }, tb: { human: 'DEFINE TABLE human SCHEMALESS PERMISSIONS NONE' } } -- human이라는 테이블이 존재함
```

> SurrealDB를 시작으로 여러 데이터베이스 다루는 법을 익혀야겠습니다.
