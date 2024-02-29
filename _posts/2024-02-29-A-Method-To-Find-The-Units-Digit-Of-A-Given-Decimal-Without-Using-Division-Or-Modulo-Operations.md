---
layout: post
title: "A Method To Find The Units Digit Of A Given Decimal Without Using Division Or Modulo Operations"
summary: ""
author: david232818
category: ['learning']
thumbnail: /assets/img/posts/default.png
keywords: units digit
usemathjax: false
permalink: /blog/A-Method-To-Find-The-Units-Digit-Of-A-Given-Decimal
---


Contents
--------
1. Introduction
2. Problem Description and Analysis
3. Solutions
   1. Mapping units digit of each bit
3. Conclusion
4. Appendix
   1. Why if-else statements are counted?
0. References



## Introduction
나눗셈이나 나머지 연산은, 특히 반도체 칩에게는, 상대적으로 비싼 비용을 치러야 하는 연산이다. 그럼에도 반도체 칩 설계자는 이를 사용하면 간단히 해결되는 문제를 종종 맞닥뜨리게 된다. 이때 설계자는 나눗셈이나 나머지 연산을 구현해서 해결하거나, 또다른 방법을 찾아야만 한다. 


본 글에서는 이러한 유형의 문제 중 하나를 해결한 방법을 다룬다. 다만, 문제를 해결한 방법을 구현한 코드는 Verilog HDL이 아닌 C로 제공할 것이다.

## Problem Description and Analysis
본 글에서 다루고자 하는 문제는 다음과 같다: 

> 0 - 1155 범위의 숫자가 주어질 때 주어진 수보다 작거나 같은 5의 배수 중에서 가장 큰 양의 정수를 구하는 프로그램을 작성하라. 단, 나눗셈이나 나머지 연산을 사용해서는 안되며, 가능한 적은 if-else 문을 사용해야 한다.


위 문제는 주어진 수의 일의 자리를 구했을 때 얻은 수와 5의 차이만큼 주어진 수에서 빼서 해결할 수 있다. 다만, 나눗셈이나 나머지 연산을 사용하지 않아야 하므로 10으로 나눈 나머지로 일의 자리를 구하는 방법은 쓸 수 없다.


즉, 이 문제는 주어진 수의 일의 자리를 제약 조건을 만족하면서 구할 수 있다면 해결된다.

## Solutions
그럼 일의 자리를 구하는 방법을 알아보자. 여기에는 여러가지 방법이 있을 수 있으며, 그 중 간단한 방법으로는 단계적으로 특정 값을 빼는 것이 있다. 하지만 단계적으로 값을 빼는 방법은 if-else 문의 개수를 줄이는 방법에 대한 고민이 필요하므로 해결책이 되기 어렵다[1].


여기서는 각 비트의 일의 자리를 매핑하는 방법을 설명할 것이다.

### Mapping units digit of each bit
비트열의 각 비트는 십진수의 자릿수와 마찬가지로 자릿수를 가진다. Least significant bit (LSB)로부터 0번 째는 1 (2 ^ 0), 1번 째는 2 (2 ^ 1), 2번 째는 4 (2 ^ 2), 3번 째는 8 (2 ^ 3), ... 과 같은 식으로 갖게 된다. 그리고 이러한 자릿수를 모두 더하여 십진수로 변환할 수 있다.


여기서 일의 자리 이외의 자릿수는 관심 대상이 아니다. 즉, 최종적인 일의 자리에 해당하는 수는 비트열의 (1인 위치의) 각 자릿수의 일의 자리에 해당하는 수를 더하여 구할 수 있다. 그리고 자릿수의 일의 자리를 더해나갈 때, 그 값이 10을 초과하는지 매번 검사하여 초과한다면 10을 빼야할 것이다.


예를 들어, 23의 일의 자리를 구한다고 하자. 23은 이진수로 10111(2)이다. 이때 더해야 하는 자릿수의 일의 자리는 LSB로부터 1, 2, 4, 6이다. 이들을 모두 더하면 13이므로 (10을 초과하므로) 10을 빼면 일의 자리인 3을 얻는다.


그런데 본 문제에서 주어지는 수의 범위는 0 - 1155이므로 11 비트상에서 각 자릿수의 일의 자리를 switch case 문으로 매핑하고 이들을 더해나가면 된다. 그럼 일의 자리를 얻기 위해 하나의 switch case 문, 두 개의 if-else 문이 필요할 것으로 보인다. 이는 다음과 같이 C 코드를 작성해봄으로써 알 수 있다:

```C
#include <stdio.h>

unsigned int map_units_digit_of_each_bit(unsigned int n)
{
  unsigned int res;

  switch (n) {
  case 0x000:
    res = 0;
    break;
  case 0x001:
    res = 1;
    break;
  case 0x002:
    res = 2;
    break;
  case 0x004:
    res = 4;
    break;
  case 0x008:
    res = 8;
    break;
  case 0x010:
    res = 6;
    break;
  case 0x020:
    res = 2;
    break;
  case 0x040:
    res = 4;
    break;
  case 0x080:
    res = 8;
    break;
  case 0x100:
    res = 6;
    break;
  case 0x200:
    res = 2;
    break;
  case 0x400:
    res = 4;
    break;
  default:
    res = 0;
    break;
  }
  return res;
}

unsigned int get_units_digit(unsigned int n, unsigned int len)
{
  unsigned int mask, i, digit_1;

  mask = 1;
  digit_1 = 0;
  for (i = 0; i < len; i++) {
    if (n & mask)
      digit_1 += map_units_digit_of_each_bit(mask);
    if (digit_1 >= 0xa)
      digit_1 -= 0xa;
    mask <<= 1;
  }
  return digit_1;
}

unsigned int get_biggest_multiple_of_5_less_than_n(unsigned int n,
						   unsigned int bitlen)
{
  unsigned int digit_1, res;

  digit_1 = get_units_digit(n, bitlen);
  if (digit_1 == 0x0 || digit_1 == 0x5) {
    res = n;
  } else if (digit_1 < 0x5) {
    res = n - digit_1;
  } else if (digit_1 > 0x5) {
    res = n - (digit_1 - 0x5);
  }
  return res;
}

int main()
{
  unsigned int n;

  for (n = 0; n < 1156; n++)
    printf("%d => %d\n", n, get_biggest_multiple_of_5_less_than_n(n, 12));
  return 0;
}
```

## Conclusion
지금까지 나눗셈이나 나머지 연산을 사용하지 않고 일의 자리를 구하는 방법에 대해 살펴보았다. 비록 실제 문제는 주어진 수보다 작거나 같은 5의 배수 중에서 가장 큰 수를 구하는 것이지만, 이를 해결하기 위해 반드시 해결해야 하는 핵심 문제는 일의 자리를 구하는 것이었다.


물론, Verilog HDL이 아닌 C로 구현하였지만, Verilog HDL의 코딩 스타일을 알고 있다면 C 코드를 변환하는 것에 문제가 없을 것이라고 생각한다.

## Appendix

### Why if-else statements are counted?
이를 이해하기 위해서는 Verilog HDL이 어떻게 synthesis (compile)되는지 이해해야 한다. Verilog HDL 코드는 synthesis를 거쳐서 논리 게이트로 변환된다. 이때 이를 수행하는 툴 중 대표적인 것으로 Design Compiler (DC)가 있다. 여기서 일반적으로 multiplexer (MUX)는 Verilog HDL에서 if와 case (C에서의 switch 문) 문으로 표현된다. 그리고 if-else if-else if- ... -else는 chained multiplexer를 생성하고 이는 성능을 감소시킴이 알려져 있다. 그래서 if-else if-else가 얼마나 사용되는지 확인할 필요가 있는 것이다[2].

## References
1. node ninja, "How to split a two-digit number up in Verilog," 2011. [Online]. Available: https://stackoverflow.com/questions/5267331/how-to-split-a-two-digit-number-up-in-verilog, [Accessed Feb. 28, 2024].
2. "Design Compiler User Guide Version L-2016.03-SP2," Synopsys, 2016.
