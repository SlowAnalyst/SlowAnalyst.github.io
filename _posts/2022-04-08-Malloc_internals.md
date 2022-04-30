---
layout: post
title: Malloc 내부를 통해 보는 tcache의 동작
---

> 개요
>
> MallocInternals 번역
>
>> Malloc의 개요
>>
>> Chunk는 무엇인가?
>>
>> Arenas와 Heaps
>>
>> Thread Local Cache (tcache)
>
> Thread Local Cache (tcache)의 동작
>
> References

# 개요
 본 글은 [1]의 일부를 번역하여 malloc 함수가 메모리를 어떤 형태로 다루는지와
tcache의 동작을 일부 살펴본다.
 
# MallocInternals 번역
 여기서는 [1]의 일부를 번역한 내용을 다룬다.
## Malloc의 개요
 GNU C Library (glibc의) 의 malloc 라이브러리는 어플리케이션의 주소 공간에
할당된 메모리를 관리하는 다양한 함수들을 포함한다. Glibc malloc은
ptmalloc (pthreads malloc) 으로부터 파생되었고, ptmalloc은
dlmalloc (Doug Lea malloc) 으로부터 파생되었다. 이 malloc은 "힙 (heap)"
스타일을 따르고, 그것은 큰 영역의 메모리에 다양한 크기의 청크 (chunks) 가
존재한다는 것을 의미한다. 그리고 이는 예를 들자면, 비트맵과 배열 또는 동일한
크기의 블록 등을 사용하는 구현과 반대이다. 과거에는, 한 어플리케이션마다 하나의
힙만 할당되었지만, glibc의 malloc은 한 어플리케이션에 다수의 힙들이 할당되는
것을 허용한다. 이때 할당된 각각의 힙은 그 주소 공간 내부에서 자라게 된다.

 그럼, 이 문서에서 사용되는 몇 가지 용어를 정의해보자.
* Arena: 하나 또는 그 이상의 스레드가 공유하는, 하나 또는 그 이상의 힙 또는
힙 내부에서 "free"된 청크들의 링크드 리스트를 참조하는 구조체이다. 각 아레나에
연결된 스레드들은 그 아레나의 free 리스트로부터 메모리를 할당받을 것이다.
* Heap: 할당될 청크들로 나누어진 연속적인 영역의 메모리이다. 각 힙은 정확히
하나의 아레나에 속한다.
* Chunk: 할당될 수 있는(어플리케이션이 소유하는) 작은 범위의 메모리로, free될
수 있고 (이때는 glibc가 소유한다), 인접한 청크들과 결합되어 더 큰 범위를 가질
수 있다. 청크는 어플리케이션에게 할당되는 메모리의 블록을 둘러싸는 랩퍼 (wrapper)
라는 것을 기억하자. 각 청크는 하나의 힙에 존재하고 하나의 아레나에 속한다.
* Memory: 일반적으로 램 또는 스왑에 의해 얻어지는 어플리케이션 주소 공간의
비율을 말한다.

 본 문서에서 "메모리 (memory)"라는 용어를 오직 일반적인 용법으로만 사용할
것이다. 한편 glibc malloc 코드에는 리눅스 커널 (또는 다른 OS) 과 작업하는
코드가 있다. 그 코드는 커널에게 어떤 메모리가 매핑되어야 하는지와 어떤 메모리가
커널에게 리턴될 수 있는지에 대한 힌트를 주기 위한 것이다. 이때 "실제 메모리
(real memory)"와 "가상 메모리 (virtual memory)"의 구분은 명시되지 않는
한, 여기서 논하고자 하는 것과 관련이 없다.

## Chunk는 무엇인가?
 Glibc의 malloc은 chunk-oriented이다. 이는 큰 영역의 메모리 ("힙") 를
다양한 크기의 청크로 나눈다. 각 청크는 자신의 크기 (청크 헤더의 size 필드로)
와 인접한 청크들을 메타 데이터로 포함한다. 청크가 어플리케이션에 의해 사용될
때, "저장되는 데이터"는 청크의 크기뿐이다. 청크가 free되면, 어플리케이션의
데이터로 사용되던 부분은 아레나와 관련된 데이터로 채워지도록 재지정된다.
여기서 아레나와 관련된 데이터는 적합한 청크가 빠르게 검색되고, 필요하다면
재사용될 수 있도록, 링크드 리스트에서 사용되는 포인터와 같은 것이다. 또한,
free된 청크의 마지막 워드는 청크 크기가 복사된 것이다 (이때 LSB로부터
세 비트들을 0으로 둘 수도 있고, 플래그로 사용할 수도 있다).

 Malloc 라이브러리 내부에서 "청크 포인터" 또는 mchunkptr은 청크의 시작
주소를 가리키지 않고, 이전 청크의 마지막 워드를 가리킨다 - i.e. mchunkptr의
첫 번째 필드는 free된 이전 청크를 알지 못하는 한 유효하지 않다.

 모든 청크들의 크기는 8 바이트의 배수이므로, 청크 크기의 세 LSB들은
플래그로 사용될 수 있다. 이러한 세 개의 플래그는 다음과 같이 정의된다:
* A (0x04): 할당된 아레나 - 메인 아레나는 어플리케이션의 힙을 쓴다. 그리고
나머지 아레나들은 mmap된 힙들을 사용한다. 청크를 힙에 매핑하기 위해 어떤
경우에 속하는지 알아야 한다. 이 플래그의 비트가 0이면, 청크는 메인 아레나와
메인 힙에 속한다. 이 플래그의 비트가 1이면, 이 청크는 mmap된 메모리에
속하고 청크의 주소를 통해 힙의 위치를 계산할 수 있다.
* M (0x02): MMap된 청크 - 이 청크는 mmap 함수를 호출하여 할당되었고
힙에 속하지 않는다.
* P (0x01): 이전 청크가 사용되고 있음 - 이 플래그가 셋트이면, 이전 청크는
어플리케이션에 의해 사용 중이고, 따라서 prev_size 필드는 유효하지 않다.
단, 몇몇 청크들, 예를 들면 fastbin에 있는 (아래에서 설명) 청크들은
어플리케이션으로부터 free되었다고 해도 이 비트를 셋트한 채로 유지할 수
있다. 이 비트의 실제 의미는 이전 청크가 병합 대상으로 인식돼서는 안된다는
것이다 - 이 청크는 어플리케이션이나 malloc의 코드 맨 위에 있는 최적화
계층에 의해 "사용 중"이다.

 청크의 페이로드 영역이 malloc에서 요구하는 오버헤드를 충족시킬만큼
크다면, 청크의 최소 크기는 4 * sizeof(void *) (size_t가 void *와 같은
크기를 가지지 않는 한) 이다. 이러한 최소 크기는 플랫폼의 ABI가 추가적인
정렬을 요구한다면 더 커질 수 있다. 단, prev_size가 청크의 최소 크기를
5*sizeof(void *)로 증가시키지는 않는다. 왜냐하면 청크가 작다면
bk_nextsize 포인터가 사용되지 않을 것이고, 청크가 충분히 크다면 결국
충분한 공간이 있는 것이기 때문이다.
<pre><code>
        이미 사용 중인 청크
      [  prev_size   ] <- mchunkptr
-     [ size  ][ AMP ]
^     [              ] <- returned by malloc
|     [              ]
chunk [   payload    ]
|     [              ]
v     [              ]
-     [ size  ][ AMP ] P = 1

        free된 청크
      [  prev_size   ] <- mchunkptr
-     [ size  ][ AMP ]
^     [ fwd          ] <- returned by malloc
|     [    bck       ]
chunk [ fd_nextsize  ] -> large chunks only
|     [  bk_nextsize ] -> large chunks only
|     [  ...         ]
v     [  prev_size   ] -> same as size
-     [ size  ][ AMP ] P = 0
</code></pre>
여기서 청크가 메모리에서 서로 인접하므로, 만약 첫 번째 청크 (가장 낮은
주소를 가지는) 의 주소를 안다면, 힙에 존재하는 모든 청크를 size 정보를
이용하여 접근할 수 있다. 하지만 주소를 증가시키는 연산만을 사용해야 하고,
마지막 청크를 감지하는 것은 어려울 수 있다.

 할당된 힙들은 항상 2의 제곱의 주소로 정렬된다. 즉, 청크가 할당된 힙에
존재할 때 (i.e. A 비트가 셋트일 때) 그 힙을 위한 heap_info의 주소는
청크의 주소를 기반으로 계산될 수 있다.
<pre><code>
mchunkptr -> 0x7ffa6b3414123dc0
             |           |max   |
             0x7ffa6b3414000000 -> heap_info *
</code></pre>

## Arenas와 Heaps
 멀티 스레드 어플리케이션을 효율적으로 다루기 위해 glibc의 malloc은
메모리의 다수 영역이 활성화되는 것을 허용한다. 즉, 다른 스레드가 서로 영향을
주지 않으면서 다른 메모리의 영역에 접근할 수 있다. 이러한 메모리의 영역들은
아레나라고 통칭한다. 그리고 "메인 아레나 (main arena)"는 애플리케이션의
초기 힙을 의미한다. Malloc 코드에는 이 아레나를 가리키는 정적 변수가 존재하고,
각각의 아레나에는 추가된 아레나를 연결하기 위한 next 포인터가 있다.

 스레드 충돌로 인한 성능에 대한 압박이 증가함에 따라, 추가적인 아레나는
mmap으로 생성되어 이를 해소한다. 아레나의 개수는 시스템이 가지고 있는
CPU의 개수의 8배로 제한되고 (사용자가 직접 명시하지 않는한, mallopt를 보라),
그것은 스레드가 상당히 많은 프로그램의 경우에는 여전히 성능에 대한 긴장이
있을 것을 의미하지만, 그것에 대한 트레이드 오프 (trade-off)로 적은 파편화
(fragmentation)가 발생할 것이다.

 각 아레나 구조체는 아레나에 대한 접근을 제어하는데 사용되는 뮤텍스 (mutex)를
가진다. 몇몇 연산들, 예를 들면 fastbins에 접근하는 것은, 원자적 연산 (atomic
operations)으로 이루어질 수 있고 아레나를 잠글 (lock) 필요는 없다. 다른 모든
연산들은 스레드가 아레나를 잠글 필요가 있는 것들이다. 뮤텍스로 인해 발생하는
성능에 대한 긴장은 다수의 아레나가 생성되는 이유이다 - 다른 아레나에 접근하는
스레드는 서로를 기다릴 필요가 없는 것이다. 스레드는 성능과 관련된 것이 요구한다면
자동적으로 사용되지 않고 있는 (잠기지 않은, unlocked) 아레나로 전환할 것이다.

 각 아레나는 하나 또는 다수의 힙으로부터 메모리를 얻는다. 메인 아레나는
프로그램의 초기 힙을 (.bss 바로 뒤에서 시작하는 et al) 사용한다. 추가적인
아레나는 메모리를 mmap을 통해 생성한 힙으로부터, 오래된 힙들이 다 사용되는
경우에 추가적인 힙을 리스트에 추가하면서, 얻는다. 각 아레나는 특별한 맨 위에
있는 청크 (top chunk)를 추적하는데 이는 전형적으로 사용가능한 가장 큰 청크이고,
가장 최근에 할당된 힙을 의미하기도 한다.

 할당된 아레나를 위한 메모리는 편의성을 위해 아레나의 초기 힙으로부터 얻어진다.
 
<pre><code>
          Heap #1             Heap #2            Heap #3
	 _               |--------------------|
 	/ [ ar_ptr  ]----+---[ ar_ptr   ]<--+ +-[  ar_ptr    ]
       /  [         ]<---|---[  prev    ]   |---[  prev      ]
    heap  [ prev    ]--+ |   [  size    ]       [  size      ]
    info  [size     ]  | |   [  ....    ]       [   ...      ]
 	\_[ ...     ]  - |   [   chunks ]       [  chunks    ]
	  [ arena   ]<---+                      [            ]
	  [         ]-------------------------->["top" chunk ]
	  [ chunks  ]
	
</code></pre>

각 아레나에서, 청크들은 애플리케이션에 의해 사용 중 (in use)이거나 사용가능
(available, free) 상태일 수 있다. 사용 중인 청크들은 아레나에 의해 추적되지는
않는다. 사용가능한 청크들은 그 크기와 할당 기록에 기반하여 다양한 리스트들에
저장되므로, 라이브러리는 할당 요청에 따른 적합한 청크들을 빠르게 찾을 수 있다.
이 리스트들은, "빈 (bins)" 이라고 여기서 부를 것이다.
* Fast: 작은 청크들은 특정 크기의 (size-specific) bin에 저장된다. Fast
bin ("fastbin")에 추가된 청크들은 인접한 청크와 병합되지 않는다 - 빠른 접근
(이름에서 알 수 있듯이)을 위해 최소의 로직만을 사용한다. Fastbin에 있는
청크들은 필요하다면 다른 bin들로 옮겨질 수 있다. Fastbin 청크들은, 청크들이
모두 같은 크기를 가지고 중간 크기는 절대 접근될 일이 없으므로, 단일
연결 리스트에 저장된다.
* Unsorted: 청크들이 해제되면 (free'd) 그들은, 초기에 한 bin에 저장된다.
그들은 나중에 정렬되고, malloc에서, 이는 빠르게 재사용될 기회를 주기 위한
것이다. 이는 또한 정렬 로직이 어느 한 시점에만 존재하면 된다는 것을 의미한다 -
모두가 해제된 청크를 이 bin에 넣고, 그것들은 나중에 정렬될 것이다. 이러한
"정렬되지 않은 ("unsorted")" bin은 간단히 보통 bins의 첫 번째이다.
* Small: 일반적인 bins는 모든 청크의 크기가 같은 "작은 ("small")" bins와
청크의 크기가 범위를 가지는 "거대한 ("large")" bins로 나뉜다. 청크가 이러한
bins에 추가되면 인접한 청크와 더 거대한 청크를 만들기 위해 "병합"한다. 따라서
이들은 (비록 fast 또는 unsorted 청크, 그리고 사용 중인 청크와는 인접할 수
있다고 하더라도) 다른 청크와 절대 인접할 일이 없다. Small 그리고 large 청크들은
이중으로-연결되며 (doubly-linked) 그렇기에 (그들이 새롭게 해제된 청크와
병합하는 경우와 같이) 중간에서 삭제될 수 있다.
* Large: 한 청크가 속한 bin이 다수의 크기를 포함할 때 그 청크는 "거대하다
("large")". Small bins에서는, 첫 번째 청크를 선택하여 사용하면 된다. Large
bins에서는, "가장 적절한" 청크를 찾아야 하고, 두 청크 (하나는 사용자가 요청한
크기, 하나는 그 나머지)로 나누어야 할 수 있다.

<pre><code>

ar_ptr--->[   mutex   ]  |-->[   chunk   ]  |-->[  chunk  ]
          [ fastbins[]]--+   [    fwd    ]--+   [   fwd   ]
	  [           ]
	  [    top    ]
	  [   bins[]  ]<-| |-->[  chunk  ]<-| |--->[  chunk  ]
       /->[           ]--+-+   [   fwd   ]--+-+    [   fwd   ]--------+ 
      +---[           ]  |-----[   bck   ]  |------[   bck   ]<-------+
     /    [   next    ]                                               |
    /	  [ nextfree  ]                                               |
   /	  [  stats    ]                                               |
  +-------------------------------------------------------------------+
  +-------------------------------------------------------------------+
</code></pre>
주의: 위 (와 아래)의 다이어그램에서, 모든 포인터는 "chunk"(mchunkptr)를 가리킨다.
Bins는 청크가 아니므로 (그들은 fwd/bck의 배열이다), 한 hack이 사용되어
mchunkptr의 fwd, bck 필드가 적합한 bin에 접근하는데 사용될 수 있도록 bins를
"그저 적절히" 덮어쓰는 mchunkptr을 chunk-like 객체에 제공한다.

Large 청크에서 "가장 적절한 것("best fit")"을 찾아야 하기 때문에, large 청크들은
추가적인 이중으로-연결된 리스트를 가지며 이는 리스트에서 각 크기 필드를 연결하고,
청크들은 크기에 따라 큰 것에서 작은 것으로 정렬된다. 이는 충분한 크기를 가지는
첫 번째 청크를 malloc이 빠르게 검색할 수 있도록 만든다. 이때 주어진 크기에서 다수의
청크가 존재한다면, 두 번째 청크가 일반적으로 선택되는데, 그 이유는 다음 크기의
연결 리스트가 수정될 필요가 없도록 만들기 위함이다.

<pre><code>

[  ...  ]  |-->[ size = 132 ]<-+ [ size = 132 ] +->[ size = 120  ]
[ bins[]]--+   [    fwd     ]--|>[    fwd     ]-|->[    fwd      ]
[       ]<-----[    bck     ]<-|-[    bck     ]<|--[    bck      ]
[  ...  ] +--->[ fd_nextsize]--|----------------+  [ fd_nextsize ]---+
         /  +--[ bk_nextsize]  +-------------------[ bk_nextsize ]<-+|
        /  /                                                        ||
       /  +---------------------------------------------------------+|
      +--------------------------------------------------------------+

</code></pre>

## Thread Local Cache (tcache)
 이 malloc은 멀티 스레드가 있음을 인식하지만 그 인식은 생각보다 그 범위가 넓다 -
이는 멀티 스레드가 동작 중인지 아닌지를 인식할 수 있다. 이 malloc에는 NUMA
아키텍처에 대한 최적화, thread locality에 대한 coordinate, 코어에 따른
스레드 정렬 등에 대한 코드는 없다. Malloc은 이러한 문제들은 커널이 충분히 잘
다룰 것이라고 가정한다.

 각 스레드에는 스레드-지역 변수가 존재하여 마지막으로 사용한 아레나를 기억한다.
만약 그 스레드가 그 아레나를 사용하고자 할 때, 그 아레나가 이미 사용 중이라면
스레드는 그 아레나가 사용가능할 때까지 기다려야 한다. 만약 스레드가 이전에
사용한 아레나가 없다면, 새로운 아레나를 생성하거나, 전역 리스트에서 다음
아레나를 선택할 수도 있다.

 각 스레드는 스레드 별 캐시 (per-thread cache, tcache라고 불린다)가 존재하며
이는 작은 청크의 모음으로 스레드가 아레나를 잠글 필요 없이 접근할 수 있다. 이
청크들은 fastbins와 같이 단일 연결 리스트의 형태로 저장되지만, 링크들은
청크의 헤더가 아닌 페이로드 (사용자 데이터 영역)를 가리킨다. 각 bin은 한 크기의
청크를 포함하기 때문에, 그 배열은 청크의 크기로 (간접적으로) 인덱싱될 수 있다.
Fastbins와는 다르게, tcache는 각 bin이 허용하는 청크의 개수가 정해져 있다
(tcache_count). 만약 tcache bin이 주어진 요청 크기에 대해 비어있다면, 그 다음
크기의 청크가 사용되지는 않고 (내부 파편화를 발생시킬 수 있으므로), 대신 일반적인
malloc 루틴이 동작하기 시작한다. 즉, 스레드의 아레나를 잠그고 작업을 시작하는
것이다.

<pre><code>

entries[]
[          ]       [  size = 48  ]       [   size = 48   ]
[          ]------>[     next    ]------>[     next      ]--+
[          ]       [   .....     ]       [   .......     ]  |
[  ...     ]       [   .....     ]       [   .......     ]  -
                   \                                     /
                    \__tcache_count(TCACHE_FILL_COUNT)__/
					
counts[]    _
[          ] \
[   2      ]  \
[          ]   tcache_bins(TCACHE_MAX_BINS)
[          ]  /
[  ...     ]_/

</code></pre>

# Thread Local Cache (tcache)의 동작
 [2, p. 1090]은 '--disable-experimental-malloc' 옵션을 설명하면서
스레드 별 캐시 (tcache)가 malloc에서 기본값으로 허용됨을 명시하였다.

 [3]은 다음과 같은 코드를 통해 tcache를 관리하는 구조체가
tcache_perthread_struct이고, tcache 또한 malloc으로 할당됨을 보인다.

```C
/* There is one of these for each thread, which contains the
   per-thread cache (hence "tcache_perthread_struct").  Keeping
   overall size low is mildly important.  Note that COUNTS and ENTRIES
   are redundant (we could have just counted the linked list each
   time), this is for performance reasons.  */
typedef struct tcache_perthread_struct
{
  uint16_t counts[TCACHE_MAX_BINS];
  tcache_entry *entries[TCACHE_MAX_BINS];
} tcache_perthread_struct;
```

```C
static void
tcache_init(void)
{
  mstate ar_ptr;
  void *victim = 0;
  const size_t bytes = sizeof (tcache_perthread_struct);

  if (tcache_shutting_down)
    return;

  arena_get (ar_ptr, bytes);
  victim = _int_malloc (ar_ptr, bytes);
  if (!victim && ar_ptr != NULL)
    {
      ar_ptr = arena_get_retry (ar_ptr, bytes);
      victim = _int_malloc (ar_ptr, bytes);
    }


  if (ar_ptr != NULL)
    __libc_lock_unlock (ar_ptr->mutex);

  /* In a low memory situation, we may not be able to allocate memory
     - in which case, we just keep trying later.  However, we
     typically do this very early, so either there is sufficient
     memory, or there isn't enough memory to do non-trivial
     allocations anyway.  */
  if (victim)
    {
      tcache = (tcache_perthread_struct *) victim;
      memset (tcache, 0, sizeof (tcache_perthread_struct));
    }

}
```

 Tcache는 기본값으로 허용되므로 애플리케이션에서 malloc을 통해 메모리를 요청하면
먼저 tcache를 설정하는 작업이 진행되고 tcache에 저장된 청크를 할당한다.
이때 tcache는 초기에 malloc을 통해 할당받아 존재하기 때문에 힙 영역에서
상대적으로 낮은 주소에 위치하게 된다.

# References
[1] Carlos Donell et al., "MallocInternals", glibc wiki, 2022.
[Online]. Available: https://sourceware.org/glibc/wiki/MallocInternals,
[Accessed Apr. 08, 2022]

[2] Sandra Loosemore with Richard M. Stallman, et al., "The GNU
C Library Reference Manual", GNU Press a division of the Free
Software Foundation, 2021

[3] "malloc.c", GNU & Elixir, 2021. [Online]. Available:
https://elixir.bootlin.com/glibc/glibc-2.34.9000/source/malloc/malloc.c,
[Accessed Apr. 08, 2022]
