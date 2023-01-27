---
layout: post
title:  "Rust - Task, Future, Context 사용법"
summary: "std::task::Context를 사용하는 방법"
author: eveheeero
date: '2023-01-27 10:40:23 +0530'
category: ['rust','learning']
tags: rust
# thumbnail: 
keywords: rust, task, future, context
usemathjax: false
permalink: /blog/rust_task-future-context/
---


Rust std 라이브러리를 둘러보면, 생소한 task모듈이 있는 것을 볼 수 있다.

해당 task모듈에는 다른 모듈과 함께 작성할 수 있는 Context기능이 들어있다.

----

Context - 여러 작업의 시작 혹은 종료를 다루는 전반적인 플래그

현재 Rust의 Context는 많은기능을 담고있지 않으며, waker() 메소드를 통해 waker를 받는 기능밖에 구현되어 있지 않습니다.

[Context.as_raw() 를 이용해 RawWaker를 받은 후, RawWaker.data()를 사용하면 원본 데이터의 포인터를 얻을 수 있는 것 같으나, 현재는 둘 다 nightly단계입니다.](https://github.com/rust-lang/rust/issues/87021)

----

전체 소스는 다음과 같습니다.

```rust
use std::{
    future::Future,
    pin::Pin,
    sync::Arc,
    task::{Context, Poll, RawWaker, RawWakerVTable, Waker},
};

static VTABLE: RawWakerVTable = RawWakerVTable::new(
    |data| unsafe {
        // RawWaker를 클론할 때 수행해야 하는 작업 명시
        // Clone data perform

        // Arc를 복사해서
        let arc = Arc::from_raw(data as *const i32);
        // 레퍼런스 카운트를 늘려준 뒤
        std::mem::forget(arc.clone());
        // 기존 포인터로 새로운 RawWaker를 생성한다.
        // 이때 자신의 주소가 필요하기 때문에 VTABLE을 전역변수로 지정한 듯 하다.
        RawWaker::new(Arc::into_raw(arc) as *const _, &VTABLE)
        // 혹은 RawWaker::new(data, &VTABLE)
    },
    |data| {
        // 포인터를 기반으로 프로그램 중지 명령 (Mutex 락 설정)
        // Data lock
    },
    |data| {
        // 레퍼런스를 기반으로 프로그램 중지 명령 (Mutex 락 설정)
        // Mutex lock by ref
    },
    |data| unsafe {
        // RawWaker를 드롭할 때 수행해야 하는 작업 명시
        // Drop data perform

        // Arc 레퍼런스카운터 제거
        drop(Arc::from_raw(data as *const i32));
    },
);

fn main() {
    // 컨텍스트에 들어갈 데이터, 테스트용으로 숫자를 넣었지만, mutex나 condition var이 들어간 struct이 들어갈 경우가 많을듯하다.
    let data = Arc::new(123123123);
    // 데이터를 포인터로 만든 후 RawWaker 생성, VTABLE엔 클론, 드롭시 수행해야 할 작업 명시
    let waker = RawWaker::new(Arc::into_raw(data) as *const _, &VTABLE);
    // RawWaker를 Waker로 변환
    let waker = unsafe { Waker::from_raw(waker) };
    // Waker로 Context 생성
    let mut ctx = Context::from_waker(&waker);

    // Context로 특정 작업 수행
    // Do something with context

    // Context를 수행하는 작업은 std::task::Poll 을 반환값으로 이루어진다.
    // Run function with context that returns Poll(result)

    // std::future::ready 함수를 통해 반환할 값을 준비합니다. 준비되지 않았으면 std::future::pending함수를 사용해 준비되지 않았다고 알립니다.
    let mut result = std::future::ready(123);
    // result.poll(&mut ctx)를 통해 std::future::Ready 혹은 Pending에서 std::task::Poll로 변환해줍니다.
    // poll을 사용하기 위해선 Pin을 통해, 변수 위치가 고정되어야 합니다. (여러 스레드에서 사용하기 때문에 오류방지를 위해)
    let result = Pin::new(&mut result);
    let result = result.poll(&mut ctx);

    // Poll match를 통해 값이 준비되었는지 아닌지를 판단합니다.
    // std::task::ready! 를 사용하면, 값이 준비되지 않았을 경우 Poll::Pending (준비중)을 반환할 수 있습니다.
    let result = match result {
        Poll::Ready(data) => data,
        Poll::Pending => 0,
    };

    println!("Result: {}", result);
}
```

----

Rust의 로우 모듈인 std::task::Context에 대해 보았습니다.

해당 기능을 사용하여, async를 사용하지 않고도 비동기 테스크 프로그램을 작성하거나, 여러 병렬처리 프로그램을 작성할 수 있습니다.

Go나 다른 언어처럼, Context에 데이터를 담는 기능은 없어보이며, 현재는 waker를 호출해, 다른 전체작업을 종료하는 역활밖에 없습니다.

Spring batch처럼 테스크 기반 프로그램을 작성할 때 유용한 것 같습니다.

> 해당 글의 내용은 [해당 링크](https://github.com/spacejam/extreme)를 참조하였습니다.
>
> 내부 구현의 데이터에는 Arc를 사용하였지만, 반드시 Arc를 사용할 필요는 없습니다. 계속해서 유지되는 데이터의 포인터면 됩니다. static을 사용하거나 Box::leak를 사용해도 좋을 듯 합니다.
