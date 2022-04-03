---
layout: post
title: Format String, and Ret2libc in AMD64
---

> Format String
>
> Ret2libc
>
> Reference

# Format String
 [5]는 형식 문자열 취약점을 사용할 수 있게 된다면, 공격자가 원하는 곳에
어떤 값이든 쓸 수 있는 길로 안내한다고 설명하고 있다. 즉, 임의 메모리
쓰기가 가능한 것이다. 여기서는 AMD64 시스템에서 형식 문자열 취약점이
발생할 수 있는 이유를 설명할 것이다.

 [1, p. 286]은 템플릿 (또는 형식 문자열) 내에 있는 형식 특정자의 동작을
기술한다. 그리고 그것은 형식 특정자가 형식 문자열 다음에 나오는
호출 매개 변수들이 포맷된 후에 출력되도록 만든다는 것이다. 여기서 '포맷된
후'라는 것은 형식 특정자가 해석되어 출력할 호출 매개 변수의 값과 타입이
추출되고 주어진 형식에 맞도록 처리하는 연산들이 모두 수행된 이후를 의미한다.
한편 [2, p. 54]에서는 x86과 AMD64의 매개 변수 전달 방식이 다르기
때문에 가변 인자의 구현이 x86에서 자주 사용되던 방식과는 같을 수 없음을
언급하고 있지만, [2, pp. 57-58]에서 매개 변수의 타입이 레지스터에 배치될 수
없는 경우에는 스택에 배치되는 동작을 서술하고 있다.

 형식 문자열 취약점은 대표적으로 다음과 같은 코드가 컴파일되었을 때 발생할 수
있다.
```C
...
char buf[1024];

scanf("%s", buf);
printf(buf);
...
```
그리고 위 코드를 포함해서 형식 문자열 취약점을 발생시키는 코드는 출력 함수가
기대하는 정도의 호출 매개 변수가 없다. 즉, 사용자 입력에
형식 특정자가 포함되었을 때, 그 특정자에 대응되는 호출 매개 변수가 없는
상황인 것이다. 그러므로 형식 특정자에 대응되는 호출 매개 변수의 타입을
특정할 수 없으므로 레지스터에 배치되는 호출 매개 변수는 존재하지 않는다.
따라서 스택으로부터 호출 매개 변수의 값을 얻게된다. 이는 공격자가
스택의 메모리 값을 읽어들일 수 있고, 임의의 메모리 값을 읽어올 수 있음을
의미한다.

# Ret2libc
 Ret2libc (Return-to-Libc) 공격은 [3]에서 서술하듯이, 실행 불가능 스택을
우회하기 위해 주로 사용되고, x86과 같은 시스템에서는 다음과 같은 페이로드를
사용하여 수행하게 된다.
<pre><code>
<- 스택이 자라는 방향
   주소가 자라는 방향 ->
 -----------------------------------------------------------------
 | buffer fill-up(*) | function_in_lib | dummmy_int32 | arg_1 | arg_2 |
 -----------------------------------------------------------------
</code></pre>
위 그림에서 dummy_int32는 [4, p. 36]에서 제시한 표준 스택 프레임과
[4, pp. 39-40]에서 설명한 함수 프롤로그와 에필로그에 비추어보면, 호출된
함수 시점에서 리턴 주소가 된다.

 그러나 AMD64 시스템에서는 [2, pp. 19-24]가 설명하듯이 매개 변수를 몇몇
클래스로 나누고 클래스에 따라 다른 레지스터에 배치하여 전달하기 때문에
위 그림과 같이 페이로드를 구성할 수 없다. [2, p. 23]은 호출 매개 변수가
배치되는 레지스터의 순서를 다음과 같이 소개한다.
<pre><code>
_______________________________________________________
%rdi | 첫 번째 호출 매개 변수
%rsi | 두 번째 호출 매개 변수
%rdx | 세 번째 호출 매개 변수
%rcx | 네 번째 호출 매개 변수
%r8  | 다섯 번째 호출 매개 변수
%r9  | 여섯 번째 호출 매개 변수
_______________________________________________________
</code></pre>
그러므로 호출 매개 변수의 순서에 대응되는 레지스터에 값을 대입하는 작은
명령어 조각이 필요하고, 이것은 가젯이라고 불린다. 즉, 가젯으로 호출 매개
변수의 값을 그에 대응되는 레지스터에 대입하고 호출하고자 하는 함수로
분기하도록 스택을 구성하는 페이로드를 작성해야 한다. 그러나 [2, p. 18]이
제시하는 스택 프레임 레이아웃과 [4, p. 36]이 제시하는 스택 프레임
레이아웃은 정렬 단위를 제외하고는 같으므로 호출된 함수 시점의 리턴 주소의
위치는 x86에서의 것과 동일하다.

그럼 가젯은 어떻게 찾을 수 있을까? 가젯은 상기 서술하였듯이 작은 명령어
조각이므로 ELF 파일 또는 링크된 동적 라이브러리의 오브젝트 코드에서
동일한 조각을 찾아야 한다. 예를 들어 다음은 한 실행 파일을 objdump로
출력한 결과이다. pop rdi; ret라는 가젯을 찾을 수 있겠는가?
```assembly
...
 40090a:    5b        pop %rbx
 40090b:    5d        pop %rbp
 40090c:    41 5c     pop %r12
 40090e:    41 5d     pop %r13
 400910:    41 5e     pop %r14
 400912:    41 5f     pop %r15
 400914:    c3        retq
...
```
위 어셈블리 코드를 보았을 때는 여기서 얻고자 하는 가젯인 pop rdi; ret는
없어 보인다. 그러나 실제로 어셈블리 코드가 실행되는 것은 아니라는 것을
명심하자. 실제로 CPU의 동작을 결정하는 것은 헥스 코드로 표현되는 기계어이다.
그리고 pop rdi; ret에 대한 기계어는 위 실행 파일에서는 5f c3이다. 즉,
0x400912부터 보면 pop %r15; ret이지만, 0x400913부터 보면 pop rdi; ret가
되는 것이다. 정리하면, 실행 파일 또는 동적으로 링크된 라이브러리에서 가젯에
대응되는 기계어를 검색하여 가젯을 찾을 수 있다.

이제 다음 페이로드를 통해 전체적인 동작을 살펴보자.
<pre><code>
[ buffer fill-up(*)     ]
[ Gadget (pop rdi; ret) ] -> return address of a caller
[ GOT entry of puts()   ]
[ PLT entry of puts()   ]
[ main() address        ] -> return address of a callee
</code></pre>
먼저 함수가 리턴하는 시점에서 스택 포인터는 가젯을 가리키게 되어 명령
포인터는 스택에서 꺼내어진 가젯 주소를 저장하게 된다. 이때 스택은 다음과
같은 상태가 된다.
<pre><code>
(LOW)
...
[ buffer fill-up(*)     ]
[ Gadget (pop rdi; ret) ]
[ GOT entry of puts()   ] <= stack pointer
[ PLT entry of puts()   ]
[ main() address        ]
...
(HIGH)
</code></pre>
이제 가젯의 pop rdi;가 실행되어 puts() 함수의 GOT 엔트리 주소가 스택에서
꺼내어져 %rdi 레지스터에 저장된다.
<pre><code>
(LOW)
...
[ buffer fill-up(*)     ]
[ Gadget (pop rdi; ret) ]
[ GOT entry of puts()   ]
[ PLT entry of puts()   ] <= stack pointer
[ main() address        ]
...
(HIGH)
</code></pre>
그다음으로 가젯의 ret가 실행된다. 따라서 puts() 함수의 PLT 엔트리 주소가
스택에서 꺼내어져 명령 포인터에 저장된다.
<pre><code>
(LOW)
...
[ buffer fill-up(*)     ]
[ Gadget (pop rdi; ret) ]
[ GOT entry of puts()   ]
[ PLT entry of puts()   ]
[ main() address        ] <= stack pointer
...
(HIGH)
</code></pre>
마지막으로 스택을 정리하며 puts() 함수로 점프하여 puts() 함수를 위한
스택 프레임을 구성한다. 여기서 main() 함수의 주소는 [2, p. 18]이
제시한 스택 프레임 레이아웃에 비추어보면 리턴 주소가 된다.
<pre><code>
(LOW)
...
[                       ]
[                       ]
[                       ]
[                       ] <= base pointer, stack pointer
[ main() address        ] <= return address
...
(HIGH)
</code></pre>

# Reference
[1] Sandra Loosemore et al., The GNU C Library Reference
Manual, GNU Press a division of the Free Software Foundation,
2021

[2] H.J. Lu et al., System V Application Binary Interface
AMD64 Architecture Processor Suplement, 2018

[3] Nergal, "The advanced return-into-lib(c) exploits: PaX case
study", "http://phrack.org/issues/58/4.html", Phrack Volume 11
Issue 58 Phile 4, 2001

[4] System V Application Binary Interface IntelI386 Architecture
Processor Supplement Fourth Edition, The Santa Cruz Operation, Inc.,
1997

[5] gera, "Advances in format string exploitation",
"http://phrack.org/issues/59/7.html", Phrack Volume 11 Issue 59
Phile 7, 2002
