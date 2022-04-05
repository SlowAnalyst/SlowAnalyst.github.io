---
layout: post
title: GNU malloc internals
---

# Malloc 개요
 GNU C 라이브러리의 malloc 라이브러리는 어플리케이션의 주소 공간에서
할당된 메모리를 관리하는 몇 개의 함수를 가진다. Glibc malloc은
ptmalloc (pthreads malloc) 으로부터 파생되었고, 그것은 dlmalloc
(Doug Lea malloc) 으로부터 파생되었다. 이 malloc은 "힙 (heap)"
스타일의 malloc이고, 그것은 거대한 메모리 영역 ("힙")
에 다양한 크기의 청크 (chunks) 가 존재한다는 것이며, 비트맵과
배열, 또는 동일한 크기의 블록 등에 대한 구현과 반대된다는 것이다.

 이제 본 글에서 자주 사용되는 용어들을 정의할 것이다.

* Arena:
* Heap:
* Chunk:
* Memory:
