---
layout: post
title: Rust 자유로운 match 사용법
---


Rust의 문법에는 Match가 있어, C나 C++의 switch case 처럼 사용할 수 있습니다.

하지만 Rust의 Match문법은 Go의 Match문법과 다르게, 주어진 값에 대한 매칭만 가능하도록 되어있습니다.

```rust
// 러스트에서는 다음과 같은 문법만 사용가능합니다.
match "string" {
  "integer" => {}
  "string" => {}
  _ => {}
}
```

Rust에서는 위에처럼, 특정한 값을 넣고, 해당 값에 대한 매칭만 진행할 수 있는데, Rust 안의 void와 마찬가지인 ()을 이용하여 확장된 match를 사용할 수 있습니다.

```rust
match () {
  // 해당 구문이 실행됩니다.
  () if "string" == "string" => {}
  // 해당 구문으로도 실행할 수 있습니다.
  _ if "string" == "string" => {}
  // 아래 구문은 default와 비슷합니다.
  _ => {}
}
```

위의 구문 없이, if와 else의 중첩만으로 위를 구현할 수 있지만, match를 사용하면 여러 분기에 대해 깔끔하게 정리할 수 있기 때문에, 해당방법을 사용하여 여러 변수에 대한 분기문 처리를 깔끔하게 유지할 수 있습니다.
