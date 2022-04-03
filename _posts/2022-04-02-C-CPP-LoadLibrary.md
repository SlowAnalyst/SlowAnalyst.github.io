---
layout: post
title: C/C++ LoadLibrary
---


# LoadLibrary는 윈도우에서 사용자가 만든 DLL파일을 불러오는 함수이다.

LoadLibrary를 사용하여 동적 라이브러리인 DLL파일을 불러와, 내부에 존재하는 함수를 사용할 수 있다.

아래는 DLL 사용 예제이다.

![DLL을 불러와 GetProcAddress를 이용하여 함수를 불러오는 예시](<https://eveheeero-github-io.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc896f0f7-7421-45f3-9ac2-e79472cf5ec6%2FUntitled.png?table=block&id=48c9d0d3-0b0d-472b-b8e2-5694c8861fca&spaceId=c2eb73c4-6260-4fb7-8470-2e07bff25e55&width=2000&userId=&cache=v2> "DLL을 불러와 GetProcAddress를 이용하여 함수를 불러오는 예시")

LoadLibrary로 dll파일의 주소를 적어줘, DLL을 로드하고 메모리에 매핑된 주소를 불러온다.

이후, 함수의 원형(반환값과 인자값)을 이용하여 타입을 지정한 뒤, 해당 타입을 이용하여 함수를 불러올 수 있다. (함수는 기본적으로 주소를 가지고 있는 포인터변수이다. 이에 괄호를 통해 인자를 주거나, 인자를 주지 않으면 직접 실행하도록 되는것이다.)

함수의 주소를 받는것은 GetProcAddress를 이용해 받을 수 있다. 인자로 모듈주소와 모듈이름을 주면 인자를 반환한다. 모듈 이름을 모를경우 DLL정보를 보는 프로그램을 이용할 수 있다.

> 함수의 이름을 알 수 없는경우 함수 내부 번호를 이용해 MAKEINTRESOURCE(ordinal)을 대신 넣어주면 되는 것 같다. 확인필요

![DLL을 사용한 함수의 결과값](<https://eveheeero-github-io.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F9e0c9957-096d-4956-ab97-e7be87c73fe5%2FUntitled.png?table=block&id=5f320ab0-be6b-49be-9059-83f07cdef242&spaceId=c2eb73c4-6260-4fb7-8470-2e07bff25e55&width=2000&userId=&cache=v22> "DLL을 사용한 함수의 결과값")

추가적으로 검색하다 알게 된 내용인데 다음과 같은 방법으로도 함수를 불러올 수 있는 모양이다.

![typedef 없이 함수를 지정하는 방법](<https://eveheeero-github-io.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fb2afa580-46f6-40e9-84c6-9707082e350e%2FUntitled.png?table=block&id=bf40fc08-4b64-469b-b831-381d06a3ee05&spaceId=c2eb73c4-6260-4fb7-8470-2e07bff25e55&width=2000&userId=&cache=v2> "typedef 없이 함수를 지정하는 방법")

그리고 과거에 만들었던 코드에서 찾아보니 다음과 같은 코드도 유효했다.

![\_\_stdcall을 이용한 함수호출](<https://eveheeero-github-io.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F443b10c6-e8ac-4949-a27f-cd829f29bed3%2FUntitled.png?table=block&id=e778ff9b-dee8-442e-b3aa-082d060245a6&spaceId=c2eb73c4-6260-4fb7-8470-2e07bff25e55&width=1680&userId=&cache=v2> "\_\_stdcall을 이용한 함수호출")

위의 두 경우에도 결과값은 잘 나온다.

![위의 두 코드에 대한 결과값](<https://eveheeero-github-io.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F70fb3d51-5193-4cd4-9f44-f66a4660d5de%2FUntitled.png?table=block&id=f5f012a0-4cd4-4990-b12f-922904b423bc&spaceId=c2eb73c4-6260-4fb7-8470-2e07bff25e55&width=180&userId=&cache=v2> "위의 두 코드에 대한 결과값")

추가적으로 인자에 괴상한 값이나 콜백함수를 줘야할 때는 다음과 같은 방법을 사용하면 된다.

![과거에 작성했던 코드](<https://eveheeero-github-io.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fdf4b916d-a446-4a0b-8dfc-db51ae7bf66b%2FUntitled.png?table=block&id=a1224a01-5694-4fe1-85c8-5ef477f5712f&spaceId=c2eb73c4-6260-4fb7-8470-2e07bff25e55&width=1720&userId=&cache=v2> "과거에 작성했던 코드")

다음은 소스코드이다.

```CPP
#include <iostream>     // __stdcall은 함수를 끝낼 때 스택정리를 하지 않는것, __cdecl방식은 함수를 끝낼 때 스택정리를 하는것
#include <windows.h>

int main (int argc, char **argv) {
    HMODULE hmodule = LoadLibrary("makeDll.dll");    // DLL을 로드한다.
    if (hmodule == nullptr) {                       // 만약 제대로 로드되지 않았으면 종료한다.
        std::cout << "DLL 로드에 실패하였습니다." << std::endl;
        return 1;
    }
    // 불러온 라이브러리에서 "add"함수를 찾아 주소를 가져온다.
    // 함수에 대한 타입을 지정해준다. (반환값과 인자값때문에 반드시 지정해줘야한다.)
    typedef int (*MyType1)(int, int);   // 반환형이 int인, 인자를 int, int로 주는 함수의 타입이다.
    MyType1 add = (MyType1)GetProcAddress(hmodule, "add");
    int result = add(10, 20); // 불러온 함수를 다음과 같은 방법으로 사용하면 된다.

    std::cout << "result is " << result << std::endl;

    PVOID AddPointer = reinterpret_cast<void*>(GetProcAddress(hmodule, "add")); // getProcAddress는 단순히 주소를 반환하는것이기 때문에 해당 코드처럼 포인터를 저장해놔도 된다.
    // (PVOID는 void*이다.)
    int(__cdecl* add2)(int, int) = (int(__cdecl*)(int, int))AddPointer;   // __cdecl*방식으로 해당 포인터를 받아 할당하는 방법도 있다.
    result = add2(30, 40);

    std::cout << "result is " << result << std::endl;

    typedef int (__stdcall *MyType2)(int, int); // __stdcall방식으로 타입을 지정해도 된다.
    MyType2 add3 = (MyType1)GetProcAddress(hmodule, "add");
    result = add(5, 10);    // 그리고 이런 방식으로 호출한것은 __cdecl방법으로 적용되는진 모르겠다. 디버깅이 필요하다.

    std::cout << "result is " << result << std::endl;

    return 0;
}

```

> \_\_stdcall과 \_\_cdecl에 대한것을 오랫만에 보아 찾아보았다. \_\_stdcall은 표준 호출이고, \_\_cdecl은 c 선언이라는 듯 같다.
