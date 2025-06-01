---
title: Rounding Errors
weight: 2
published: true
---

하드웨어에서 부동소수점 수의 반올림 방식은 매우 단순합니다. 연산 결과를 정확히 표현할 수 없을 때에만 반올림이 발생하며, 기본적으로는 가장 가까운 표현 가능한 값으로 반올림됩니다. 값이 두 개의 후보 사이에 정확히 중간일 경우, 끝자리가 0인 값을 선호하는 방식으로 결정됩니다.

다음 코드를 살펴보겠습니다.

```c++
float x = 0;
for (int i = 0; i < (1 << 25); i++)
    x++;
printf("%f\n", x);
```

수학적으로 옳은 결과인 $2^{25} = 33554432$를 출력하는 것 대신, 이것은 $16777216 = 2^{24}$를 출력합니다. 왜일까요?

부동 소수점 숫자 $x$를 반복적으로 1씩 증가시키다 보면, 어느 순간 $(x + 1)$이 $x$로 반올림되어 더 이상 값이 변하지 않게 됩니다. 32비트 float에서는 그 첫 번째 숫자가 바로 $2^{24}%입니다. 왜냐하면 이 값은 가수 비트 수보다 1 큰 값이기 때문입니다.

$$2^{24} + 1 = 2^{24} \cdot 1.\underbrace{0\ldots0}_{\times 23} 1$$

이 값은 표현 가능한 두 부동소수점 값의 정확히 중간에 위치하며, IEEE 754의 반올림 규칙(짝수 쪽으로 반올림)에 따라 $2^{24}$로 반올림됩니다. 반면, $2^{24}$보다 작은 값에 1을 더하는 연산은 여전히 정확히 표현 가능하므로 반올림이 일어나지 않습니다.

### 반올림 오차 및 작동 순서

부동 소수점 계산은 수학적으로 옳더라도, 연산 순서에 따라 결과가 달라질 수 있습니다.

예를 들어, 덧셈과 곱셈은 순수한 수학적으로는 교환법칙과 결합법칙이 성립하지만, 부동 소수점 연산에서는 반올림 오차로 인해 그렇지 않습니다. 세 개의 부동 소수점 변수 $x$, $y$, $z$가 있을 때, $(x + y + z)$의 결과는 덧셈 순서에 따라 달라질 수 있습니다. 이러한 비교환성은 대부분의 부동 소수점 연산에도 마찬가지로 적용됩니다.

컴파일러는 [명세에 위배되는 결과](/hpc/compilation/contracts/)를 생성해서는 안되므로, 이러한 미묘한 차이로 인해 피연산자의 순서를 바꾸는 최적화가 제한됩니다. 하지만 GCC와 Clang에서는 `-ffast-math` 플래그를 통해 이러한 엄격한 규정을 비활성화할 수 있습니다. 이 플래그를 추가하고 앞서의 코드를 다시 컴파일하면 실행 속도가 [상당히 빨라질 뿐만 아니라](/hpc/simd/reduction), 올바른 결과인 33554432가 출력되기도 합니다.(다만 이 경우, 컴파일러가 더 낮은 정밀도의 연산 경로를 선택했을 가능성도 있다는 점은 유의해야 합니다.)

### 반올림 모드

(Banker의 반올림이라 알려진)기본 모드 외에도, 4가지 다른 종류의 반올림 모드를 [설정](https://www.cplusplus.com/reference/cfenv/fesetround/)할 수 있습니다.

- 가장 가까운 수로 반올림하되, 정확히 중간일 경우 0에서 멀어지는 방향으로 반올림
- 올림(양의 무한대로 반올림, 음수의 경우 0에 가까워짐)
- 내림(음의 무한대로 반올림, 음수의 경우 0에서 멀어짐)
- 0 방향으로 반올림(이진 결과를 자르는 방식, 절삭)

예를 들어, 위 반복문 실행 전에 `fesetround(FE_UPWARD)`를 호출하면, 결과는 $2^{24}$도, $2^{25}$도 아닌 $67108864 = 2^{26}$이 출력됩니다. 이는 $2^{24}$에 도달했을 때 $(x + 1)$이 더 이상 표현되지 않고, 가장 가까운 표현 가능한 수인 $(x + 2)$ 올림되기 시작하기 때문입니다. 그 결과, $2^{25}$에는 절반의 횟수만에 도달하게 되고, 그 이후로는 $(x + 1)$이 $(x + 4)$로 반올림되면서, 속도가 네 배로 빨라집니다.

이러한 대체 반올림 모드의 주요 활용 중 하나는 수치 불안정성을 진단하는 것입니다. 만약 양의 무한대 반올림과 음의 무한대 반올림 간에 알고리즘의 결과가 크게 달라진다면, 이는 해당 알고리즘이 반올림 오차에 민감하다는 신호입니다.

이런 방식의 테스트는 종종 모든 연산을 낮은 정밀도로 변경한 뒤 결과 변화량을 확인하는 것보다 더 효과적입니다. 그 이유는, 기본 반올림 방식(가장 가까운 값으로 반올림)은 충분히 평균을 취하면 결국 기댓값에 수렴하기 때문입니다. 반올림 오차의 절반은 위로, 절반은 아래로 발생하기 때문에, 통계적으로는 서로 상쇄되는 경향이 있습니다.


### 측정 오차

로그나 제곱근과 같은 복잡한 계산을 수행하는 하드웨어에서 이러한 보장을 기대하는 것은 놀라울 수 있습니다. 하지만 실제로는 모든 연산에 대해 가능한 한 최고의 정밀도가 보장됩니다. 덕분에 반올림 오차를 분석하기가 매우 쉬워지며, 곧 그 예를 보게될 것입니다.

계산 오차를 측정하는 대표적인 방법에는 두 가지가 있습니다.

* 하드웨어나 사양에 맞는 정확한 소프트웨어를 설계하는 엔지니어들은 보통 ULP(Units in the Last Place)에 주목합니다. 이는 실제 정확한 값과 연산 결과 사이에 존재할 수 있는 표현 가능한 수의 개수, 즉 두 수 사이의 거리입니다.
* 반면 수치 알고리즘을 다루는 사람들은 상대 정밀도에 관심을 가집니다. 이는 실제 값으로 근사 오차를 나눈 절댓값, 즉 $|\frac{v-v'}{v}|$입니다.

어느 경우든 오차 분석의 일반적인 전략은 최악의 경우를 가정하고 오차의 상한선을 정하는 것입니다.

기본적인 산술 연산을 한 번 수행했을 때 최악의 경우는, 결과가 가장 가까운 표현 가능한 수로 반올림되는 것이며, 이때의 오차는 최대 0.5ULP를 넘지 않습니다. 같은 방식으로 상대 오차를 다루기 위해 우리는 기계 엡실론이라 불리는 수 $\epsilon$을 정의할 수 있습니다. 이는 값 1과 그 다음 표현가능한 수 간의 차이로, 일반적인 가수에 할당된 비트 수에 따라 $2^{-\text{비트 수}}$로 결정됩니다.

즉, 단일 산술 연산의 결과로 $x$를 얻었다면, 실제 값은 다음 범위 안에 존재합니다.

$$
[x \cdot (1-\epsilon),\; x \cdot (1 + \epsilon)]
$$

이러한 오차가 항상 존재한다는 사실은 특히 부동 소수점 연산의 결과를 바탕으로 이진적인 '참/거짓' 결정을 내려야 할 때 반드시 염두에 두어야 합니다. 예를 들어, 두 값이 같은지를 확인하려면 다음과 같이 해야 합니다.

```c++
const float eps = std::numeric_limits<float>::epsilon; // ~2^(-23)
bool eq(float a, float b) {
    return abs(a - b) <= eps;
}
```

`eps`값은 사용하는 상황에 따라 달라져야 합니다. 위 예제에서 사용한 값은 `float`형에 대한 기계 엡실론이며, 단 하나의 부동 소수점 연산에서만 의미 있는 수준의 정밀도를 가집니다.

### 구간 산술

어떤 알고리즘이 계산 도중 발생한 오차가 크게 중폭되지 않는다면, 이를 수치적으로 안정적(numerically stable)이라 합니다. 이러한 안정성은 문제 자체가 좋은 조건(well conditioned)일 때에만 보장됩니다. 즉, 입력값이 조금 바뀌었을 때 결과도 그에 비례하여 조금만 변하는 경우입니다.

수치 알고리즘을 분석할 때는 실험 물리학에서 사용하는 방식과 같은 접근을 취하는 것이 유용합니다. 즉, 정확한 실수 값을 다루는 대신, 해당 값이 존재할 수 있는 구간을 기준으로 작업하는 것입니다.

예를 들어, 어떤 변수에 임의의 실수를 연속적으로 곱하는 다음과 같은 연산을 생각해 봅시다.

```cpp
float x = 1;
for (int i = 0; i < n; i++)
    x *= a[i];
```

첫 번째 곱셈 이후, 계산된 값 $x$는 실제 곱의 값에 대해 $(1 + \epsilon)$의 오차범위 내에 있게 됩니다. 이후 매번 곱셈이 수행될 때마다 이 경계는 또 다른 $(1 + \epsilon)$이 곱해집니다. 귀납적으로 보면, $n$번의 곱셈이 끝난후 계산된 값은 $(1 + \epsilon)^n = 1 + n \epsilon + O(\epsilon^2)$의 범위 내에 있게 됩니다. 하한 또한 유사하게 정의됩니다.

이로부터 상대 오차가 $O(n \epsilon)$임을 알 수 있으며, 일반적으로 $n \ll \frac{1}{\epsilon}$이므로 이는 크게 문제가 되지 않습니다.

수치적으로 불안정한 계산의 예로 다음과 같은 함수를 생각해 봅시다.

$$
f(x, y) = x^2 - y^2
$$

$x > y$라 가정하면, 이 함수가 반환할 수 있는 최대 값은 대략 다음과 같습니다.

$$
x^2 \cdot (1 + \epsilon) - y^2 \cdot (1 - \epsilon)
$$

이 때 발생하는 절대 오차는 다음과 같습니다.

$$
x^2 \cdot (1 + \epsilon) - y^2 \cdot (1 - \epsilon) - (x^2 - y^2) = (x^2 + y^2) \cdot \epsilon
$$

따라서 상대 오차는 다음과 같습니다.

$$
\frac{x^2 + y^2}{x^2 - y^2} \cdot \epsilon
$$

$x$와 $y$가 가깝다면 오차는 $O(\epsilon \cdot |x|)$ 수준으로 커질 수 있습니다.

이와 같은 직접 계산 방식에서는 제곱된 값의 오차가 뺄셈 과정에서 증폭됩니다. 하지만 다음과 같은 식으로 변형하면 이를 완화할 수 있습니다.

$$
f(x, y) = x^2 - y^2 = (x + y) \cdot (x - y)
$$

이 방식에서는 오차가 $\epsilon \cdot |x - y|$이하로 제한된다는 것을 쉽게 증명할 수 있습니다. 또한 이 방법은 연산 속도 측면에서도 유리한데, 두 번의 덧셈과 한 번의 곱셈만 수행하므로, 원래 방식보다 느린 곱셈이 하나 줄고 빠른 덧셈이 하나 늘어납니다.

### Kahan 가법

이전 예제에서 알 수 있듯이, 긴 연산의 연쇄 자체는 큰 문제가 되지 않습니다. 하지만 서로 크기가 다른 수들 간의 덧셈과 뺄셈에서는 문제가 발생합니다. 이러한 문제를 다룰 때의 일반적인 접근법은 큰 수는 큰 수끼리, 작은 수는 작은 수끼리 연산하도록 구성하는 것입니다.

표준적인 덧셈 알고리즘을 살펴봅시다.

```c++
float s = 0;
for (int i = 0; i < n; i++)
    s += a[i];
```

이 경우는 곱셈이 아닌 덧셈을 수행하고 있기 때문에, 상대 오차가 단순히 $O(\epsilon \cdot n)$으로 제한되지 않으며, 입력값의 구성에 크게 의존하게 됩니다.

가장 극단적인 예로, 첫 번째 값이 $2^{24}$이고 나머지 값들이 모두 $1$인 경우를 생각해 봅시다. 이때 덧셈의 결과는 $n$이 얼마이든 간에 계속해서 $2^{24}$로 고정됩니다. 이는 다음 코드를 실행하면 쉽게 확인할 수 있습니다. 아래 코드에서는 두 번 모두 $16777216 = 2^{24}$가 출력됩니다.

```cpp
const int n = (1<<24);
printf("%d\n", n);

float s = n;
for (int i = 0; i < n; i++)
    s += 1.0;

printf("%f\n", s);
```

이 현상은 `float` 타입이 23비트의 가수부만을 가지기 때문에 발생합니다. 즉, $2^{24} + 1$은 부동 소수점으로 정확하게 표현할 수 없는 최초의 정수입니다. 따라서 $s = 2^{24}$에 $1$을 더하려 할때마다 반올림에 의해 무시되어 더해지지 않게 됩니다. 이 경우 오차는 실제로 $O(n \cdot \epsilon)$ 수준이지만, 이는 상대 오차가 아닌 절대 오차의 관점입니다. 위 예제에서는 오차가 2이고, 만약 마지막 값이 $-2^{24}$였다면 그 오차는 무한히 커질 수도 있습니다.

가장 단순한 해결책은 `double`과 같이 더 큰 정밀도의 타입으로 바꾸는 것입니다. 하지만 이는 현실적으로 확장성이 떨어지는 방법입니다. 보다 우아한 해법은, 더해지지 못한 소수점 아래 값들을 따로 저장하고 이를 다음 값에 보정하여 더하는 방식입니다.

```c++
float s = 0, c = 0;
for (int i = 0; i < n; i++) {
    float y = a[i] - c; // c is zero on the first iteration
    float t = s + y;    // s may be big and y may be small, losing low-order bits of y
    c = (t - s) - y;    // (t - s) cancels high-order part of y
    s = t;
}
```

이 기법은 Kahan 가법이라 알려져 있습니다. 이 방법을 사용하면 상대 오차는 $2 \epsilon + O(n \epsilon^2)$이내로 제한됩니다. 여기서 첫 항은 마지막 덧셈 연산에서 오고, 두 번째 항은 각 단계에서 $\epsilon$보다 작은 수준의 오차만 발생하기 때문에 추가로 누적되는 오차를 반영한 것입니다.

물론 배열의 합뿐 아니라 보다 일반적인 상황에서도 사용할 수 있는 방법은, 정밀도가 더 높은 타입인 `double`로 바꾸는 것입니다. 이렇게 하면 기계 엡실론도 사실상 제곱 수준으로 감소하게 됩니다. 더 나아가, `double` 두 개를 묶어서 하나는 실제 값, 다른 하나는 표현할 수 없는 오차의 누적값을 저장하게 하면, 전체적으로 $a + b$형태로 보다 높은 정밀도를 표현할 수 있습니다. 이 방법은 Double-Double 산술법이라 불리며, 이를 확장하여 Quad-Double 또는 더 높은 정밀도까지 일반화할 수 있습니다.

<!--

## Conversion to Decimal

It is unfortunate that humans evolved to have 10 fingers, because owing to this fact we ended up with a very clumsy number system.

Digit is actually also a anatomical term meaning either a finger or a toe

Six fingers on each hand would be more convenient, because it would be straightforward to divide numbers by 2, 3, 4 and 6. This numbering system was used by ancient Babylonians, and it is still the reason why we have 60 seconds in a minute, 24 hours in a day, 12 months and 6 cans of beer in a pack: you can perfectly divide items by low divisors.

Four fingers on each hand (like in The Simpsons) would give us a very convenient octal system where you can divide by powers of 2, although most people would not start appreciating it until invention of computers.

But here we are, and we have a problem of converting binary floating-point numbers to decimal numbers in scientific notation. But what does that even mean, exactly?

Note that some decimal numbers are not representable in finite form in binary. Here is a famous JavaScript joke (that you can reproduce by pressing F12 in your browser):

```
> 0.1
< 0.1
> 0.2
< 0.2
> 0.1+0.2
< 0.30000000000000004
```

Neither of them are exact, so JavaScript prints the shortest number that would be parsed back as the same number (reading numbers is defined similarly: it rounds to the closest representable number). The result of "0.3" and "0.1+0.2" is off by exactly one ULP, so it isn't printed as 0.3.

The way to approach any hard problem is to figure out how to solve some partial cases in then to figure out how to reduce the initial problem. Let's start with processing the sign bit: it's simple, just print "-" in front of a number in case it is 1. Next, we can check for special values.

Then, we can notice that some numbers are easy to print. If we have 23-bit mantissa and our exponent value is exactly 23, then we can reinterpret the mantissa as integer, add it to $2^23$ (the implicit 1) and then print it as we would print an integer. We can also do the same thing for small exponents, except that we would need to multiply that intermediate integer by a small power of two.

But what to do in general case, if the exponent value is either too large or too small? We can reduce the problem to the previous case by multiplying it by $\frac{10^a}{2^b}$ for some integers $a$ and $b$ with precise enough arithmetic so that the exponent is small.

Multiplying or dividing by 10 is the same as incrementing the exponent (the resulting one after the "e" in scientific notation, not the binary). The idea is to find a proper power of 10 so that the resulting number will have . We need to precalculate numbers of the form $\frac{10^a}{2^b}$ (since exponent is limited, there won't be many of them). To get the precalculated number, we need to look at the exponent (or possibly its neighbors).

The tricky part is the "shortest possible." It can be solved by printing digits one by one and trying to parse it back, but this would be too slow.

How many decimal digits do we need to print a `float`?

-->
