---
layout: post
title: "Rust - 정적 코드 분석"
summary: "인라인 타입 체킹"
author: eveheeero
date: '2023-10-14 13:36:22 +0900'
category: ['rust', 'learning']
tags: rust
thumbnail: /assets/img/posts/default.png
keywords: rust, trait, macro
usemathjax: false
permalink: /blog/Rust-Static_Validation/
---

## 요약

Rust 코드를 자면서, 매크로 이용 중 인라인에서 A trait을 impl하고 있는지 확인을 위해 `static_assertion` 분석 중 찾아낸 응용법입니다.

```rust
/* A트레잇을 구현하지 않았을 때 동작할 내용을 정의합니다. */
struct TraitNotImplemented {
  const DATA: bool = false;
  fn get_data() -> &str { "This type/trait does not implement that trait." }
}
// 모든 타입에 대해 ::DATA를 적으면 false가, ::get_data를 적으면 위 메세지가 나오게 합니다.
impl<T> TraitNotImplemented for T {}

struct Wrapper<T>(::core::marker::PhantomData<T>);
/* A트레잇을 구현했을 때 동작할 내용을 정의합니다. */
// 타입에 대해 직접 impl한게 우선권이 높아
// A를 구현한 타입에 ::DATA를 하면 true, ::get_data를 적으면 아래 메세지가 나오게 됩니다.
impl<T: ?Sized + A> Wrapper<T> {
  const DATA:bool = true;
  fn get_data() -> &str { "This type/trait implement that thait" }
}

let b_type_impl_a = <Wrapper<B>>::DATA;
```
