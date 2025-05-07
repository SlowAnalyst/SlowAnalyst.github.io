---
layout: post
title: "러스트 에러 콜스택"
summary: "러스트 에러 정보 내부 콜스택 추가하기"
author: eveheeero
date: '2025-05-08 03:43:00 +0900'
category: ['rust', 'learning']
tags: rust
thumbnail: /assets/img/posts/default.png
keywords: rust, backtrace, error
usemathjax: false
permalink: /blog/rust-backtraced-error/
---


## 배경

새로운 프로젝트를 시작하면서 전에 만들어둔 `백트레이스가 포함된 에러 처리`가 생각나 작성하였습니다.

## 요약

- `From`처리를 통해서만 생성할 수 있는 `backtracedError` 구조체를 생성 후, `From`구현체에 백트레이스 생성 로직을 넣는다.

## 문제 상황

러스트 오류 타입을 그냥 생성할 시 백트레이스가 생성되지 않는다. 로깅에 어려움이 있어 수정이 필요하다.

## 분석 과정

```rust
pub enum Error {
    Unknown(String),
}
pub struct BacktracedError {
    backtrace: std::backtrace::Backtrace,
    inner: Error,
}
impl BacktracedError {
    pub fn backtrace(&self) -> &std::backtrace::Backtrace {
        &self.backtrace
    }
    pub fn inner(&self) -> &Error {
        &self.inner
    }
}
impl From<Error> for BacktracedError {
    fn from(value: Error) -> Self {
        // 이곳에 tracing등을 통해 로깅을 넣어도 좋습니다.
        BacktracedError {
            backtrace: std::backtrace::Backtrace::capture(),
            inner: value,
        }
    }
}
fn error_occured() -> Result<(), BacktracedError> {
    Err(Error::Unknown(String::from("error")))?
}
```

## 알아낸 내용

- .
