---
layout: post
title: "Introduction To Implementation Of The SLUB Allocator Via Socket Buffer Allocation And Deallocation"
summary: ""
author: david232818
category: ['learning']
thumbnail: /assets/img/posts/default.png
keywords: SLUB Allocator
usemathjax: false
permalink: /blog/Introduction-To-Implementation-Of-The-SLUB-Allocator-Via-Socket-Buffer-Allocation-And-Deallocation
---


Contents
--------
1. Introduction
2. Per-CPU
3. NUMA Node
4. Kernel Memory Cache and Socket Buffer
5. SLUB allocator and SKB allocation
6. SLUB allocator and SKB deallocation
7. References



## Introduction

Jeff Bonwick이 SLAB allocator를 소개하고 Christoph Lameter가 SLUB allocator를 대안으로 제시한 이후로 현대 리눅스 커널에는 SLUB allocator가 주로 사용되고 있다. 본 글에서는 리눅스 커널 5.15.30 버전을 기준으로 SKB (`struct sk_buff`)의 할당과 해제를 통해 SLUB allocator 구현을 살펴볼 것이다[1, 2].

## Per-CPU

Symmetric multiprocessing system에서 다수의 CPU들이 핸들링하게 되는 데이터는 락으로 인한 병목이나 cache line bouncing 등을 초래할 수 있다. 이에 리눅스 커널 등은 각 CPU local에 가용한 데이터 공간을 두어 성능 저하를 최소화한다. 즉, CPU 마다 lockless하게 접근할 수 있는 공간을 두는 것이다.



리눅스 커널은 CPU에 접근하기 위한 인터페이스로 `per_cpu_ptr()`나 `raw_cpu_ptr()` 등을 제공한다. 이들은 다음과 같은 call trace를 통해 접근할 수 있다. (`raw_cpu_ptr()`의 call trace가 `SHIFT_PERCPU_PTR()`에서 겹침을 확인하라)

```
/* function call trace */

per_cpu_ptr(ptr, cpu)                                 raw_cpu_ptr(ptr)
SHIFT_PERCPU_PTR() /* __p = ptr,                      arch_raw_cpu_ptr()
           __offset = per_cpu_offset((cpu)) */        SHIFT_PERCPU_PTR() /* __p = ptr
RELOC_HIDE()                                                __offset = __my_cpu_offset */
```

위 call trace에서 `RELOC_HIDE()`는 전달된 `ptr`에 전달된 `off` (= `__offset`)을 더하여 CPU에 접근가능한 포인터를 반환한다. 이때 그 오프셋은 각각 다음과 같이 얻어진다.

```c
/* include/asm-generic/percpu.h */

#include <linux/compiler.h>
#include <linux/threads.h>
#include <linux/percpu-defs.h>

#ifdef CONFIG_SMP

/*
 * per_cpu_offset() is the offset that has to be added to a
 * percpu variable to get to the instance for a certain processor.
 *
 * Most arches use the __per_cpu_offset array for those offsets but
 * some arches have their own ways of determining the offset (x86_64, s390).
 */
#ifndef __per_cpu_offset
extern unsigned long __per_cpu_offset[NR_CPUS];

#define per_cpu_offset(x) (__per_cpu_offset[x])
#endif
```

```c
/* arch/ia64/include/asm/percpu.h */

#define __my_cpu_offset	__ia64_per_cpu_var(local_per_cpu_offset)

extern void *per_cpu_init(void);

#else /* ! SMP */

#define per_cpu_init()				(__phys_per_cpu_start)

#endif	/* SMP */

#define PER_CPU_BASE_SECTION ".data..percpu"

/*
 * Be extremely careful when taking the address of this variable!  Due to virtual
 * remapping, it is different from the canonical address returned by this_cpu_ptr(&var)!
 * On the positive side, using __ia64_per_cpu_var() instead of this_cpu_ptr() is slightly
 * more efficient.
 */
#define __ia64_per_cpu_var(var) (*({					\
	__verify_pcpu_ptr(&(var));					\
	((typeof(var) __kernel __force *)&(var));			\
}))

#include <asm-generic/percpu.h>

/* Equal to __per_cpu_offset[smp_processor_id()], but faster to access: */
DECLARE_PER_CPU(unsigned long, local_per_cpu_offset);
```



이러한 per-CPU는 kernel memory cache를 초기화할 때 할당되며, 다음 call trace가 그 예시이다.

```
/* function call trace */

socket_init()
skb_init()
kmem_cache_create_usercopy() /* name = "skbuff_head_cache,
                                size = sizeof(struct sk_buff) = 0xE0,
                                align = 0,
                                flags = SLAB_HWCACHE_ALIGN | SLAB_PANIC,
                                useroffset = offsetof(struct sk_buff, cb),
                                usersize = sizeof_field(struct sk_buff, cb),
                                ctor = NULL */
create_cache() /* object_size = size = sizeof(struct sk_buff) */
__kmem_cache_create()
kmeme_cache_open()
alloc_kmem_cache_cpus()
      |
      +-----------------------+
      |1                      |2
      V                       V
 __alloc_percpu()        init_kmem_cache_cpus()
 pcpu_alloc()
 pcpu_find_block_fit()
```

위 calltrace의 `pcpu_alloc()`에서 `pcpu_setup_first_chunk()` 등으로 할당된 slots인 per-CPU chunks로부터 할당하게 된다.

## NUMA Node

Non-Uniformed Memory Access (NUMA)는 일정 단위의 하드웨어 자원 (processor, I/O bus, ...)마다 가용한 메모리 부분을 나누어 symmetric multiprocessing system의 평균적인 경우에 대한 access time을 줄이기 위한 shared memory architecture이다. 리눅스 커널은 물리적 뷰에서 메모리를 포함한 각 하드웨어 자원을 cell로 구분하며, 이를 다시 node로 추상화한다. 이러한 node는 다시 cell과 매핑된다. 이를 그림으로 나타내면 다음과 같다[4, 5].

```
              Node 1                                           Node 2
+-----------------------------------+         +-----------------------------------+
| +------------------------------+  |         | +------------------------------+  |
| |  processor1, processor2, ... |  |         | |  processor1, processor2, ... |  |
| +------------------------------+  |---------| +------------------------------+  |
|           |       |               |inter    |           |       |               |
|           |       |               |connected|           |       |               |
|   +--------------------------+    |---------|    +------------------------+     |
|   |       memory             |    |         |    |      memory            |     |
|   +--------------------------+    |         |    +------------------------+     |
+-----------------------------------+ \     / +-----------------------------------+
                                  \     \  /     /
                                   \     \/     /
                                    \    /\    /
                                     \  /  \  /
                                      \/    \/
                                      /\    /\                                    
              Node 3                 /  \  /  \                 Node 4
+-----------------------------------+    \/   +-----------------------------------+
| +------------------------------+  |    /\   | +------------------------------+  |
| |  processor1, processor2, ... |  |   /  \  | |  processor1, processor2, ... |  |
| +------------------------------+  |---------| +------------------------------+  |
|           |       |               |inter    |           |       |               |
|           |       |               |connected|           |       |               |
|   +--------------------------+    |---------|    +------------------------+     |
|   |       memory             |    |         |    |      memory            |     |
|   +--------------------------+    |         |    +------------------------+     |
+-----------------------------------+         +-----------------------------------+
```

NUMA 아키텍처는 위 그림에서 볼 수 있듯이 local node에서의 access time과 remote node에서의 access time에 차이가 나게 된다.



리눅스 커널은 NUMA node에 접근하기 위한 인터페이스로 `get_node()` 등을 제공한다. 그리고 그 동작을 살펴보면 node를 인덱스로 하여 반환함을 알 수 있다.

```c
/* mm/slab.h */

static inline struct kmem_cache_node *get_node(struct kmem_cache *s, int node)
{
	return s->node[node];
}
```



이러한 NUMA node는 kernel memory cache를 초기화할 때 매핑되며, 다음 call trace를 통해 알 수 있다. (상대적 순서임에 주의하라)

```
/* function call trace */

socket_init()
skb_init()
kmem_cache_create_usercopy() /* name = "skbuff_head_cache,
                                size = sizeof(struct sk_buff) = 0xE0,
                                align = 0,
                                flags = SLAB_HWCACHE_ALIGN | SLAB_PANIC,
                                useroffset = offsetof(struct sk_buff, cb),
                                usersize = sizeof_field(struct sk_buff, cb),
                                ctor = NULL */
create_cache() /* object_size = size = sizeof(struct sk_buff) */
__kmem_cache_create()
kmeme_cache_open()
init_kmem_cache_nodes()
       |
       +----------------------------+
       |1                           |2
       V                            V
    kmem_cache_alloc_node()     init_kmem_cache_node()
```

## Kernel Memory Cache and Socket Buffer

리눅스 커널은 소켓 버퍼에 대한 slab cache를 관리하기 위해 `skbuff_head_cache`와 `skbuff_fclone_cache`를 둔다. 여기서 관심을 갖는 것은 `skbuff_head_cache`이며, `struct kmem_cache` 타입으로 선언되어 있다.

```c
/* include/linux/skbuff.h */

extern struct kmem_cache *skbuff_head_cache;
```

```c
/* include/linux/slub_def.h */

/*
 * Slab cache management.
 */
struct kmem_cache {
	struct kmem_cache_cpu __percpu *cpu_slab;
	/* Used for retrieving partial slabs, etc. */
	slab_flags_t flags;
	unsigned long min_partial;
	unsigned int size;	/* The size of an object including metadata */
	unsigned int object_size;/* The size of an object without metadata */
	struct reciprocal_value reciprocal_size;
	unsigned int offset;	/* Free pointer offset */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	/* Number of per cpu partial objects to keep around */
	unsigned int cpu_partial;
#endif
	struct kmem_cache_order_objects oo;

	/* Allocation and freeing of slabs */
	struct kmem_cache_order_objects max;
	struct kmem_cache_order_objects min;
	gfp_t allocflags;	/* gfp flags to use on each alloc */
	int refcount;		/* Refcount for slab cache destroy */
	void (*ctor)(void *);
	unsigned int inuse;		/* Offset to metadata */
	unsigned int align;		/* Alignment */
	unsigned int red_left_pad;	/* Left redzone padding size */
	const char *name;	/* Name (only for display!) */
	struct list_head list;	/* List of slab caches */
#ifdef CONFIG_SYSFS
	struct kobject kobj;	/* For sysfs */
#endif
#ifdef CONFIG_SLAB_FREELIST_HARDENED
	unsigned long random;
#endif

#ifdef CONFIG_NUMA
	/*
	 * Defragmentation by allocating from a remote node.
	 */
	unsigned int remote_node_defrag_ratio;
#endif

#ifdef CONFIG_SLAB_FREELIST_RANDOM
	unsigned int *random_seq;
#endif

#ifdef CONFIG_KASAN
	struct kasan_cache kasan_info;
#endif

	unsigned int useroffset;	/* Usercopy region offset */
	unsigned int usersize;		/* Usercopy region size */

	struct kmem_cache_node *node[MAX_NUMNODES];
};
```

이 캐시는 `skb_init()`에서 초기화 작업을 진행한다.

```
/* function call trace */

socket_init()
skb_init()
kmem_cache_create_usercopy() /* name = "skbuff_head_cache,
                                size = sizeof(struct sk_buff) = 0xE0,
                                align = 0,
                                flags = SLAB_HWCACHE_ALIGN | SLAB_PANIC,
                                useroffset = offsetof(struct sk_buff, cb),
                                usersize = sizeof_field(struct sk_buff, cb),
                                ctor = NULL */
create_cache() /* object_size = size = sizeof(struct sk_buff) */
```

```c
static struct kmem_cache *create_cache(const char *name,
		unsigned int object_size, unsigned int align,
		slab_flags_t flags, unsigned int useroffset,
		unsigned int usersize, void (*ctor)(void *),
		struct kmem_cache *root_cache)
{
	struct kmem_cache *s;
	int err;

	/* ... */

	err = -ENOMEM;
	s = kmem_cache_zalloc(kmem_cache, GFP_KERNEL);
	if (!s)
		goto out;

	s->name = name;
	s->size = s->object_size = object_size;
	s->align = align;
	s->ctor = ctor;
	s->useroffset = useroffset;
	s->usersize = usersize;

	s->refcount = 1;
	list_add(&s->list, &slab_caches);
out:
	if (err)
		return ERR_PTR(err);
	return s;

out_free_cache:
	kmem_cache_free(kmem_cache, s);
	goto out;
}
```

따라서 `s->object_size`는 `struct sk_buff` 타입의 크기임을 알 수 있고, GDB로 확인해보면 `0xe0` 바이트임을 알 수 있다. 그리고 `struct kmem_cache`에서 중요한 멤버로 `struct kmem_cache_cpu`가 있다. 이는 per-cpu 오브젝트이며, 다음과 같은 call trace를 거쳐 할당되고 초기화된다. (아래 call trace가 모든 함수를 표기하지 않았으며, 상대적 순서임에 주의하라)

```
/* function call trace */

socket_init()
skb_init()
kmem_cache_create_usercopy() /* name = "skbuff_head_cache,
                                size = sizeof(struct sk_buff) = 0xE0,
                                align = 0,
                                flags = SLAB_HWCACHE_ALIGN | SLAB_PANIC,
                                useroffset = offsetof(struct sk_buff, cb),
                                usersize = sizeof_field(struct sk_buff, cb),
                                ctor = NULL */
create_cache() /* object_size = size = sizeof(struct sk_buff) */
__kmem_cache_create()
kmeme_cache_open()
      |
      |
      +---------------------------+------------------------------------+
      |1                          |2                                   |3
      |                           |                                    |
      V                           V                                    V
alloc_kmem_cache_cpus()       set_min_partial() /* s = s, */      set_cpu_partial()
      |                            /* min = ilog2(s->size / 2) */
      +-----------------------+
      |1                      |2
      V                       V
 __alloc_percpu()        init_kmem_cache_cpus()
 pcpu_alloc()
 pcpu_find_block_fit()
```

```c
/* include/linux/slub_def.h */

/*
 * When changing the layout, make sure freelist and tid are still compatible
 * with this_cpu_cmpxchg_double() alignment requirements.
 */
struct kmem_cache_cpu {
	void **freelist;	/* Pointer to next available object */
	unsigned long tid;	/* Globally unique transaction id */
	struct page *page;	/* The slab from which we are allocating */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	struct page *partial;	/* Partially allocated frozen slabs */
#endif
	local_lock_t lock;	/* Protects the fields above */
#ifdef CONFIG_SLUB_STATS
	unsigned stat[NR_SLUB_STAT_ITEMS];
#endif
};
```

```c
/* mm/slub.c */

static void init_kmem_cache_cpus(struct kmem_cache *s)
{
	int cpu;
	struct kmem_cache_cpu *c;

	for_each_possible_cpu(cpu) {
		c = per_cpu_ptr(s->cpu_slab, cpu);
		local_lock_init(&c->lock);
		c->tid = init_tid(cpu);
	}
}
```

또, 살펴볼만한 것으로 `set_min_partial()`와 `set_cpu_partial()`이 있다. 전자는 일부의 오브젝트만 free된 slab의 linked list인 partial list의 최소 노드 수를 설정한다. 그리고 후자는 per-cpu partial list의 최대 노드 수를 설정한다. 이들은 후에 설명할 slab free 과정에서 다시 나올  것이다.

```c
/* mm/slub.c */

/*
 * Minimum number of partial slabs. These will be left on the partial
 * lists even if they are empty. kmem_cache_shrink may reclaim them.
 */
#define MIN_PARTIAL 5

/*
 * Maximum number of desirable partial slabs.
 * The existence of more partial slabs makes kmem_cache_shrink
 * sort the partial list by the number of objects in use.
 */
#define MAX_PARTIAL 10

static void set_min_partial(struct kmem_cache *s, unsigned long min)
{
	if (min < MIN_PARTIAL)
		min = MIN_PARTIAL;
	else if (min > MAX_PARTIAL)
		min = MAX_PARTIAL;
	s->min_partial = min;
}
```

```c
/* mm/slub.c */

static void set_cpu_partial(struct kmem_cache *s)
{
#ifdef CONFIG_SLUB_CPU_PARTIAL
	/*
	 * cpu_partial determined the maximum number of objects kept in the
	 * per cpu partial lists of a processor.
	 *
	 * Per cpu partial lists mainly contain slabs that just have one
	 * object freed. If they are used for allocation then they can be
	 * filled up again with minimal effort. The slab will never hit the
	 * per node partial lists and therefore no locking will be required.
	 *
	 * This setting also determines
	 *
	 * A) The number of objects from per cpu partial slabs dumped to the
	 *    per node list when we reach the limit.
	 * B) The number of objects in cpu partial slabs to extract from the
	 *    per node list when we run out of per cpu objects. We only fetch
	 *    50% to keep some capacity around for frees.
	 */
	if (!kmem_cache_has_cpu_partial(s))
		slub_set_cpu_partial(s, 0);
	else if (s->size >= PAGE_SIZE)
		slub_set_cpu_partial(s, 2);
	else if (s->size >= 1024)
		slub_set_cpu_partial(s, 6);
	else if (s->size >= 256)
		slub_set_cpu_partial(s, 13);
	else
		slub_set_cpu_partial(s, 30);
#endif
}
```

```c
#define slub_cpu_partial(s)		((s)->cpu_partial)
#define slub_set_cpu_partial(s, n)		\
({						\
	slub_cpu_partial(s) = (n);		\
})
```



이밖에도 여러 멤버 변수들이 초기화되며, 그 예시는 다음과 같다:

```
gef> p *(struct kmem_cache *) skbuff_head_cache
$3 = {
  cpu_slab = 0x2e380,
  flags = 0x40042000,
  min_partial = 0x5,
  size = 0x100,
  object_size = 0xe0,
  reciprocal_size = {
    m = 0x1,
    sh1 = 0x1,
    sh2 = 0x7
  },
  offset = 0x70,
  cpu_partial = 0xd,
  oo = {
    x = 0x10
  },
  max = {
    x = 0x10
  },
  min = {
    x = 0x10
  },
  allocflags = 0x0,
  refcount = 0x1,
  ctor = 0x0 <fixed_percpu_data>,
  inuse = 0xe0,
  align = 0x40,
  red_left_pad = 0x0,
  name = 0xffffffff8261d1f0 "skbuff_head_cache",
  list = {
    next = 0xffff8880036a4168,
    prev = 0xffff8880036a4368
  },
  kobj = {
    name = 0xffffffff8261d1f0 "skbuff_head_cache",
    entry = {
      next = 0xffff8880036a4180,
      prev = 0xffff8880036a4380
    },
    parent = 0xffff888003629a98,
    kset = 0xffff888003629a80,
    ktype = 0xffffffff8295b620 <slab_ktype>,
    sd = 0xffff88800363ff80,
    kref = {
      refcount = {
        refs = {
          counter = 0x1
        }
      }
    },
    state_initialized = 0x1,
    state_in_sysfs = 0x1,
    state_add_uevent_sent = 0x0,
    state_remove_uevent_sent = 0x0,
    uevent_suppress = 0x0
  },
  remote_node_defrag_ratio = 0x3e8,
  useroffset = 0x28,
  usersize = 0x30,
  node = {
    [0x0] = 0xffff8880036a5080,
    [0x1] = 0x0 <fixed_percpu_data>,
    [0x2] = 0x0 <fixed_percpu_data>,
    [0x3] = 0x0 <fixed_percpu_data>,
    [0x4] = 0x0 <fixed_percpu_data>,
    [0x5] = 0x0 <fixed_percpu_data>,
    [0x6] = 0x0 <fixed_percpu_data>,
    [0x7] = 0x2e3a0,
    [0x8] = 0x40042000,
    [0x9] = 0x5 <fixed_percpu_data+5>,
    [0xa] = 0x20000000200,
    [0xb] = 0x80100000001,
    [0xc] = 0xd000000e0,
    [0xd] = 0x1001000010010,
    [0xe] = 0x4000000000008,
    [0xf] = 0x2 <fixed_percpu_data+2>,
    [0x10] = 0x0 <fixed_percpu_data>,
    [0x11] = 0x4000000200,
    [0x12] = 0x0 <fixed_percpu_data>,
    [0x13] = 0xffffffff8261d202,
    [0x14] = 0xffff8880036a4268,
    [0x15] = 0xffff8880036a4468,
    [0x16] = 0xffff8880034fc2d0,
    [0x17] = 0xffff8880036a4280,
    [0x18] = 0xffff8880036a4480,
    [0x19] = 0xffff888003629a98,
    [0x1a] = 0xffff888003629a80,
    [0x1b] = 0xffffffff8295b620 <slab_ktype>,
    [0x1c] = 0xffff88800363f000,
    [0x1d] = 0x300000001,
    [0x1e] = 0x3e8,
    [0x1f] = 0x0 <fixed_percpu_data>,
    [0x20] = 0xffff8880036a50c0,
    [0x21] = 0x0 <fixed_percpu_data>,
    [0x22] = 0x0 <fixed_percpu_data>,
    [0x23] = 0x0 <fixed_percpu_data>,
    [0x24] = 0x0 <fixed_percpu_data>,
    [0x25] = 0x0 <fixed_percpu_data>,
    [0x26] = 0x0 <fixed_percpu_data>,
    [0x27] = 0x2e3c0,
    [0x28] = 0x40022000,
    [0x29] = 0x5 <fixed_percpu_data+5>,
    [0x2a] = 0x30000000340,
    [0x2b] = 0x9013b13b13c,
    [0x2c] = 0xd00000300,
    [0x2d] = 0x2001300020013,
    [0x2e] = 0x4001000000004,
    [0x2f] = 0x1 <fixed_percpu_data+1>,
    [0x30] = 0xffffffff818d8e40 <init_once>,
    [0x31] = 0x4000000300,
    [0x32] = 0x0 <fixed_percpu_data>,
    [0x33] = 0xffffffff8261cbe8,
    [0x34] = 0xffff8880036a4368,
    [0x35] = 0xffff888003508c68,
    [0x36] = 0xffffffff8261cbe8,
    [0x37] = 0xffff8880036a4380,
    [0x38] = 0xffff888003508c80,
    [0x39] = 0xffff888003629a98,
    [0x3a] = 0xffff888003629a80,
    [0x3b] = 0xffffffff8295b620 <slab_ktype>,
    [0x3c] = 0xffff88800363e100,
    [0x3d] = 0x300000001,
    [0x3e] = 0x3e8,
    [0x3f] = 0x0 <fixed_percpu_data>
  }
}
gef> 
```

## SLUB allocator and SKB allocation

SLUB allocator는 기존에 사용되던 SLAB allocator의 대안으로 제시되었으며, slab을 특정 오브젝트로 구성된 페이지의 그룹으로 바라본다. 이러한 slab은 그 자체로는 freelist를 관리하지 않는다. 그래서 freelist를 관리하기 위한 데이터를 페이지 구조체에 저장한다[1].

```c
/* include/linux/mm_types.h */

/*
 * Each physical page in the system has a struct page associated with
 * it to keep track of whatever it is we are using the page for at the
 * moment. Note that we have no way to track which tasks are using
 * a page, though if it is a pagecache page, rmap structures can tell us
 * who is mapping it.
 *
 * If you allocate the page using alloc_pages(), you can use some of the
 * space in struct page for your own purposes.  The five words in the main
 * union are available, except for bit 0 of the first word which must be
 * kept clear.  Many users use this word to store a pointer to an object
 * which is guaranteed to be aligned.  If you use the same storage as
 * page->mapping, you must restore it to NULL before freeing the page.
 *
 * If your page will not be mapped to userspace, you can also use the four
 * bytes in the mapcount union, but you must call page_mapcount_reset()
 * before freeing it.
 *
 * If you want to use the refcount field, it must be used in such a way
 * that other CPUs temporarily incrementing and then decrementing the
 * refcount does not cause problems.  On receiving the page from
 * alloc_pages(), the refcount will be positive.
 *
 * If you allocate pages of order > 0, you can use some of the fields
 * in each subpage, but you may need to restore some of their values
 * afterwards.
 *
 * SLUB uses cmpxchg_double() to atomically update its freelist and
 * counters.  That requires that freelist & counters be adjacent and
 * double-word aligned.  We align all struct pages to double-word
 * boundaries, and ensure that 'freelist' is aligned within the
 * struct.
 */
#ifdef CONFIG_HAVE_ALIGNED_STRUCT_PAGE
#define _struct_page_alignment	__aligned(2 * sizeof(unsigned long))
#else
#define _struct_page_alignment
#endif

struct page {
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	/*
	 * Five words (20/40 bytes) are available in this union.
	 * WARNING: bit 0 of the first word is used for PageTail(). That
	 * means the other users of this union MUST NOT use the bit to
	 * avoid collision and false-positive PageTail().
	 */
	union {
		/* ... */
		struct {	/* slab, slob and slub */
			union {
				struct list_head slab_list;
				struct {	/* Partial pages */
					struct page *next;
#ifdef CONFIG_64BIT
					int pages;	/* Nr of pages left */
					int pobjects;	/* Approximate count */
#else
					short int pages;
					short int pobjects;
#endif
				};
			};
			struct kmem_cache *slab_cache; /* not slob */
			/* Double-word boundary */
			void *freelist;		/* first free object */
			union {
				void *s_mem;	/* slab: first object */
				unsigned long counters;		/* SLUB */
				struct {			/* SLUB */
					unsigned inuse:16;
					unsigned objects:15;
					unsigned frozen:1;
				};
			};
		};
		/* ... */

		/** @rcu_head: You can use this to free a page by RCU. */
		struct rcu_head rcu_head;
	};

	union {		/* This union is 4 bytes in size. */
		/*
		 * If the page can be mapped to userspace, encodes the number
		 * of times this page is referenced by a page table.
		 */
		atomic_t _mapcount;

		/*
		 * If the page is neither PageSlab nor mappable to userspace,
		 * the value stored here may help determine what this page
		 * is used for.  See page-flags.h for a list of page types
		 * which are currently stored here.
		 */
		unsigned int page_type;

		unsigned int active;		/* SLAB */
		int units;			/* SLOB */
	};

	/* Usage count. *DO NOT USE DIRECTLY*. See page_ref.h */
	atomic_t _refcount;

#ifdef CONFIG_MEMCG
	unsigned long memcg_data;
#endif

	/*
	 * On machines where all RAM is mapped into kernel address space,
	 * we can simply calculate the virtual address. On machines with
	 * highmem some memory is mapped into kernel virtual memory
	 * dynamically, so we need a place to store that address.
	 * Note that this field could be 16 bits on x86 ... ;)
	 *
	 * Architectures with slow multiplication can define
	 * WANT_PAGE_VIRTUAL in asm/page.h
	 */
#if defined(WANT_PAGE_VIRTUAL)
	void *virtual;			/* Kernel virtual address (NULL if
					   not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */

#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
	int _last_cpupid;
#endif
} _struct_page_alignment;
```

SLUB은, 위 소스 코드의 주석에서도 볼 수 있듯이 `freelist` 멤버를 참조하여 오브젝트를 할당하고 다음으로 할당할 위치를 `offset`으로 관리한다. 이를 그림으로 나타내면 다음과 같다:

```
               page:
                     +-----------------------+
                     |         ...           |
    freelist ------->+-----------------------+
                     | returned object       |
                     +-----------------------+
                     |         ...           |
                     +-----------------------+<----- freelist + offset
                     | next object           |
                     +-----------------------+
                     |         ...           |
                     +-----------------------+
```

그럼 SLUB이 `struct sk_buff`를 할당하는 과정을 소스 코드로 알아보겠다[1].



그 시작은 `alloc_skb` 함수이고, `__alloc_skb` 함수의 레퍼 함수 (wrapper function)이다. (`kmalloc()`의 call trace가 `slab_alloc_node()`부터 겹침을 확인하라)

```
/* function call trace */

alloc_skb()                                                 kmalloc()
__alloc_skb() /* flags = 0, node = NUMA_NO_NODE = -1 */ __kmalloc() /* size<pagesize*2 */
kmem_cache_alloc_node() /* cache = skbuff_head_cache, */        |
                        /* gfp_mask & ~GFP_DMA, */              +-----------+
                        /* node = NUMA_NO_NODE = -1 */          |           |
slab_alloc_node() /* s = cache = skbuff_head_cache, */          V           V
                  /* gfpflags = gfp_mask & ~GFP_DMA, */  kmalloc_slab()  slab_alloc()
                  /* node = NUMA_NO_NODE = -1, */                       slab_alloc_node()  
                  /* addr = _RET_IP_, */
                  /* orig_size = s->object_size */
```

위 call trace에서 `slab_alloc_node` 함수의 구현을 살펴보면 할당의 상황들이 나오기 시작한다. 만약 현재 상황이 초기 할당이거나 slab cache가 모두 할당된 상황이라면 `struct kmem_cache_cpu`의 `freelist`나 `page` 멤버가 유효한 메모리 주소를 갖고 있지 않을 것이다. 하지만 이에 해당하지 않는다면 이들은 어떤 주소든 갖고 있을 것이며, 따라서 이로부터 할당하면 될 것이다.

```c
/* mm/slub.c */

/*
 * Inlined fastpath so that allocation functions (kmalloc, kmem_cache_alloc)
 * have the fastpath folded into their functions. So no function call
 * overhead for requests that can be satisfied on the fastpath.
 *
 * The fastpath works by first checking if the lockless freelist can be used.
 * If not then __slab_alloc is called for slow processing.
 *
 * Otherwise we can simply pick the next object from the lockless free list.
 */
static __always_inline void *slab_alloc_node(struct kmem_cache *s,
		gfp_t gfpflags, int node, unsigned long addr, size_t orig_size)
{
	void *object;
	struct kmem_cache_cpu *c;
	struct page *page;
	unsigned long tid;
	struct obj_cgroup *objcg = NULL;
	bool init = false;

	/* ... */

redo:
	/*
	 * Must read kmem_cache cpu data via this cpu ptr. Preemption is
	 * enabled. We may switch back and forth between cpus while
	 * reading from one cpu area. That does not matter as long
	 * as we end up on the original cpu again when doing the cmpxchg.
	 *
	 * We must guarantee that tid and kmem_cache_cpu are retrieved on the
	 * same cpu. We read first the kmem_cache_cpu pointer and use it to read
	 * the tid. If we are preempted and switched to another cpu between the
	 * two reads, it's OK as the two are still associated with the same cpu
	 * and cmpxchg later will validate the cpu.
	 */
	c = raw_cpu_ptr(s->cpu_slab);
	tid = READ_ONCE(c->tid);

	/*
	 * Irqless object alloc/free algorithm used here depends on sequence
	 * of fetching cpu_slab's data. tid should be fetched before anything
	 * on c to guarantee that object and page associated with previous tid
	 * won't be used with current tid. If we fetch tid first, object and
	 * page could be one associated with next tid and our alloc/free
	 * request will be failed. In this case, we will retry. So, no problem.
	 */
	barrier();

	/*
	 * The transaction ids are globally unique per cpu and per operation on
	 * a per cpu queue. Thus they can be guarantee that the cmpxchg_double
	 * occurs on the right processor and that there was no operation on the
	 * linked list in between.
	 */

	object = c->freelist;
	page = c->page;
	/*
	 * We cannot use the lockless fastpath on PREEMPT_RT because if a
	 * slowpath has taken the local_lock_irqsave(), it is not protected
	 * against a fast path operation in an irq handler. So we need to take
	 * the slow path which uses local_lock. It is still relatively fast if
	 * there is a suitable cpu freelist.
	 */
	if (IS_ENABLED(CONFIG_PREEMPT_RT) ||
	    unlikely(!object || !page || !node_match(page, node))) {
		object = __slab_alloc(s, gfpflags, node, addr, c);
	} else {
		void *next_object = get_freepointer_safe(s, object);

		/*
		 * The cmpxchg will only match if there was no additional
		 * operation and if we are on the right processor.
		 *
		 * The cmpxchg does the following atomically (without lock
		 * semantics!)
		 * 1. Relocate first pointer to the current per cpu area.
		 * 2. Verify that tid and freelist have not been changed
		 * 3. If they were not changed replace tid and freelist
		 *
		 * Since this is without lock semantics the protection is only
		 * against code executing on this cpu *not* from access by
		 * other cpus.
		 */
		if (unlikely(!this_cpu_cmpxchg_double(
				s->cpu_slab->freelist, s->cpu_slab->tid,
				object, tid,
				next_object, next_tid(tid)))) {

			note_cmpxchg_failure("slab_alloc", s, tid);
			goto redo;
		}
		prefetch_freepointer(s, next_object);
		stat(s, ALLOC_FASTPATH);
	}

	maybe_wipe_obj_freeptr(s, object);
	init = slab_want_init_on_alloc(gfpflags, s);

out:
	slab_post_alloc_hook(s, objcg, gfpflags, 1, &object, init);

	return object;
}
```

계속해서 나아가면, 다음과 같이 call trace를 정리할 수 있다. (`kmalloc()`의 call trace가 `slab_alloc_node()`부터 겹침을 확인하라)

```
/* function call trace */

alloc_skb()                                                 kmalloc()
__alloc_skb() /* flags = 0, node = NUMA_NO_NODE = -1 */ __kmalloc() /* size<pagesize*2 */
kmem_cache_alloc_node() /* cache = skbuff_head_cache, */        |
                        /* gfp_mask & ~GFP_DMA, */              +-----------+
                        /* node = NUMA_NO_NODE = -1 */          |           |
slab_alloc_node() /* s = cache = skbuff_head_cache, */          V           V
     |            /* gfpflags = gfp_mask & ~GFP_DMA, */  kmalloc_slab()  slab_alloc()
     |            /* node = NUMA_NO_NODE = -1, */                       slab_alloc_node()  
     |            /* addr = _RET_IP_, */
     |            /* orig_size = s->object_size */
     |
     +-------------------+
     |slowpath           |fastpath (slab cache freelist)
     |                   |
     |no freelist or     |
     |no page            |--------------------------+
     |                   |1                         |2
     V                   V                          V
__slab_alloc()         get_freepointer_safe()   this_cpu_cmpxchg_double()
___slab_alloc()                                 raw_cpu_cmpxchg_double()
     |                                          __pcpu_double_call_return_bool()
     |                                          raw_cpu_cmpxchg_double_[1248]()
     |                                          raw_cpu_generic_cmpxchg_double()
     |
     |
     +---------------------------+------------------------+
     |3                          |1                       |2
     |no page                    |per-cpu partial list    |NUMA node partial list
     |                           |                        |
     V                           V                        V
new_slab()                slub_percpu_partial()       get_partial()
allocate_slab()
alloc_slab_page()
alloc_pages()
alloc_pages_node()
__alloc_pages_node()
__alloc_pages() /* This is the 'heart' of the zoned buddy allocator */
get_page_from_freelist()
rmqueue()
    |
    |order <= 3
    |
    V
rmqueue_pcplist()
__rmqueue_pcplist() /* Remove page from the per-cpu list, caller must protect the list */
    |
    |if the list is empty
    |
    V
rmqueue_bulk() /* Obtain a specified number of elements from the buddy allocator */
```

위 call trace를 살펴보면, slab의 초기 할당에서는 page allocator (zoned buddy allocator)로부터 페이지를 할당받는 것이 분명해보인다. 특히 per-cpu의 page list (linked list)가 비어있을 때는 buddy allocator로부터 페이지를 얻어올 것이다. 여기서는 slab fastpath를 살펴보도록 하겠다. 먼저 `get_freepointer_safe()`와 `slab_alloc_node()`의 구현을 읽어보면 slab cache의 `freelist`에 `s->offset`를 더하여 할당할 오브젝트의 주소를 얻음을 알 수 있다. 그리고 여기에 KASLR이 활성화될 경우 추가적인 연산 (`freelist_ptr()`을 참고하라)이 수행된다.

```c
/* mm/slub.c */

static inline void *get_freepointer_safe(struct kmem_cache *s, void *object)
{
	unsigned long freepointer_addr;
	void *p;

	if (!debug_pagealloc_enabled_static())
		return get_freepointer(s, object);

	object = kasan_reset_tag(object);
	freepointer_addr = (unsigned long)object + s->offset;
	copy_from_kernel_nofault(&p, (void **)freepointer_addr, sizeof(p));
	return freelist_ptr(s, p, freepointer_addr);
}
```

이렇게 얻어진 주소는 `raw_cpu_generic_cmpxchg_double()`에서 `freelist`에 쓰여지게 된다.

```c
/* include/asm-generic/percpu.h */
/* 
   pcp1 = s->cpu_slab->freelist
   pcp2 = s->cpu_slab->tid
   oval1 = object (= freelist)
   oval2 = tid
   nval1 = next_object (= get_freepointer_safe())
   nval2 = next_tid(tid)
*/

#define raw_cpu_generic_cmpxchg_double(pcp1, pcp2, oval1, oval2, nval1, nval2) \
({									\
	typeof(pcp1) *__p1 = raw_cpu_ptr(&(pcp1));			\
	typeof(pcp2) *__p2 = raw_cpu_ptr(&(pcp2));			\
	int __ret = 0;							\
	if (*__p1 == (oval1) && *__p2  == (oval2)) {			\
		*__p1 = nval1;						\
		*__p2 = nval2;						\
		__ret = 1;						\
	}								\
	(__ret);							\
})
```



정리하면, 커널 내부에서 사용되는 오브젝트, 특히 SKB를 할당할 때의 우선순위는 다음과 같다:

1. Slab cache (skbuff cache) freelist
2. Slub per-cpu partial list
3. NUMA node partial list
4. Page allocator (zoned buddy allocator)
   1. Per-cpu freelis
   2. Other paths ...

## SLUB allocator and SKB deallocation

SLUB allocator는 slab이 해제될 때 이 slab이 속한 페이지를 partial list에 넣는다. 그런데 이때 slab 전체가 해제된 경우 그 페이지는 page allocator가 갖게 된다[1].



SKB를 해제하는 과정은 `kfree_skb` 함수에서부터 시작된다. (`kfree()`의 call trace가 `slab_free()`에서부터 겹침을 확인하라)

```
/* function call trace */

kfree_skb()                                           kfree()
__kfree_skb()                                         slab_free()
kfree_skbmem()
kmem_cache_free()
slab_free() /* s = skbuff_head_cache
               page = virt_to_head_page(skb)
               head = skb
               tail = NULL
               cnt = 1
               addr = _RET_IP */
do_slab_free() /* this function is fastpath */
```

만약 해제하고자 하는 slab이 속한 페이지와 cpu slab이 바라보고 있는 페이지가 같다면, 그 freelist를 조작하는 것으로 작업을 끝낼 수 있다. 이것이 slab free에서의 fastpath이고, 상기의 조건을 만족시키지 못한다면 slowpath로 넘어간다.

```c
/* mm/slub.c */

/*
 * Fastpath with forced inlining to produce a kfree and kmem_cache_free that
 * can perform fastpath freeing without additional function calls.
 *
 * The fastpath is only possible if we are freeing to the current cpu slab
 * of this processor. This typically the case if we have just allocated
 * the item before.
 *
 * If fastpath is not possible then fall back to __slab_free where we deal
 * with all sorts of special processing.
 *
 * Bulk free of a freelist with several objects (all pointing to the
 * same page) possible by specifying head and tail ptr, plus objects
 * count (cnt). Bulk free indicated by tail pointer being set.
 */
static __always_inline void do_slab_free(struct kmem_cache *s,
				struct page *page, void *head, void *tail,
				int cnt, unsigned long addr)
{
	void *tail_obj = tail ? : head;
	struct kmem_cache_cpu *c;
	unsigned long tid;

	/* memcg_slab_free_hook() is already called for bulk free. */
	if (!tail)
		memcg_slab_free_hook(s, &head, 1);
redo:
	/*
	 * Determine the currently cpus per cpu slab.
	 * The cpu may change afterward. However that does not matter since
	 * data is retrieved via this pointer. If we are on the same cpu
	 * during the cmpxchg then the free will succeed.
	 */
	c = raw_cpu_ptr(s->cpu_slab);
	tid = READ_ONCE(c->tid);

	/* Same with comment on barrier() in slab_alloc_node() */
	barrier();

	if (likely(page == c->page)) {
#ifndef CONFIG_PREEMPT_RT
		/* ... */
#else /* CONFIG_PREEMPT_RT */
		/*
		 * We cannot use the lockless fastpath on PREEMPT_RT because if
		 * a slowpath has taken the local_lock_irqsave(), it is not
		 * protected against a fast path operation in an irq handler. So
		 * we need to take the local_lock. We shouldn't simply defer to
		 * __slab_free() as that wouldn't use the cpu freelist at all.
		 */
		void **freelist;

		local_lock(&s->cpu_slab->lock);
		c = this_cpu_ptr(s->cpu_slab);
		if (unlikely(page != c->page)) {
			local_unlock(&s->cpu_slab->lock);
			goto redo;
		}
		tid = c->tid;
		freelist = c->freelist;

		set_freepointer(s, tail_obj, freelist);
		c->freelist = head;
		c->tid = next_tid(tid);

		local_unlock(&s->cpu_slab->lock);
#endif
		stat(s, FREE_FASTPATH);
	} else
		__slab_free(s, page, head, tail_obj, cnt, addr);

}
```

Slowpath에서 계속 따라가면 다음과 같이 call trace를 그려볼 수 있다. (아래 call trace에서 `else condition`은 다른 조건이 만족되지 않을 경우 진행되는 경로임에 주의;  `kfree()`의 call trace가 `slab_free()`에서부터 겹침을 확인하라)

```
/* function call trace */

kfree_skb()                                           kfree()
__kfree_skb()                                         slab_free()
kfree_skbmem()
kmem_cache_free()
slab_free() /* s = skbuff_head_cache,
               page = virt_to_head_page(skb),
               head = skb,
               tail = NULL,
               cnt = 1,
               addr = _RET_IP */
do_slab_free() /* this function is fastpath */
     |
     |slowpath
     |
     V
__slab_free() /* s = skbuff_head_cache,
     |           page = virt_to_head_page(skb),
     |           head = skb,
     |           tail = skb,
     |           cnt = 1,
     |           addr = _RET_IP_ */
     |
     +---------------------------+---------------------------------+
     |if slab is empty           |likely,                          |
     |                           |if slab is full                  |
     |                           |                                 |
     |                           |                                 |
     |page list                  |per-cpu partial list             |NUMA node partial
     |                           |                                 |list
     |                           |                                 |
     V                           V                                 V
discard_slab()<--+            put_cpu_partial()/* drain=1 */ add_partial() <---+
free_slab()      |               |                           __add_partial()   |
__free_slab()    |slab empty     |per-cpu partial list                         |else
__free_pages()   |               |is full                                      |condition
free_the_page()  |               |                                             |
    |            |               V                                             |
    |            +----------- __unfreeze_partials()----------------------------+
    |order <= 3
    |
    V
free_unref_page()
free_unref_page_commit() /* Free a pcp page */
```

위 call trace에서 `slab empty`와 `slab full`, `per-cpu partial list full`의 조건은 `__slab_free()`에서 확인할 수 있다.

```c
/* mm/slab.h */

/*
 * The slab lists for all objects.
 */
struct kmem_cache_node {
	spinlock_t list_lock;

#ifdef CONFIG_SLAB
	/* ... */
#endif

#ifdef CONFIG_SLUB
	unsigned long nr_partial;
	struct list_head partial;
#ifdef CONFIG_SLUB_DEBUG
	atomic_long_t nr_slabs;
	atomic_long_t total_objects;
	struct list_head full;
#endif
#endif

};
```

```c
/* mm/slub.c */

/*
 * Slow path handling. This may still be called frequently since objects
 * have a longer lifetime than the cpu slabs in most processing loads.
 *
 * So we still attempt to reduce cache line usage. Just take the slab
 * lock and free the item. If there is no additional partial page
 * handling required then we can return immediately.
 */
static void __slab_free(struct kmem_cache *s, struct page *page,
			void *head, void *tail, int cnt,
			unsigned long addr)

{
	void *prior;
	int was_frozen;
	struct page new;
	unsigned long counters;
	struct kmem_cache_node *n = NULL;
	unsigned long flags;

	/* ... */
  
	do {
		if (unlikely(n)) {
			spin_unlock_irqrestore(&n->list_lock, flags);
			n = NULL;
		}
		prior = page->freelist;
		counters = page->counters;
		set_freepointer(s, tail, prior);
		new.counters = counters;
		was_frozen = new.frozen;
		new.inuse -= cnt;
		if ((!new.inuse || !prior) && !was_frozen) {

			if (kmem_cache_has_cpu_partial(s) && !prior) {

				/*
				 * Slab was on no list before and will be
				 * partially empty
				 * We can defer the list move and instead
				 * freeze it.
				 */
				new.frozen = 1;

			} else { /* Needs to be taken off a list */

				n = get_node(s, page_to_nid(page));
				/*
				 * Speculatively acquire the list_lock.
				 * If the cmpxchg does not succeed then we may
				 * drop the list_lock without any processing.
				 *
				 * Otherwise the list_lock will synchronize with
				 * other processors updating the list of slabs.
				 */
				spin_lock_irqsave(&n->list_lock, flags);

			}
		}

	} while (!cmpxchg_double_slab(s, page,
		prior, counters,
		head, new.counters,
		"__slab_free"));

	if (likely(!n)) {

		if (likely(was_frozen)) {
			/*
			 * The list lock was not taken therefore no list
			 * activity can be necessary.
			 */
			stat(s, FREE_FROZEN);
		} else if (new.frozen) {
			/*
			 * If we just froze the page then put it onto the
			 * per cpu partial list.
			 */
			put_cpu_partial(s, page, 1);
			stat(s, CPU_PARTIAL_FREE);
		}

		return;
	}

	if (unlikely(!new.inuse && n->nr_partial >= s->min_partial))
		goto slab_empty;

	/*
	 * Objects left in the slab. If it was not on the partial list before
	 * then add it.
	 */
	if (!kmem_cache_has_cpu_partial(s) && unlikely(!prior)) {
		remove_full(s, n, page);
		add_partial(n, page, DEACTIVATE_TO_TAIL);
		stat(s, FREE_ADD_PARTIAL);
	}
	spin_unlock_irqrestore(&n->list_lock, flags);
	return;

slab_empty:
    if (prior) {
		/*
		 * Slab on the partial list.
		 */
		remove_partial(n, page);
		stat(s, FREE_REMOVE_PARTIAL);
	} else {
		/* Slab must be on the full list */
		remove_full(s, n, page);
	}
  
	/* ... */
    discard_slab(s, page);
}
```

```c
/* mm/slub.c */

static inline bool cmpxchg_double_slab(struct kmem_cache *s, struct page *page,
		void *freelist_old, unsigned long counters_old,
		void *freelist_new, unsigned long counters_new,
		const char *n)
{
#if defined(CONFIG_HAVE_CMPXCHG_DOUBLE) && \
    defined(CONFIG_HAVE_ALIGNED_STRUCT_PAGE)
	if (s->flags & __CMPXCHG_DOUBLE) {
		if (cmpxchg_double(&page->freelist, &page->counters,
				   freelist_old, counters_old,
				   freelist_new, counters_new))
			return true;
	} else
#endif
	{
		unsigned long flags;

		local_irq_save(flags);
		__slab_lock(page);
		if (page->freelist == freelist_old &&
					page->counters == counters_old) {
			page->freelist = freelist_new;
			page->counters = counters_new;
			__slab_unlock(page);
			local_irq_restore(flags);
			return true;
		}
		__slab_unlock(page);
		local_irq_restore(flags);
	}

	cpu_relax();
	stat(s, CMPXCHG_DOUBLE_FAIL);

#ifdef SLUB_DEBUG_CMPXCHG
	pr_info("%s %s: cmpxchg double redo ", n, s->name);
#endif

	return false;
}
```

이때 slab empty를 판별하는 과정에서 NUMA node의 partial list 내 페이지 수 (`n->nr_partial`) 와 `s->min_partial`을 비교함을 확인하라. 이는 전달된 페이지에 할당된 slab이 없고, NUMA node에 이미 최소한의 partial slab이 확보되어 있음을 의미한다.

## References

1. corbet, Apr. 2007, "The SLUB allocator," LWN. [Online]. Available: https://lwn.net/Articles/229984/
2. Jeff Bonwick, "The Slab Allocator: An Object-Cacheing Kenrel Memory Allocator," in Proc. USENIX Summer 1994 Technical Conference, Boston, Massachusetts, USA, Jun. 1994, pp. 87 -- 98.
3. Jonathan Corbet, Jul. 2011, "Per-CPU variables and the realtime tree," LWN. [Online]. Available: https://lwn.net/Articles/452884/
4. Kanoj Sarcar, et al., Nov. 1999, "What is NUMA?," Linux Memory Management Documentation. [Online]. Available: https://www.kernel.org/doc/html/v4.18/vm/numa.html
5. "Optimizing Applications for NUMA," Intel Guide for Developing Multithreaded Applications. [Online]. Available: https://www.intel.com/content/dam/develop/external/us/en/documents/3-5-memmgt-optimizing-applications-for-numa-184398.pdf
