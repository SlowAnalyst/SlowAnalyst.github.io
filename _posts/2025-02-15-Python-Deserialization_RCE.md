---
layout: post
title: "파이썬 역직렬화 원격 코드 실행 취약점"
summary: "파이썬 역역직렬화 원격 코드 실행 취약점 분석"
author: eveheeero
date: '2025-02-15 15:17:00 +0900'
category: ['etc', 'learning']
tags: rce
thumbnail: /assets/img/posts/default.png
keywords: rce, python, pickle
usemathjax: false
permalink: /blog/python-deserialization-rce/
---


## 배경

리버싱 CTF문제를 푸는 도중 `pickle.load`를 실행할 때 특정 코드가 실행되는 문제가 있어, 해당 문제를 푸는 중 발견한 내용을 요약하게 되었습니다.

## 요약

- 파이썬의 역직렬화중 클래스의 `__reduce__`에서 반환된 함수가 호출되므로, 이를 이용해 역직렬화를 시도하는 컴퓨터에서 임의의 코드를 실행할 수 있음
- 역직렬화시 어떤 코드가 실행될지는 `python -m pickletools <file>`명령어 혹은 `pickletools.dis`메소드를 이용해 확인할 수 있음

## 분석 과정

`__reduce__` 메소드를 포함하는 클래스를 생성하여 직렬화, 역직렬화시 어떤 코드가 실행되는지 분석함

```python
import pickle
import pickletools


def temp_callback(a):
    print("callback called")
    return "callback generated string"


class A:
    def temp_method(self):
        print(123)

    def __reduce__(self):
        print("reduce inner string")
        return (temp_callback, (v1,))


v1 = "12356"

with open("picklefile", "wb") as f:
    pickle.dump(A(), f)

print("dumped")

with open("picklefile", "rb") as f:
    pickletools.dis(f)

print("disassembled")

with open("picklefile", "rb") as f:
    result = pickle.load(f)
    print("unpickled")
    print(result)
```

위와 같은 코드를 실행했을때 아래와 같은 결과가 나옴

```text
reduce inner string
dumped
    0: \x80 PROTO      4
    2: \x95 FRAME      42
   11: \x8c SHORT_BINUNICODE '__main__'
   21: \x94 MEMOIZE    (as 0)
   22: \x8c SHORT_BINUNICODE 'temp_callback'
   37: \x94 MEMOIZE    (as 1)
   38: \x93 STACK_GLOBAL
   39: \x94 MEMOIZE    (as 2)
   40: \x8c SHORT_BINUNICODE '12356'
   47: \x94 MEMOIZE    (as 3)
   48: \x85 TUPLE1
   49: \x94 MEMOIZE    (as 4)
   50: R    REDUCE
   51: \x94 MEMOIZE    (as 5)
   52: .    STOP
highest protocol among opcodes = 4
disassembled
callback called
unpickled
callback generated string
```

## 알아낸 내용

- 직렬화시에는 `__reduce__` 내부의 코드가 실행된 후 `호출할 메소드 명` 및 `호출할 메소드의 인자`가 파일로 담기게 됨
- 역직렬화시에는 `호출할 메소드 명` 및 `호출할 메소드의 인자`가 실행됨, `__reduce__` 내부의 코드는 실행되지 않음, 다른 메소드의 정보는 담기지 않음
- `호출할 메소드 명`에 `os.system` 등의 메소드를 넣음으로써 쉘 코드 실행이 가능하며, `eval`을 넣음으로써 임의의 코드 실행이 가능함
