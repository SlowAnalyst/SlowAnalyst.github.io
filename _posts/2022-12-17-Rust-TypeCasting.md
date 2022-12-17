---
layout: post
title: Rust 타입 변환
---


## Rust에서 타입 변환을 시도하는 방법은 다음과 같다

- std::mem::transmute::<T, Y>(data)
- std::mem::transmute_copy::<T, Y>(data)
- as
- std::convert::From
- std::convert::TryFrom
- std::convert::Into
- std::convert::TryInto

## 상세 설명

### std::mem::transmute::<T, Y>(data)

같은 비트 수의 데이터 타입을 변환합니다. 즉 u32에서 i32로 변환하는 등의 작업이 가능합니다.

주로 포인터간의 변환을 위해 사용됩니다.

```rust
let var = 1;
let data: *const _ = &var;
let ptr = unsafe { std::mem::transmute::<*const _, &mut i32>(data) };
// 혹은 let ptr: &mut i32 = unsafe { std::mem::transmute(data) };
*ptr += 1;
```

### std::mem::transmute_copy::<T, Y>(data)

T타입의 비트 수만큼의 메모리를 복사하여, 해당 메모리와 같은 값을 가진 Y타입 데이터를 생성합니다.

프리미티브 타입에서 사용되지 않으며, 구조체 내부의 메모리를 그대로 복사하는데 사용됩니다.

```rust
struct A {
  data1: i32,
  data2: u64,
}

struct B {
  data1: u32,
  data2: i64,
}
let a = A { data1: 0, data2: 1 };
let b = unsafe { std::mem::translate_copy::<A, B>(a) };
// b의 data1에는 a의 data1 데이터가, b의 data2에는 a의 data2 데이터가 그대로 들어가있다.
// (타입변환등이 일어나지 않아 unsigned int를 int로 제대로 바꿀 순 없습니다.)
// 변수 순서를 제대로 지키는지 알 수 없습니다. (data1, data2 순으로 정의되었지만,
// 그 내부엔 패딩이 있을수도 있고, data2, data1순으로 데이터가 있을수도 있습니다.)
```

### as

해당 타입 변환은 Primitive 타입에서만 사용 가능합니다.

Rust에서 배열의 인덱스를 지정하기 위해선 기본적으로 usize타입이 필요하기 때문에, 해당 타입으로 변환해주기 위해 주로 사용합니다.

```rust
let index = 123u32;
array[index as usize];
```

### std::convert::From

보통 Rust에서는 A타입의 구조체를 B타입으로 변환하기 위해 From trait을 구현시켜줍니다.

`impl From<A> for B`

해당 From에는 반드시 구현해줘야 하는 함수가 있으며, 해당 함수를 구현해 줄 경우 여러 타입 변환을 적용할 수 있습니다.

`u64::from(123)`

프리미티브 타입에서는 타입 간 From 혹은 Tryfrom이 적용되어, 타입 변환을 적용할 수 있습니다.

```rust
struct A {
  data: i32,
}
impl From<i32> for A {
  // 해당 함수를 구현해줘야 합니다.
  fn from(data: i32) -> Self {
    A { data }
  }
}
let a1: A = 123.into();
let a2 = A::from(456);
```

> `impl From<i32> for A`를 적용하면 자동으로 `impl Into<A> for i32`가 구현됩니다.

### std::convert::TryFrom

실패할 수 있는 타입 변환에 사용되며, 제네릭 타입 중 i32를 u32로 바꾸거나 하는 등의, 오류가 발생할 수 있는 변환에 적용되어 있습니다.

```rust
struct A {
  data: i32,
}
impl TryFrom<i32> for A {
  // 변환에 실패할 시 다음 타입을 반환합니다.
  type Error = ();
  // 해당 함수를 구현해줘야 합니다.
  fn try_from(value: i32) -> Result<Self, Self::Error> {
    Ok(A { data: value })
  }
}
let a1: A = 123.try_into().unwrap();
let a2 = A::try_from(456).unwrap();
```

> 오류를 체크하기 위해 다른 타입 변환보다 속도가 느려질 수 있습니다.

### std::convert::Into

A타입의 값을 B타입으로 암시적으로 변환하기 위해 Into를 구현해줍니다.

`impl Into<B> for A`

해당 구현은 From을 구현할 시 자동으로 구현됩니다.

```rust
struct A {
  data: i32,
}
impl Into<A> for i32 {
  // 해당 함수를 구현해줘야 합니다.
  fn into(self) -> A {
    A { data: self }
  }
}
let a1: A = 123.into();

// 보통 함수에서 A타입의 인자로 받기 위해, 제네릭 형태로 사용해줍니다.
fn a2<T>(data: T)
where
  T: Into<A>,
{
  let data: A = data.into();
}
```

### std::convert::TryInto

`TryFrom`과 마찬가지로, 오류가 발생할 수 있는 타입 변환을 할 때 사용됩니다.

```rust
struct A {
  data: i32,
}
impl TryInto<A> for i32 {
  // 변환에 실패할 시 다음 타입을 반환합니다.
  type Error = ();
  // 해당 함수를 구현해줘야 ㅎ바니다.
  fn try_into(self) -> Result<A, Self::Error> {
    Ok(A { data: self })
  }
}
```

> 위와 같은 방법을 이용하여 프리미티브 타입 별 변환을 사용할 수 있으며, 여러 구조체 별 변환을 적용하고 구현할 수 있습니다.
>
>> 그 외로, AsRef나 AsMut키워드를 통해 암시적으로 해당 타입으로 변환할 수 있다는 글을 보았지만, 해당 기능을 사용해 본 적은 없다.
