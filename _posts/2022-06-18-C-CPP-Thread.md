---
layout: post
title: C/C++ 스레드 프로그래밍
---

> C++에서의 스레드는 \<thread\>를 사용해서 사용할 수 있다.
>
> > 리눅스에서 스레드 관련 프로그램 컴파일을 위해서 -lpthread 옵션이 필요할 수 있다.

---

### 스레드는 다음과 같은 방법으로 사용할 수 있다.

```C++
std::thread ThreadName(ThreadFunc, arg1, arg2, ...);
```

이후 ThreadName.join()을 이용해 스레드가 끝날때까지 기다릴 수 있다.

---

### 스레드는 클래스 생성을 통해 정의해줘야 하는데, 가변적 길이의 클래스를 정의하기 위해서는 다음과 같은 방법을 사용할 수 있다

```C++
std::vector<std::thread> Threads;
Threads.push_back(std::thread(ThreadFunc, arg1, arg2, ...));
Threads[0].join();
```

---

### 클래스 내부에 있는 메소드를 클래스 함수로 사용하기 위해 다음과 같은 방법을 사용할 수 있다

```C++
std::thread(&myClass::Threadmethod, this, arg1, arg2, ...);
// or
typedef 리턴타입 (myFunc::*MethodPointerType)(arg1type, arg2type, ...);
MethodPointerType MethodPointer = &myClass::Threadmethod;
std::thread(MethodPointer, this, arg1, arg2, ...);
```

이때 thread의 두번째 인자에 this를 적어줘야 한다.

---

### 스레드 함수에 레퍼런스 인자를 두고 싶으면 다음과 같은 방법을 사용할 수 있다

```C++
std::thread(ThreadFunc, std::ref(arg1), std::ref(arg2), ...);
```

### 스레드 함수의 리턴값을 받고 싶으면 다음과 같은 방법을 사용할 수 있다

> \<future\>헤더의 기능을 사용하여 스레드의 결과값을 받아올 수 있다.

```C++
std::future<리턴타입> var = std::async(ThreadFunc, arg1, arg2, ...);
var.get(); // 결과를 반환한다. 반환할때까지 실행함.
// or
std::promise<리턴타입> promise;
std::future<리턴타입> var = promise.get_future();
std::thread thread(ThreadFunc, std::move(promise), args...);
thread.join();  // 종료할때까지 기다린다.
var.get(); // 결과를 받아온다.
```

아래의 방법에서는 스레드 반환 타입은 void이지만, 스레드 내부에서 인자로 promise를 받고, promise의 변수 값을 설정해주는 방법이다. 그냥 레퍼런스 변수를 사용하는 방법과 비슷하다.

여러 값을 다양한 방법을 통해 가져올 수 있지만, 인자를 수정해주고 직접 다뤄줘야 하는 불편함이 있다.

> 여러 값을 받아올 수 있지만 차라리 std::tuple을 통해 여러 값을 반환하는 방법이 나을 듯 하다.
