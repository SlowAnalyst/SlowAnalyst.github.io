---
layout: post
title: "Bit-Wise Analysis Of The Collatz Conjecture"
summary: ""
author: david232818
date: '2023-05-16 10:40:23 +0900'
category: ['learning']
# tags: 
thumbnail: /assets/img/posts/default.png
keywords: Collatz Conjecture, bit
usemathjax: true
permalink: /blog/Colatz_Conjecture/
---


Contents
--------
1. Introduction
2. 3x + 1 Problem
3. Bit-wise Analysis
   1. Rightmost Bit Pattern: 1000...01
   2. Rightmost Bit Pattern: 1111...11
4. Conclusion
5. References



## Introduction

콜라츠 추측이라고도 알려져 있는 3x + 1 문제는 홀수인 정수 n을 3n + 1로, 짝수인 정수 n을 n / 2로 만드는 함수의 반복이 만들어내는 동작에 관심을 갖는다. 3x + 1 추측이 주장하는 것은, 임의의 양의 정수 n에 대해 이 함수를 반복적으로 적용하면 결국 1에 도달한다는 것이다[1].



본 글에서는 3x + 1 문제를 분석하기 위해 비트를 사용한 방법을 제시한다. 물론 아직 이 방법으로 문제를 증명을 할 수 있는지는 명확하지 않지만, 분명히 통계적 방법과 다르다.

## 3x + 1 Problem

3x + 1 문제는 다음과 같이 정의하는 함수의 반복으로 표현된다[1]:

$$
\begin{equation}
T(n) = 
\begin{cases}
    \frac{3n + 1}{2}& \text{if } n \equiv 1 \pmod 2 \\
    \frac{n}{2}& \text{if } n \equiv 0 \pmod 2 \\
\end{cases}
\end{equation}
$$

## Bit-wise Analysis

임의의 양의 정수 $n$의 비트열 표현을 생각해보자. 여기서 우리는 이 비트열의 우측 끝 부분의 유형을 다음과 같이 세 가지로 나눌 수 있다:

$$
\begin{equation}
\begin{cases}
    1 \underbrace{000 \dots 0}_\text{N} 0_2 & \text{(i)} \\
    1 \underbrace{000 \dots 0}_\text{N} 1_2 & \text{(ii)} \\
    1 \underbrace{111 \dots 1}_\text{N} 1_2 & \text{(iii)} \\
\end{cases}
\end{equation}
$$

식 2의 i의 경우에는 자명하게 1에 도달함을 알 수 있다. 그럼 식 2의 ii, iii의 경우에도 1에 도달하는지 알아보자.

### Rightmost Bit Pattern: 1000...01

여기서 우리가 관심을 가지는 부분은 비트열의 우측 끝 부분의 1000...01(2)이다. 이러한 비트열에 대해 T(n)을 적용하면 다음을 얻는다:

$$
1 \underbrace{000 \dots 0}_\text{N} 1_2 \\
11 \underbrace{000 \dots 0}_\text{N - 2} 10_2 \\
11 \underbrace{000 \dots 0}_\text{N - 2} 1_2 \\
1001 \underbrace{000 \dots 0}_\text{N - 4} 10_2 \\
1001 \underbrace{000 \dots 0}_\text{N - 4} 1_2 \\
11011 \underbrace{000 \dots 0}_\text{N - 6} 10_2 \\
11011 \underbrace{000 \dots 0}_\text{N - 6} 1_2 \\

$$

위 식에서 각 비트열의 우측 끝 부분의 1000...01(2)에 주목하라. 그럼 1000...01(2)의 두 개의 1 사이에 있는 0의 개수가 두 개씩 감소함을 알 수 있다. 그럼 이를 일반화해보자. 먼저 홀수인 001(2)에 3을 곱하고 1을 더하면 100(2)이고 이를 2로 나누면 10(2)이다. 이때 10(2)는 짝수이므로 2로 나누면 1(2)이 된다. 따라서 

$$1 \underbrace{000 \dots 0}_\text{N} 1_2 (N \geq 2)$$

이면,  T(n)을 두 번 적용할 때마다 N은 2씩 감소한다. 이때, N이 짝수였다면 1001(2)까지 감소할 수 있을 것이고, 홀수였다면 10001(2)까지 감소할 수 있을 것이다. 그리고 11(2), 101(2)는 1에 도달함을 계산할 수 있다.


따라서 우측 끝 부분이 

$$1 \underbrace{000 \dots 0}_\text{N} 1_2 (N \geq 2)$$

일 때,  T(n)을 반복해서 적용하면 우측 끝 부분의 1000...01(2)인 부분이 줄어들다가 결국 1이 된다.



이제 

$$1 \underbrace{000 \dots 0}_\text{N} 1_2 (N \geq 2)$$

의 초기 길이인 N + 2의 변화를 살펴보자. 우측 끝 부분의 

$$1 \underbrace{000 \dots 0}_\text{N} 1_2 (N \geq 2)$$

는 좌측 끝 비트에는 “곱하기 3”만 적용되고, 우측 끝 비트에 “곱하기 3 더하기 1”이 적용된다. 이때 “곱하기 3”은 비트열에서 최대 2비트를 증가시킨다. 이는 다음과 같이 계산할 수 있다:

$$
n = \underbrace{111 \dots 1_2}_\text{N}  \\
3n = \underbrace{111 \dots 10_2}_\text{N + 1} + \underbrace{111 \dots 11_2}_\text{N} = \underbrace{1011 \dots 01_2}_\text{N + 2} \\
$$

그런데 

$$1 \underbrace{000 \dots 0}_\text{N} 1_2 (N \geq 2)$$

이면,  T(n)을 두 번 적용할 때마다 N은 2씩 감소하므로 T(n)을 반복적으로 적용하면 초기 길이인 N + 2보다 작아진다.

정리하면, 우측 끝 부분의 비트열이 

$$1 \underbrace{000 \dots 0}_\text{N} 1_2 (N \geq 2)$$

인 경우에는 T(n)을 반복적으로 적용함에 따라 1000...01(2)의 두 개의 1 사이에 있는 0의 개수가 감소하면서 동시에 전체 비트열의 길이도 감소한다.

### Rightmost Bit Pattern: 1111...11

그럼 이제 비트열의 우측 끝 부분이 1111...11(2)인 경우를 살펴보자. 이러한 비트열에 T(n)을 적용하면 다음을 얻는다:

$$
1 \underbrace{111 \dots 1}_\text{N} 1_2 \\
101 \underbrace{111 \dots 1}_\text{N - 1} 1_2 \\
10001 \underbrace{111 \dots 1}_\text{N - 2} 1_2 \\
110101 \underbrace{111 \dots 1}_\text{N - 3} 1_2 \\
10100001 \underbrace{111 \dots 1}_\text{N - 4} 1_2 \\
111100101 \underbrace{111 \dots 1}_\text{N - 5} 1_2 \\
10110110001 \underbrace{111 \dots 1}_\text{N - 6} 1_2 \\

$$

위 식에서 각 비트열의 우측 끝 부분의 1111...11(2)에 주목하라. 그럼 각 비트열의 1111...11(2)$ 부분의 개수가 감소함을 알 수 있다. 그럼 이를 일반화해보자. 먼저 

$$1 \underbrace{111 \dots 1}_\text{N} 1_2$$

에 T(n)을 적용하면

$$101 \underbrace{111 \dots 1}_\text{N - 1} 1_2$$

이 된다. 이때 연산이 진행됨에 따라 우측 끝 부분의 1111...11(2)의 앞에는 항상 0이 존재하게 된다. 이는 1111...11의 가장 왼쪽 부분의 1을 연산하는 과정에서 생기는 것이므로 우측 끝 부분의 1111...11(2)의 개수는 감소한다.



이제 

$$1 \underbrace{111 \dots 1}_\text{N} 1_2$$

의 길이인 N + 2를 살펴보자. 3n + 1 연산은 최대 2 비트를 증가시키고 그 결과는 짝수이므로 1 비트가 감소하므로 비트열의 전체 길이는 최대 1비트씩 증가한다. 이때 우측 끝 부분의 1111...11(2)는 감소하지만 그 앞에는 항상 우측 끝 부분의 0이 존재한다. 즉, 우측 끝 부분의 1111...11(2)이 1로 감소하면 해당 비트열은 식 2의 ii 유형이 된다. 이때 이 유형은 비트열의 전체 길이를 감소시킨다. 이는 비트열을 증가시키는 유형이 존재하지만 해당 유형은 종료될 것이고 종료되면 증가된 비트열은 감소할 것임을 의미한다.

## Conclusion

본 글에서는 3x + 1 문제에서의 임의의 수 n을 비트로 표현하여 문제를 분석하였고, 비트열이 증가하는 부분이 존재하지만 증가한 비트열은 감소할 것임을 보였다. 그러나 이렇게 증가한 비트열이 계속해서 감소할 것인지에 대해서는 추가해야하는 부분이 있는 것으로 보인다.

## References

1. Jeff Lagarias, “The 3x + 1 problem and its generalizations,” American Mathematical Monthly, vol. 92, pp. 3 — 23, 1985.
