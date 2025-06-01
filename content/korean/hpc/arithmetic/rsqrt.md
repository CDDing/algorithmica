---
title: Fast Inverse Square Root
weight: 4
---

부동 소수점 숫자의 역 제곱근 $\frac{1}{\sqrt x}$은 정규화된 벡터를 계산하는 데 사용됩니다. 이러한 정규화된 벡터는 조명에서의 입사각과 반사각을 결정하는 등의 컴퓨터 그래픽 시뮬레이션을 포함해 다양한 시뮬레이션 시나리오에서 광범위하게 활용됩니다.

$$
\hat{v} = \frac{\vec v}{\sqrt {v_x^2 + v_y^2 + v_z^2}}
$$

역 제곱근을 직접 계산하는 방식, 즉, 먼저 제곱근을 구하고 그 값으로 $1$을 나누는 방식은 매우 느립니다. 두 연산 모두 하드웨어 수준에서 구현되어 있음에도 불구하고 연산 비용이 크기 때문입니다.

하지만 부동 소수점 숫자가 메모리에 저장되는 방식을 활용한 놀라울 정도로 효율적인 근사 알고리즘이 존재합니다. 이 알고리즘은 매우 뛰어나서 결국 [하드웨어 수준](https://www.felixcloutier.com/x86/rsqrtps)에서 구현되었으며, 오늘날 소프트웨어 엔지니어에게는 더 이상 직접 사용될 필요가 없을지도 모릅니다. 그럼에도 불구하고 이 알고리즘은 그 자체로 아름답고 교육적인 가치가 높기 때문에, 그 내부 동작을 함께 살펴볼 것입니다.

이 기법 자체도 흥미롭지만, 그 기원이 특히 흥미롭습니다. 이 알고리즘은 id Software라는 게임 스튜디오에서 사용되었으며, 그들의 전설적인 1999년 게임 Quake III Arena에 적용되었습니다. 다만, 이 알고리즘은 "누군가 배운 사람에게 또 배웠다"는 식의 전해 내려온 지식으로 알려져 있으며, 그 끝에는 IEEE 754 표준과 Kahan summation algorithm을 만든 William Kahan이 있다는 설이 있습니다.

이 알고리즘은 2005년 Quake III Arena의 소스 코드가 공개되면서 게임 개발 커뮤니티에서 널리 알려졌습니다. 다음은 해당 소스 코드를 [발췌](https://github.com/id-Software/Quake-III-Arena/blob/master/code/game/q_math.c#L552)한 부분이며, 주석도 함께 포함되어 있습니다.

```c++
float Q_rsqrt(float number) {
    long i;
    float x2, y;
    const float threehalfs = 1.5F;

    x2 = number * 0.5F;
    y  = number;
    i  = * ( long * ) &y;                       // evil floating point bit level hacking
    i  = 0x5f3759df - ( i >> 1 );               // what the fuck? 
    y  = * ( float * ) &i;
    y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//  y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed

    return y;
}
```

이 함수가 어떤 방식으로 동작하는지는 차근차근 살펴볼 예정이지만, 먼저 간단한 우회 설명이 필요합니다.

### 근사 로그

컴퓨터나 계산기가 보편화되기 이전에는, 곱셈과 이와 관련된 연산을 로그 표를 이용해 계산하곤 했습니다. a와 b의 로그 값을 표에서 찾아 더한 뒤, 그 결과의 역로그를 다시 표에서 찾아 곱셈 결과를 얻는 식이었습니다.

$$
a \times b = 10^{\log a + \log b} = \log^{-1}(\log a + \log b)
$$

역 제곱근 $\frac{1}{\sqrt x}$을 계산할 때에도 이와 같은 방식의 트릭을 사용할 수 있습니다.

$$
\log \frac{1}{\sqrt x} = - \frac{1}{2} \log x
$$

빠른 역 제곱근 알고리즘은 이 식에 기반하고 있으며, 따라서 $x$의 로그 값을 매우 빠르게 계산해야 합니다. 그런데 놀랍게도, 32비트 `float` 값을 정수로 단순히 변환하는 것만으로도 로그 값을 근사할 수 있습니다.

부동 소수점 수는 부호 비트, 지수 $e_x$, 가수 $m_x$ 순서로 저장된다는 점을 [상기](../float)해봅시다. 이는 다음과 같이 표현됩니다.

$$
x = 2^{e_x} \cdot (1 + m_x)
$$

이 값의 로그를 취하면 다음과 같습니다.

$$
\log_2 x = e_x + \log_2 (1 + m_x)
$$

$m_x \in [0, 1)$ 이므로, 오른쪽 항은 다음과 같이 근사할 수 있습니다.

$$
\log_2 (1 + m_x) \approx m_x
$$

이 근사식은 구간의 양 끝에서는 정확하지만, 평균적인 경우를 고려해 약간의 상수 $\sigma$를 더하는 편이 좋습니다. 따라서 다음과 같습니다.

$$
\log_2 x = e_x + \log_2 (1 + m_x) \approx e_x + m_x + \sigma
$$

이제 이 근사식을 염두에 두고, $L = 2^{23}$(`float`의 가수 비트 수), $B = 127$(지수 바이어스)라고 정의하면, $x$의 비트 패턴을 정수 $I_x$로 재해석할 때 다음과 같은 식이 성립합니다.

$$
\begin{aligned}
I_x &= L \cdot (e_x + B + m_x)
\\  &= L \cdot (e_x + m_x + \sigma +B-\sigma )
\\  &\approx L \cdot \log_2 (x) + L \cdot (B-\sigma )
\end{aligned}
$$

정수에 $L=2^{23}$을 곱하는 것은, 23비트를 왼쪽으로 시프트하는 것과 동일합니다.

$\sigma$ 값을 평균 제곱 오차가 최소화되도록 조정하면, 결과는 놀라운 정도로 정확한 근삿값이 됩니다.

![Reinterpreting a floating-point number $x$ as an integer (blue) compared to its scaled and shifted logarithm (gray)](../img/approx.svg)

이제 이 근사식을 이용해 로그 값을 다시 표현하면 다음과 같습니다.

$$
\log_2 x \approx \frac{I_x}{L} - (B - \sigma)
$$

좋습니다. 이제 어디까지 왔죠? 아, 맞다. 우리가 하려던 건 역 제곱근을 계산하는 것이었죠.

### 결과 근사하기

식 $\log_2 y = - \frac{1}{2} \log_2 x$을 이용해 $y = \frac{1}{\sqrt x}$을 계산하려면, 이를 앞서 만든 근사식에 대입하면 됩니다.

$$
\frac{I_y}{L} - (B - \sigma)
\approx
- \frac{1}{2} ( \frac{I_x}{L} - (B - \sigma) )
$$

$I_y$에 대해 정리하면 다음과 같습니다.

$$
I_y \approx \frac{3}{2} L (B - \sigma) - \frac{1}{2} I_x
$$

결국 로그를 직접 계산할 필요조차 없다는 것을 알 수 있습니다. 위 식은 $x$를 정수로 간주했을 때의 값의 절반을 어떤 상수에서 뺀 형태일 뿐입니다. 실제 코드에서는 다음과 같이 작성됩니다.

```cpp
i = * ( long * ) &y;
i = 0x5f3759df - ( i >> 1 );
```

첫 줄에서는 `y`의 비트 배턴을 정수로 재해석하고, 두 번째 줄에서의 위의 식에 대입합니다. 첫 번째 항은 매직 넘버 $\frac{3}{2} L (B - \sigma) = \mathtt{0x5F3759DF}$이고, 두 번째 항은 나눗셈 대신 비트 시프트 연산으로 계산됩니다.

### Newton 기법 반복하기

다음으로는 $f(y) = \frac{1}{y^2} - x$에 대해 Newton 기법을 직접 두 번 정도 반복하는 코드입니다. 초기값이 매우 우수하므로 빠르게 수렴합니다.

$$
f'(y) = - \frac{2}{y^3} \implies y_{i+1} = y_{i} (\frac{3}{2} - \frac{x}{2} y_i^2) = \frac{y_i (3 - x y_i^2)}{2}
$$

코드로는 다음과 같이 표현됩니다.

```cpp
x2 = number * 0.5F;
y  = y * ( threehalfs - ( x2 * y * y ) );
```

초기 근사치가 매우 뛰어나기 때문에 게임 개발에서는 한 번의 반복으로도 충분했습니다. 첫 반복만으로도 정답의 99.8% 수준의 정확도를 달성할 수 있으며, 더 높은 정확도를 원한다면 반복 횟수를 늘리면 됩니다. 실제 하드웨어 [x86 명령어](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=3037,3009,5135,4870,4870,4872,4875,833,879,874,849,848,6715,4845,6046,3853,288,6570,6527,6527,90,7307,6385,5993&text=rsqrt&techs=AVX,AVX2)는 이러한 반복을 몇 차례 수행하며, 상대 오차가 $1.5 \times 2^{-12}$를 넘지 않도록 보장합니다.

### 읽을거리

[Wikipedia article on fast inverse square root](https://en.wikipedia.org/wiki/Fast_inverse_square_root#Floating-point_representation).
