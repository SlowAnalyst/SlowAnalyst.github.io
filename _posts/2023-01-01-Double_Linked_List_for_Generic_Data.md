---
layout: post
title: 일반적인 (generic) 데이터를 저장하는 이중 연결 리스트
---


Contents
--------
1. 이중 연결 리스트
2. 구현
   1. 노드 구성: non-intrusive vs intrusive
   2. 노드를 연결하는 방법
   3. 사용자 인터페이스: 클래스
   4. 에러 처리
   5. CRUD (Create, Read, Update, Delete)
3. References



# 이중 연결 리스트
 연결 리스트 (Linked List)라는 개념은 [5]가 제시한 개념으로, 문제의 증명을 기호 논리를 통해 찾는 프로그램인 Logical Theorist (LT)을 개발하는 과정에서 제안되었다. ([5]의 저자들은 이 논문이 카네기 공대의 H. A. Simon과 공동으로 진행한 연구 프로젝트의 일부임을 밝히고 있다) 해당 프로그램에 적용되는 여러 요구사항 (표현의 리스트를 다룰 수 있어야 하고, 논리 문제를 해결하기 위한 프로세스의 리스트를 다룰 수 있어야 한다는 것 등)을 해결하기 위해서는 한 언어가 필요했고 이것이 바로 Information Processing Language (IPL)이다. 즉, LT가 다룰 수 있어야 하는 리스트를 표현하기 위해 IPL이 사용된 것이다. 그리고 IPL은 리스트를 표현할 때 location words를 사용했는데, 이는 현대의 포인터와 같은 개념이라고 볼 수 있다. IPL은 리스트에 두 가지 location words를 두었는데, 하나는 element를 가리키고, 다른 하나는 다음 location words를 가리키도록 구성했다. 이를 그림으로 표현하면 다음과 같다.
 
```
    +--------+    +--------+    +--------+
--->|Location|--->|Location|--->|Location|---> (-1)
    +--------+    +--------+    +--------+
        |              |             |
        V              V             V
      element       element       element
```

위 그림에서 -1은 리스트의 종료를 의미하며, 이는 현대의 NULL과 같은 개념이다. 따라서 [5]가 제시한 리스트의 형태는 [1, Sec. 3.1]이 설명하는 연결된 자료 구조 (Linked Data Structures)에 속하는 단일 연결 리스트 (Single Linked List)의 형태임을 알 수 있다.

 이중 연결 리스트 (Double Linked List)는 [1, Sec. 3.1]이 설명하는 연결된 자료 구조 (Linked Data Structures)에 속하는 단일 연결 리스트 (Single Linked List)에서 이전 노드를 가리키는 것이 추가된 자료구조이다. 이를 그림으로 표현하면 다음과 같다.
 
```
    +----+    +----+    +----+
<---|node|<---|node|<---|node|--->
    |    |--->|    |--->|    |
    +----+    +----+    +----+
```

위와 같은 구조에서 첫 번째 노드와 마지막 노드는 각각 head, tail이라고 불리며 접근하기 위한 변수가 별도로 존재하여 접근하는데 O(1)이 소요된다. 그리고 이전 노드와 다음 노드를 가리키는 변수는 prev (previous)와 next 또는 bk (back)와 fd (forward)로 표현된다. 이렇게 이전 노드와 다음 노드를 가리키는 부분이 존재하기 때문에 단일 연결 리스트와 다르게 순회하는 방향을 조절할 수 있게 된다. 즉, 앞에서부터 순회할 수도 있고, 뒤에서부터 순회할 수도 있다. 이는 단일 연결 리스트에서의 오름차순과 내림차순 문제를 정렬 문제에서 읽는 순서의 문제로 단순화 시킨다. 하지만 임의의 노드에 접근하는데는 여전히 단일 연결 리스트와 같이 O(n)이다.


# 구현 방식
 여기서는 C 언어를 사용하여 이중 연결 리스트를 구현한다. 이때 노드 구성 방법, 노드 연결 방법, 리스트 라이브러리 인터페이스, 에러 처리, CRUD 구현에 대해 고민할 필요가 있다.


## 노드 구성: non-intrusive vs intrusive
 먼저 노드를 어떻게 구성할지 고민해보자. 여기에는 [2]가 설명하듯이, 두 가지 방식이 있다. 하나는 노드에 이전과 다음 노드를 가리키는 변수를 포함시키는 것 (non-intrusive)이고, 다른 하나는 노드와 이전과 다음 노드를 가리키는 변수를 분리시키는 것 (intrusive)이다. 전자는 일반적으로 다음과 같은 형태를 가지고,
 
```C
struct node {
    /* some data */
	
    struct node *prev;
    struct node *next;
};
```

후자는 일반적으로 다음과 같은 형태를 가진다.

```C
struct node {
    struct node *prev;
    struct node *next;
};

struct data {
    /* some data */
	
    struct node link;
};
```

전자는 일반적으로 흔히 사용되는 이중 연결 리스트 노드의 형태이고, 사용자가 전달한 인자를 할당된 노드에 복사한다. 따라서 메모리 할당이 2회 (사용자 데이터, 이중 연결 리스트의 노드) 발생하게 된다. 반면 후자는 노드에 사용자 데이터가 포함된 것이 아니라 사용자 데이터에 이전과 다음 노드를 가리키기 위한 변수가 포함되어 (embedded) 있다. 따라서 사용자 데이터를 위한 메모리를 할당하는 것만으로 노드가 만들어진다. 메모리 할당 횟수가 적다는 것은 그만큼 메모리 할당 에러 확인 횟수가 적어진다는 것을 의미한다.

 여기서는 intrusive한 방식으로 노드를 구성한다. 이때 노드의 구조는 다음과 같이 j_dllnode.h 헤더에 존재하고,
 
```C
struct j_dllnode {
    struct j_dllnode *prev;
    struct j_dllnode *next;
};
```

사용자는 이를 데이터에 포함시켜야 한다. 즉, 라이브러리는 사용자가 위 노드 구조체를 데이터에 포함시켰다고 가정한다. 여기서 사용자는 이 라이브러리를 사용하여 애플리케이션을 작성하는 개발자를 말한다.

 이러한 노드 구성은 노드에서 데이터를, 데이터에서 노드를 얻는 방법이 non-intrusive한 노드 구성보다 복잡해지도록 만든다. 일반적인 (generic) 데이터를 이중 연결 리스트에 저장하고자 할 때 이를 구현하는 방법은 주어진 데이터 타입에서 노드의 오프셋을 구하는 것이다. 다만, 여기서 오프셋은 사용자가 적절한 값으로 제공해야만 한다. 여기서 [4, Sec. 7.17]은 구조체 멤버의 (구조체의 시작 주소로부터의) 오프셋을 구하기 위한 offsetof 매크로를 정의한다. 따라서 사용자는 이 매크로를 사용하여 데이터에 포함된 노드 구조체의 오프셋을 제공할 수 있고, 제공해야만 한다. 이 값이 data_offset이라는 attribute로 저장된다고 가정하면, 이로부터 다음과 같이 노드에서 데이터를, 데이터에서 노드를 구하는 함수를 다음과 같이 작성할 수 있다.
 
```C
/* j_dll_get_data: get the data from the node */
static void *j_dll_get_data(j_dll_t *self, struct j_dllnode *node)
{
    if (node == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_NODE;
	return NULL;
    }
    return ((char *) node - self->data_offset);
}

/* j_dll_get_node: get the node from the data */
static struct j_dllnode *j_dll_get_node(j_dll_t *self, void *data)
{
    if (data == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_DATA;
	return NULL;
    }
    return (struct j_dllnode *) ((char *) data + self->data_offset);
}
```


## 노드를 연결하는 방법
 노드를 연결하는 방법에는 메모리 주소를 사용하는 방법도 있고, [6]이 제시하는 방법과 같이 XOR을 사용하는 방법 등이 있다. 여기서는 구현의 편의성을 위해 메모리 주소를 사용하여 노드를 연결한다.


## 사용자 인터페이스: 클래스
 여기서는 이중 연결 리스트와 관련된 변수, 함수를 모아서 리스트 클래스를 구성한다. 이렇게 리스트 클래스를 별도로 두는 이유는 여러 리스트를 관리할 때 편리하기 때문이다. 그리고 코드상에서도 어떤 리스트상에서 동작하고 있는 것인지 쉽게 알 수 있으므로 유지보수에도 용이할 것으로 보인다. C 언어는 언어 차원에서 클래스를 지원하지 않으므로 함수 포인터를 사용하여 구현해야 한다. 그러나 이러한 함수 포인터들도 초기에는 어떤 함수의 주소로 초기화되어야 한다. 따라서 초기화 함수가 필요하고, 제거 함수도 필요하다. 지금까지 설명한 것을 j_dll.h 헤더에 구성하면 다음과 같다.
 
```C
#ifndef __J_DLL_H__
#define __J_DLL_H__

#include <stdint.h>

#include "j_dllnode.h"

/* user defined method length and indexes */
#define J_DLLNODE_MTABLE_LEN 3
enum j_dllnode_mtable_idx {
    J_DLLNODE_READ = 00,
    J_DLLNODE_UPDATE = 01,
    J_DLLNODE_COMPARE = 02
};

/* list structure for intrusive double linked list */
typedef struct _j_dll {
    struct j_dllnode *head, *tail;
    size_t cnt;

    /*
     * In intrusive double linked list, we need offset of the node
     * structure to access the data from the node and the node from
     * the data. And user shall provide this.
     */
    size_t data_offset;

    /*
     * User shall provide read, update, and compare methods. Since these will
     * be applied to the all nodes in the dll, the dll structure should have
     * the method table.
     */
    struct j_dllnode_mtable {
	int (*method)(struct _j_dll *, struct j_dllnode *, void *);
    } mt[J_DLLNODE_MTABLE_LEN];

    int (*is_empty)(struct _j_dll *);
    int (*is_full)(struct _j_dll *);
    void *(*get_data)(struct _j_dll *, struct j_dllnode *);
    struct j_dllnode *(*get_node)(struct _j_dll *, void *);

    /*
     * User shall implement the compare method's return value as
     * -1 (less than), 0 (equal), 1 (greater than).
     */
    struct j_dllnode *(*search)(struct _j_dll *,
				void *,
				int);

    int (*read)(struct _j_dll *, struct j_dllnode *, void *);
    int (*update)(struct _j_dll *, void *, void *);
    int (*create)(struct _j_dll *, void *);
    int (*delete)(struct _j_dll *, void *);
} j_dll_t;

enum j_dll_flags {
    J_DLL_ERR_ALLOC = 1,	/* malloc error */
    J_DLL_ERR_INVALID_METHOD = 2, /* NULL method */
    J_DLL_ERR_INVALID_DLL = 4,	  /* NULL dll */
    J_DLL_ERR_INVALID_DATA = 8,	  /* NULL data */
    J_DLL_ERR_INVALID_NODE = 16 /* NULL node */
};

/* type define for user defined method */
typedef int (*j_dllnode_method_t)(j_dll_t *, struct j_dllnode *, void *);

extern j_dll_t *j_dll_init(j_dllnode_method_t read,
			   j_dllnode_method_t update,
			   j_dllnode_method_t compare,
			   size_t data_offset);
extern void j_dll_destroy(j_dll_t *dll);

#endif
```

위 헤더에서 주목해야 하는 것은 사용자 정의 메서드이다. 여기서는 일반적인 데이터를 이중 연결 리스트에 저장하고자 하므로 사전에 데이터 타입 또는 데이터 구성을 알 수 없다. 따라서 사용자는 데이터를 읽거나, 비교하거나, 갱신하는 방법을 제공해야만 한다. 그리고 위 헤더에서 에러 플래그는 아직 설명하지 않은 부분이지만, 다음 섹션에서 설명할 것이다.
 
 이제 클래스에서의 메서드와 그 구현부를 살펴보자. 리스트 클래스에서 메서드의 구현부는 외부에 드러날 필요가 없다. 즉, 구현부를 external linkage로 선언할 이유가 없다. 따라서 메서드의 구현부와 메서드가 호출하는 내부 함수의 구현부는 static linkage로 선언하였다. 그러나 초기화 함수와 제거 함수는 사용자가 직접 호출해야 하는 함수들이다. 따라서 이들은 외부에 드러날 필요가 있다. 그래서 이들은 external linkage로 선언하였다. 다만, 초기화 함수는 함수 포인터로 구현되는 메서드를 초기화하기 위해 메서드의 구현부를 알 수 있는 상태이어야 한다. 다시 말하면, 같은 파일에 있어야 하는 것이다. 또한 제거 함수도 내부 함수를 사용하므로 같은 파일에 있어야 한다. 지금까지 설명한 것을 j_dll.c에 작성하면 다음과 같다.
 
```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stddef.h>
#include <stdint.h>
#include "j_dll.h"
#include "j_error.h"

/* j_dll_is_empty: check the self is empty */
static int j_dll_is_empty(j_dll_t *self)
{
    return (self->cnt == 0);
}

/* j_dll_is_full: check the self is full */
static int j_dll_is_full(j_dll_t *self)
{
    return (self->cnt == SIZE_MAX);
}

/* j_dll_get_data: get the data from the node */
static void *j_dll_get_data(j_dll_t *self, struct j_dllnode *node)
{
    if (node == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_NODE;
	return NULL;
    }
    return ((char *) node - self->data_offset);
}

/* j_dll_get_node: get the node from the data */
static struct j_dllnode *j_dll_get_node(j_dll_t *self, void *data)
{
    if (data == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_DATA;
	return NULL;
    }
    return (struct j_dllnode *) ((char *) data + self->data_offset);
}

/*
 * prv_ functions are not complete. They are called by the caller and
 * and have some assumptions that shall be satisfied.
 */

/* prv_j_dll_search_node: search the start that makes cmpres with data */
static struct j_dllnode *prv_j_dll_search_node(j_dll_t *dll,
					       struct j_dllnode *start,
					       void *data,
					       int cmpres)
{
...
}

/* j_dll_search: search the start that makes cmpres with data */
static struct j_dllnode *j_dll_search(j_dll_t *self,
				      void *data,
				      int cmpres)
{
...
}

/* prv_j_dll_read_nodes: read the data and print it to the stdin or buf */
static int prv_j_dll_read_nodes(j_dll_t *dll,
			       struct j_dllnode *start,
			       void *buf)
{
...
}

/*
 * j_dll_read: read the data that the node is embedded and print it
 * to the stdin or buf
 */
static int j_dll_read(j_dll_t *self, struct j_dllnode *node, void *buf)
{
...
}

/* prv_j_dll_insert_node_at_the_end: insert the target at the end of the dll */
static void prv_j_dll_insert_node_at_the_end(j_dll_t *dll,
					  struct j_dllnode *curr,
					  struct j_dllnode *target)
{
...
}

/* prv_j_dll_insert_node: insert the target in front of the curr */
static void prv_j_dll_insert_node(j_dll_t *dll,
			       struct j_dllnode *curr,
			       struct j_dllnode *target)
{
...
}

/* prv_j_dll_insert: insert the node to the dll */
static void prv_j_dll_insert(j_dll_t *dll, struct j_dllnode *node)
{
...
}

/* j_dll_create: create the node in the self */
static int j_dll_create(j_dll_t *self, void *data)
{
...
}

/* prv_j_dll_unlink_node: unlink the node from the dll */
static void prv_j_dll_unlink_node(j_dll_t *dll, struct j_dllnode *node)
{
...
}

/* prv_j_dll_delete_node: delete the node from the dll */
static void prv_j_dll_delete_node(j_dll_t *dll, struct j_dllnode *node)
{
...
}

/* j_dll_delete: delete the node that has corresponding data from the self */
static int j_dll_delete(j_dll_t *self, void *data)
{
...
}

/* j_dll_update: update the data that has corresponding origin to new */
static int j_dll_update(j_dll_t *self, void *origin, void *new)
{
...
}

/* j_dll_init: initialize dll */
j_dll_t *j_dll_init(j_dllnode_method_t read,
		    j_dllnode_method_t update,
		    j_dllnode_method_t compare,
		    size_t data_offset)
{
    j_dll_t *dll;

    if (read == NULL || update == NULL || compare == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_METHOD;
	return NULL;
    }

    if ((dll = malloc(sizeof(*dll))) == NULL) {
	j_dll_errno = J_DLL_ERR_ALLOC;
	return NULL;
    }

    dll->head = NULL;
    dll->tail = NULL;
    dll->cnt = 0;
    dll->data_offset = data_offset;
    dll->mt[J_DLLNODE_READ].method = read;
    dll->mt[J_DLLNODE_UPDATE].method = update;
    dll->mt[J_DLLNODE_COMPARE].method = compare;
    dll->is_empty = j_dll_is_empty;
    dll->is_full = j_dll_is_full;
    dll->get_data = j_dll_get_data;
    dll->get_node = j_dll_get_node;
    dll->search = j_dll_search;
    dll->read = j_dll_read;
    dll->update = j_dll_update;
    dll->create = j_dll_create;
    dll->delete = j_dll_delete;
    return dll;
}

/* j_dll_destroy: destroy dll */
void j_dll_destroy(j_dll_t *dll)
{
    struct j_dllnode *target;

    if (dll == NULL)
	return ;
    
    if (dll->is_empty(dll)) {
	free(dll);
	return ;
    }

    target = dll->head;
    while (target->next != NULL) {
	target = target->next;
	prv_j_dll_delete_node(dll, target->prev);
    }
    prv_j_dll_delete_node(dll, target);
    free(dll);
}
```

## 에러 처리
 리스트 연산에서 발생하는 에러는 대부분 전달된 인자의 값이 정상적이지 않은 경우이며, 인자의 대부분이 포인터이므로 그 값이 NULL인 경우가 된다. 이러한 에러에는 적절하지 못한 사용자 정의 메서드, 적절하지 못한 DLL, 적절하지 못한 데이터, 적절하지 못한 노드가 있다. 이들은 NULL에 대한 역참조 (dereference)를 발생시켜 시스템으로 하여금 세그멘테이션 폴트를 발생시키도록 만든다. 인자값 에러 이외에는 할당 에러가 있다. DLL을 초기화할 때 클래스 자체는 동적할당되므로 malloc 함수가 NULL을 반환할 때도 생각해야 하는 것이다. 상기에 설명한 에러 코드는 다음과 같이 enum을 사용하여 정의된다.
 
```C
enum j_dll_flags {
    J_DLL_ERR_ALLOC = 1,	/* malloc error */
    J_DLL_ERR_INVALID_METHOD = 2, /* NULL method */
    J_DLL_ERR_INVALID_DLL = 4,	  /* NULL dll */
    J_DLL_ERR_INVALID_DATA = 8,	  /* NULL data */
    J_DLL_ERR_INVALID_NODE = 16 /* NULL node */
};
```
 
 지금까지 설명한 에러들을 처리하는 방법은 함수가 특정 에러값을 반환하도록 하고, 세부적인 에러 코드는 j_dll_errno라는 전역 변수에 저장하는 것이다. 여기서 특정 에러값은 반환 타입이 int라면 -1이고, 포인터라면 NULL이다. 이는 세부적인 에러 코드를 반환하는 것보다 에러 검사 코드를 간결하게 만들어준다는 장점이 있다.
 

## CRUD (Create, Read, Update, Delete)
 [1, Sec. 3.1]은 노드를 리스트에 추가할 때 정렬할 필요가 없다면, 리스트를 순회할 필요가 없는 방법을 선택하는 것이 가장 간단하다고 설명한다. 그리고 노드를 삭제할 때는 삭제할 노드를 찾기 위해서 함수가 필요하며, 리스트의 끝 (단일 연결 리스트에서는 맨 앞, 이중 연결 리스트에서는 맨 앞과 맨 끝) 노드를 삭제할 때의 경우를 특별히 다루어야 함을 설명한다. 여기서 노드를 찾는 함수는 재귀적으로 문제를 줄여 나가면서 (검색해야 하는 리스트의 크기를 줄이면서) 해결하는 방법을 선택하는데, 이 방법이 보다 간단한 리스트 연산을 만들고, 효율적인 분할 정복 (divide-and-conquer) 알고리즘으로 안내한다고 설명한다.
 
 여기서는 이중 연결 리스트의 노드를 다루며 노드를 추가할 때는 오름차순으로 정렬한다. 그래서 리스트의 양 끝에 노드를 삽입하는 방법을 쓸 수 없고, 노드를 추가할 위치를 찾기 위해 리스트를 순회해야만 한다. 이는 삭제, 갱신도 마찬가지이다. 이때 노드를 찾는 함수는 재귀적으로 구현하되 [3, Sec. 1.2.1]이 설명하는 선형 반복 프로세스 (linear iterative process)의 형태로 구현한다. 이러한 선형 반복 프로세스는 흔히 꼬리 재귀 (tail-recursive) 최적화라고 불리는 기법을 적용할 수 있는 형태이다.
 
 선형 반복 프로세스로 구현된 검색 함수는 다음과 같다.
 
```C
/* prv_j_dll_search_node: search the start that makes cmpres with data */
static struct j_dllnode *prv_j_dll_search_node(j_dll_t *dll,
					       struct j_dllnode *start,
					       void *data,
					       int cmpres)
{
    if (start == NULL ||
		dll->mt[J_DLLNODE_COMPARE].method(dll, start, data) == cmpres)
	return start;
    prv_j_dll_search_node(dll, start->next, data, cmpres);
}
```

```C
/* j_dll_search: search the start that makes cmpres with data */
static struct j_dllnode *j_dll_search(j_dll_t *self,
				      void *data,
				      int cmpres)
{
    if (data == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_DATA;
	return NULL;
    }

    return prv_j_dll_search_node(self, self->head, data, cmpres);
}
```

 위와 같은 검색 함수는 특정 노드를 찾는 것뿐만 아니라 노드 생성 시, 노드를 삽입할 위치도 찾을 수 있다는 것을 기억하자. 이는 검색 함수가 인자로 전달된 데이터와 노드를 비교할 때 같은 경우 (반환값이 0인 경우)만을 검사하는 것이 아니라 전달된 cmpres와 같은지 검사하기 때문이다. 이때 주의해야할 것은 사용자는 비교 함수를 정의할 때 반환값을 -1 (해당 노드의 데이터가 전달된 데이터보다 작은 경우), 0 (해당 노드의 데이터가 전달된 데이터와 같은 경우), 1 (해당 노드의 데이터가 전달된 데이터보다 큰 경우)로 정의해야 한다는 것이다. 그럼 이제 검색 함수가 노드 생성, 삭제, 갱신에서 어떻게 사용되는지 알아보자.
 
 노드 생성을 수행하는 함수는 다음과 같다.
 
```C
/* prv_j_dll_insert_node_at_the_end: insert the target at the end of the dll */
static void prv_j_dll_insert_node_at_the_end(j_dll_t *dll,
					  struct j_dllnode *curr,
					  struct j_dllnode *target)
{
    dll->tail = target;
    target->prev = curr;
    curr->next = target;
}

/* prv_j_dll_insert_node: insert the target in front of the curr */
static void prv_j_dll_insert_node(j_dll_t *dll,
			       struct j_dllnode *curr,
			       struct j_dllnode *target)
{
    if (curr == dll->head) {
	dll->head = target;
	target->next = curr;
	curr->prev = target;
	dll->tail = (dll->cnt == 1) ? curr : dll->tail;
    } else {
	target->prev = curr->prev;
	target->next = curr;
	curr->prev->next = target;
	curr->prev = target;
    }
}

/* prv_j_dll_insert: insert the node to the dll */
static void prv_j_dll_insert(j_dll_t *dll, struct j_dllnode *node)
{
    struct j_dllnode *target;

    if ((target = dll->search(dll,
			      dll->get_data(dll, node),
			      1)) == NULL)
	prv_j_dll_insert_node_at_the_end(dll, dll->tail, node);
    else
	prv_j_dll_insert_node(dll, target, node);
}
```
 
```C
/* j_dll_create: create the node in the self */
static int j_dll_create(j_dll_t *self, void *data)
{
    struct j_dllnode *node;

    if (data == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_DATA;
	return -1;
    }

    node = self->get_node(self, data);
    node->prev = NULL;
    node->next = NULL;
    if (self->is_empty(self)) {
	self->head = node;
	self->tail = node;
    } else
	prv_j_dll_insert(self, self->get_node(self, data));
    self->cnt++;
    return 0;
}
```

위 코드의 prv_j_dll_insert()가 검색 메서드를 호출할 때 네 번째 인자로 1을 전달한다는 것에 주목하자. 이는 전달된 데이터보다 큰 노드를 검색하라는 의미이며, 곧 오름차순으로 노드를 삽입할 때 삽입할 위치를 검색하는 것과 같다. 이러한 노드 생성 함수에서 재귀적인 부분은 prv_j_dll_insert()가 검색 메서드를 호출할 때 뿐이다.

 이제 노드 삭제를 수행하는 함수를 살펴보자. 노드 삭제를 수행하는 함수의 코드는 다음과 같다.
 
```C
/* prv_j_dll_unlink_node: unlink the node from the dll */
static void prv_j_dll_unlink_node(j_dll_t *dll, struct j_dllnode *node)
{
    struct j_dllnode *head, *tail;

    head = dll->head;
    tail = dll->tail;
    if (head == tail) {		/* there is only one node in the dll */
	dll->head = NULL;
	dll->tail = NULL;
    } else if (node == head) {
	dll->head = node->next;
	node->next->prev = NULL;
	node->next = NULL;
    } else if (node == tail) {
	dll->tail = node->prev;
	node->prev->next = NULL;
	node->prev = NULL;
    } else {
	node->prev->next = node->next;
	node->next->prev = node->prev;
	node->prev = node->next = NULL;
    }
}

/* prv_j_dll_delete_node: delete the node from the dll */
static void prv_j_dll_delete_node(j_dll_t *dll, struct j_dllnode *node)
{
    prv_j_dll_unlink_node(dll, node);
    dll->cnt--;
    free(dll->get_data(dll, node));
}
```

```C
/* j_dll_delete: delete the node that has corresponding data from the self */
static int j_dll_delete(j_dll_t *self, void *data)
{
    struct j_dllnode *target;

    if (data == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_DATA;
	return -1;
    }

    if ((target = self->search(self, data, 0)) == NULL)
	return -1;

    prv_j_dll_delete_node(self, target);
    return 0;
}
```

노드를 삭제하는 함수는 삭제할 노드를 찾기 위해 검색 메서드를 사용한다. 그리고 노드 생성 함수에서와 같이 검색 메서드가 노드를 삭제하는 과정에서 유일하게 재귀적인 부분이다.

 마지막으로 노드의 데이터를 갱신하는 함수를 살펴보자.

```C
/* j_dll_update: update the data that has corresponding origin to new */
static int j_dll_update(j_dll_t *self, void *origin, void *new)
{
    struct j_dllnode *target;
    void *data;
    
    if (origin == NULL || new == NULL) {
	j_dll_errno = J_DLL_ERR_INVALID_DATA;
	return -1;
    }

    if ((target = self->search(self, origin, 0)) == NULL)
	return -1;

    prv_j_dll_unlink_node(self, target);
    if (self->mt[J_DLLNODE_UPDATE].method(self, target, new) == -1) {
	prv_j_dll_insert(self, target);
	return -1;
    }
    prv_j_dll_insert(self, target);
    return 0;
}
```

노드를 갱신하는 함수는 노드 삭제 함수가 삭제할 노드를 찾듯이, 갱신할 노드를 찾기 위해 검색 함수를 사용한다. 이때 노드의 연결을 잠시 끊어 놓는데, 이는 갱신 후에 다시 삽입하여 리스트의 정렬 일관성을 유지하기 위함이다.


# References
[1] Steven S. Skiena, "The Algorithm Design Manual", Springer, 2008

[2] "Intrusive and non-intrusive containers", boost C++ LIBRARIES. [Online]. Available: https://www.boost.org/doc/libs/1_35_0/doc/html/intrusive/intrusive_vs_nontrusive.html, [Accessed Dec. 27, 2022]

[3] Harold Abelson, Gerald Jay Sussman, Julie Sussman, "Structure and Interpretation of Computer Programs", MIT Press, 1984

[4] ISO/IEC JTC 1/SC 22/WG 14, "ISO/IEC 9899:1999, Programming Languages -- C", ISO/IEC, 1999

[5] Allen Newell, J. C. Shaw, "Programming the Logic Theory Machine", Proceedings of the Western Joint Computer Conference, pp. 230-240, 1957

[6] Prokash Sinha, "A Memory-Efficient Doubly Linked List", LINUX JOURNAL, 2004
