---
layout: post
title: C/C++ 문자 리터럴
---


> C와 C++의 String 혹은 문자열에 대한 리터럴은 해당 링크에서 확인할 수 있다.

## [마이크로소프트 MSDN](<https://docs.microsoft.com/ko-kr/cpp/cpp/string-and-character-literals-cpp?view=msvc-170>)

문자 리터럴은 문자의 뒤나 앞에 붙어, 문자의 타입을 지정하는것을 도와준다.

간단하게 요약하면, 문자 리터럴은 다음이 있다.

* u8 - 1바이트(8비트) 문자 char8_t형 (char8_t형은 C++20버전 다음에 나왔으며, 그 이전에는 char형이 나타난다.)

* L - 2바이트 혹은 4바이트 wchar_t형 (widechar_type)

* u - 2바이트 char16_t형 (16비트)

* U - 4바이트 char32_t형 (32비트)

* R - 원시 문자 리터럴

* s - String클래스 -> std::string이나 std::u8string, std::wstring, std::u16string, std::u32string

----

해당 리터럴은 프로그래머가 여러 타입으로 문자열을 받고, 입력하는것을 지원한다.

기존의 char형태에서는 기본 ASCII코드 외의 각자 지역의 언어들과 특수문자를 제대로 지원하지 않으며, 이는 utf-8(u8)을 사용하거나 utf-16, utf-32등의 여러 인코딩을 사용하여 표현할 수 있다.

----

원시 문자 리터럴은 "나 \\등의 여러 문자를 입력하는것을 도와준다. 특히 HTML같은 경우 여러가지 특수기호를 사용하게 되는데, 이를 프로그래밍적으로 작성하는것은 매우 힘들다.

이러한 문제를 해결하기 위해선 C++11버전 이후에 나온 원시 문자 리터럴을 사용하여, R"(  )" 내부에 존재하는 모든 특수기호가 이스케이프 시퀀스를 뜻하는게 아닌, 문자열을 뜻하는것이라고 명시할 수 있다.

원시 문자 리터럴은 R"와 ( 사이에 최대 16글자의 사용자 정의 구분자를 사용할 수 있다.

원시 문자 리터럴은 다음과 같이 사용할 수 있다.

```C++
auto Hello = R"MyHelloLiteral(Hell\o)MyHelloLiteral";   // \ 기호가 필요하지 않지만, 이러한 값을 제대로 입력받을 수 있음을 표시하기 위해 적어주었다.
```

위 코드의 Hello 변수는 Hell\\o만을 가르키고 있으며, MyHelloLiteral로 둘러쌓여있는 문자열을 감싸주고 있다.

----

s 리터럴은 입력받은 const char*등의 타입을 std::string형태로 바꿔준다.

이는 다음과 같이 사용할 수 있다.

```C++
std::string MyStr = "This is std::string"s;
std::wstring MyStr2 = L"This is std::string for wide character"s;
std::wstring Mystr3 = LR"(This is "REALLY" std::wstring)"s;
```

----

> 해당 문자 리터럴의 사용은 winAPI관련 문서에서도 쉽게 찾아볼 수 있다.
>
> winAPI는 호환성을 가장 중요시 여기기 때문에 여러 인코딩에 대한 지원을 하여, 프로그래머가 사용할 여러 인코딩에 따라 여러가지 타입의 매개변수를 받을 수 있도록 지원하고 있다.
>
> 간단하게 LoadLibrary함수를 보아도, [LoadLibraryA](<https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibrarya>), [LoadLibraryW](<https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibraryw>) 로 다양한 인코딩 형식에 대해 지원하는것을 볼 수 있다.
>
> 추가적으로 wchar_t*형태를 char*타입으로 변환하는 방법은 stdlib.h의 wcstombs(char *dest, wchar_t *source, size_t count)함수를 사용하면 된다. [IBM](<https://www.ibm.com/docs/ko/i/7.3?topic=functions-wcstombs-convert-wide-character-string-multibyte-string>) [MSDN](<https://docs.microsoft.com/ko-kr/cpp/c-runtime-library/reference/wcstombs-wcstombs-l?view=msvc-170>)
>
> 혹은 안전을 위해 사용할 수 없다면 MSDN의 wcstombs_s을 사용할 수 있다. [MSDN](<https://docs.microsoft.com/ko-kr/cpp/c-runtime-library/reference/wcstombs-s-wcstombs-s-l?view=msvc-170>)
>
> wcstombs_s(size_t *pReturnValue, char *dest, size_t sizeOfDest, wchar_t *source, size_t size); pReturnValue는 반환된 문자열의 크기를 넣기 위한 포인터이다.
>
> wcstombs -> wide character stream to multi bytes stream(sequence)인것같다?
