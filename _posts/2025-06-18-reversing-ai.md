---
layout: post
title: "리버싱에서의 AI 사용"
summary: "AI에게 맡긴 디컴파일된 코드 분석"
author: eveheeero
date: '2025-06-18 02:40:00 +0900'
category: ['reversing', 'learning']
tags: reversing
thumbnail: /assets/img/posts/2025-06-18-0.png
keywords: reversing
usemathjax: false
permalink: /blog/reversing-ai/
---


## 배경

디컴파일된 소스코드를 보던 중 막히는 구간이 있어 오래 시간이 잡아먹혔습니다.

리버싱같은 분야는 하나의 코드가 오래 지속되지도 않으며 커뮤니티에 도움을 청하기도 애매한 분야입니다.

따라서 디컴파일 소스코드 분석 중 막히게 되면 오랜 시간을 소비하게 되는데

이번 분석이 막혔을 때 AI가 생각나서 맡겨보았습니다.

## 요약

- 충분히 AI에게 디컴파일된 소스코드 해석을 맡겨볼만함 (`gemini 2.5 pro` 기준)

## 문제 상황

![포트란으로 짜여진 프로그램의 디컴파일된 소스코드](/assets/img/posts/2025-06-18-0.png){: style="max-width: 100%; height: auto;"}

CTF 문제를 풀던 중 포트란으로 나온 프로그램을 디컴파일하게 되었습니다.

리버싱을 잡지 않은지 오래 되어서 분석에 어려움이 있었습니다.

특히 `v39`에 들어가는 값을 계산하기 힘들었습니다.

## 분석 과정

![gemini 입력](/assets/img/posts/2025-06-18-1.png){: style="max-width: 100%; height: auto;"}

`gemini 2.5 pro`에 디컴파일된 소스코드를 주고 해석을 맡겼습니다.

| Gem시스템을 통해 리버싱과 관련된 대화를 하겠다고만 넣어둔 상태입니다.

아래는 디컴파일된 소스코드 전문입니다.

```c++
__int64 sub_1246()
{
  __int64 v0; // rdx
  __int64 v1; // rcx
  __int64 v2; // r8
  __int64 v3; // r9
  __int64 v4; // rdx
  __int64 v5; // rcx
  __int64 v6; // r8
  __int64 v7; // r9
  int input_len; // eax
  __int64 v9; // rdx
  __int64 v10; // rcx
  __int64 v11; // r8
  __int64 v12; // r9
  _BYTE *v13; // rsi
  int input_len2; // ebx
  __int64 v15; // rdx
  __int64 v16; // rcx
  __int64 v17; // r8
  __int64 v18; // r9
  __int64 v19; // rdx
  __int64 v20; // rcx
  __int64 v21; // r8
  __int64 v22; // r9
  __int64 i; // rbx
  int v25; // [rsp+0h] [rbp-310h] BYREF
  int v26; // [rsp+4h] [rbp-30Ch]
  const char *v27; // [rsp+8h] [rbp-308h]
  int v28; // [rsp+10h] [rbp-300h]
  char *v29; // [rsp+50h] [rbp-2C0h]
  __int64 v30; // [rsp+58h] [rbp-2B8h]
  char v31; // [rsp+21Fh] [rbp-F1h] BYREF
  __int64 v32; // [rsp+220h] [rbp-F0h]
  unsigned int v33; // [rsp+22Ch] [rbp-E4h] BYREF
  _BYTE input[72]; // [rsp+230h] [rbp-E0h] BYREF
  unsigned int v35; // [rsp+278h] [rbp-98h] BYREF
  _DWORD index[30]; // [rsp+27Ch] [rbp-94h] BYREF
  int v37; // [rsp+2F4h] [rbp-1Ch]
  int min_len; // [rsp+2F8h] [rbp-18h]
  int v39; // [rsp+2FCh] [rbp-14h]

  v32 = 0LL;
  sub_19BD("*");
  v39 = sub_1954(dword_4020, &unk_201C);
  v39 = sub_190F(&v33);
  v39 = sub_18C2(&dword_2020);
  sub_185D(input, dword_4020, 64LL);
  v27 = "bfl.f90";
  v28 = 21;
  v25 = 128;
  v26 = 6;
  (_gfortran_st_write)(&v25, dword_4020, v0, v1, v2, v3);
  _gfortran_transfer_character_write(&v25, "key:(A)", 4LL);
  _gfortran_st_write_done(&v25);
  v27 = "bfl.f90";
  v28 = 22;
  v29 = "(A)";
  v30 = 3LL;
  v25 = 4096;
  v26 = 5;
  (_gfortran_st_read)(&v25, "key:(A)", v4, v5, v6, v7);
  _gfortran_transfer_character(&v25, input, 64LL);
  _gfortran_st_read_done(&v25);
  min_len = 24;
  input_len = _gfortran_string_len_trim(64LL, input);
  if ( min_len > input_len )
  {
    v27 = "bfl.f90";
    v28 = 26;
    v25 = 128;
    v26 = 6;
    (_gfortran_st_write)(&v25, input, v9, v10, v11, v12);
    _gfortran_transfer_character_write(&v25, "*", 0LL);
    _gfortran_st_write_done(&v25);
    _gfortran_stop_string(0LL, 0LL, 0LL);
  }
  v33 = 0;                                      // above is getting input, min 24 max 64
  v13 = input;
  input_len2 = _gfortran_string_len_trim(64LL, input);
  for ( index[0] = 1; input_len2 >= index[0]; ++index[0] )
  {
    v33 += input[index[0] - 1];
    v13 = index;
    v39 = sub_17A4(&v33, index);
  }
  v39 = sub_19BD(&v33);
  v35 = v33;
  for ( index[0] = 1; index[0] <= 28; ++index[0] )
  {
    v39 = sub_18C2(&v35);
    v35 = (4919 * v35) ^ 0xDEADBEEF;
    index[index[0]] = dword_4020[index[0] - 1] ^ v35;
    v39 = sub_190F(&v35);
  }
  for ( index[0] = 1; index[0] <= 28; ++index[0] )
  {
    v37 = index[index[0]];
    if ( v37 <= 31 || v37 > 126 )
    {
      sub_11C9();
      v27 = "bfl.f90";
      v28 = 49;
      v25 = 128;
      v26 = 6;
      (_gfortran_st_write)(&v25, v13, v15, v16, v17, v18);
      _gfortran_transfer_character_write(&v25, "*", 0LL);
      _gfortran_st_write_done(&v25);
      v13 = 0LL;
      _gfortran_stop_string(0LL, 0LL, 0LL);
    }
  }
  v39 = sub_19BD(&unk_201C);
  v27 = "bfl.f90";
  v28 = 55;
  v25 = 128;
  v26 = 6;
  (_gfortran_st_write)(&v25, v13, v19, v20, v21, v22);
  for ( i = 1LL; i <= 28; ++i )
  {
    v31 = index[i];
    _gfortran_transfer_character_write(&v25, &v31, 1LL);
  }
  return _gfortran_st_write_done(&v25);
}
```

아래는 각 블록 별 Gemini의 해석 요약입니다. (라인별 해석도 출력해주었습니다)

- 이 블록은 프로그램의 초기 설정을 수행하고, 사용자에게 "key:(A)"라는 프롬프트를 보여주며 입력을 기다립니다. 사용자가 입력한 문자열은 input 변수에 저장되며, 길이는 최소 24자... (생략)
- 이 블록은 사용자가 입력한 문자열의 각 문자에 대한 ASCII 코드 값을 모두 더하여 v33이라는 변수에 저장합니다. 이 합산된 값(v33)은 다음 단계의 연산에서 시드(seed) 값으로 사용됩니다.
- 이 블록이 핵심적인 데이터 변환(암호화)을 수행합니다. (이후 자세한 설명)
- 이 블록은 암호화 연산의 결과물인 index 배열의 28개 값이 모두 출력 가능한 ASCII 문자 범위... (생략)
- 모든 검증 과정을 통과했다면, 이 블록은 최종적으로 생성된 28자의 문자열을 화면에 출력합니다.

아래는 막혔던 내용에 대한 해석입니다.

```text
Q. v39의 역할은?
A. v39 변수는 함수의 반환 값을 임시로 저장하는 용도로 사용되지만, 그 값이 후속 연산에 전혀 사용되지 않으므로 프로그램의 핵심 로직에는 실질적인 역할을 하지 않습니다.
A. 이후 상세 설명...

Q. v39에 값을 넣는 콜들이 있었는데 내부에서 아무런 연산을 하지 않음. 이건 포트란 언어의 특징인가?
A. 아닙니다, 이것은 포트란 언어 자체의 고유한 특징이라기보다는 컴파일(compile) 및 디컴파일(decompile) 과정에서 발생하는 일반적인 현상에 가깝습니다.
A. 이후 상세 설명
# 위 내용은 신뢰성이 매우 떨어져보입니다.
```

[대화 링크](https://g.co/gemini/share/ce4cfac9161f)

## 작성자의 글

- 리버싱 분야는 데이터가 없을 줄 알아 ai에 대한 신뢰도가 바닥이었으나, 생각보다 제대로 해석해주고 있습니다.

## 참조

- [대화 링크](https://g.co/gemini/share/ce4cfac9161f)
