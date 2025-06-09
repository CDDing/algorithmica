---
title: Montgomery Multiplication
weight: 4
published: true
---

놀랍지는 않은 사실로, [모듈러 연산](../modular)중 상당 부분은 나머지 연산에 소모됩니다. 이 연산은 [일반적인 정수 나눗셈](/hpc/arithmetic/division/)만큼 느리며, 피연산자의 크기에 따라 다르지만 일반적으로 15~20 사이클이 걸립니다.

이러한 성능 저하를 피하는 가장 좋은 방법은 나머지 연산을 아예 피하거나, [분기 없는 조건 처리(predication)](/hpc/pipelining/branchless)로 대체하거나 지연시키는 것입니다. 예를 들어, 모듈러 합을 계산할 때 다음과 같은 방식으로 구현할 수 있습니다.

```cpp
const int M = 1e9 + 7;

// input: array of n integers in the [0, M) range
// output: sum modulo M
int slow_sum(int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s = (s + a[i]) % M;
    return s;
}

int fast_sum(int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++) {
        s += a[i]; // s < 2 * M
        s = (s >= M ? s - M : s); // will be replaced with cmov
    }
    return s;
}

int faster_sum(int *a, int n) {
    long long s = 0; // 64-bit integer to handle overflow
    for (int i = 0; i < n; i++)
        s += a[i]; // will be vectorized
    return s % M;
}
```

하지만 때로는 모듈러 곱셈만 반복적으로 수행되는 경우가 있으며, 이런 경우에는 [정수 나눗셈 트릭](../hpc/arithmetic/division/)들을 제외하고는 나눗셈의 나머지를 계산하는 과정을 피할 좋은 방법이 없습니다. 이 트릭들은 일정한 상수 모듈러 값과 몇 가지 사전 계산을 필요로 합니다.

하지만 Montgomery 곱셈이라는 모듈러 연산을 위해 특별히 설계된 또다른 기법이 존재합니다.

### Montgomery 공간

Montgomery 곱셈은 곱셈에 사용되는 수들을 먼저 Montgomery 공간으로 변형한 뒤, 그 공간 내에서 모듈러 곱셈을 빠르게 수행하고, 최종적으로 필요한 시점에 실제 값으로 다시 변환하는 방식입니다. 일반적인 정수 나눗셈 기법들과는 달리, Montgomery 곱셈은 단 한번의 모듈러 연산만 수행할 때에는 비효율적이며, 여러 번의 모듈러 연산이 이어질 때에만 사용할 가치가 있습니다.

이 Montgomery 공간은 모듈러 값 $n$과양의 정수 $r \ge n$으로 정의되며, 이때 $r$은 $n$과 서로소여야만 합니다. 알고리즘은 $r$에 대한 나눗셈과 모듈러 연산을 포함하므로, 실제로는 $r$을 $2^{32}$또는 $2^{64}$같은 값으로 설정하여 이러한 연산을 각각 비트 오른쪽 시프트 및 비트 AND 연산으로 대체할 수 있게 합니다.

<!-- Therefore $n$ needs to be an odd number so that every power of $2$ will be coprime to $n$. And if it is not, we can make it odd (?). -->

어떤 수 $x$의 Montgomery 공간에서의 표현 $\bar{x}$는 다음과 같이 정의됩니다.

$$
\bar{x} = x \cdot r \bmod n
$$

이 변환은 곱셈과 모듈러 연산을 필요로 하며, 이는 우리가 처음에 피하고자 했던 비용이 큰 연산입니다. 따라서 Montgomery 곱셈은 이러한 변환 및 복원 과정의 오버헤드를 감수할 가치가 있을 때, 즉 같은 모듈러 연산이 여러 번 필요한 상황에서만 유용합니다.

<!-- Note that the transformation is actually such a multiplication that we want to optimize, so it is still an expensive operation. However, we will only need to transform a number into the space once, perform as many operations as we want efficiently in that space and at the end transform the final result back, which should be profitable if we are doing lots of operations modulo $n$. -->

Montgomery 공간 내에서는 덧셈, 뺄셈, 비교는 일반적인 방식으로 수행할 수 있습니다.

$$
x \cdot r + y \cdot r \equiv (x + y) \cdot r \bmod n
$$

하지만 곱셈은 다릅니다. Montgomery 공간에서의 곱셈을 $*$로, 일반적인 곱셈을 $\cdot$로 표기하면, 결과는 다음과 같습니다.

$$
\bar{x} * \bar{y} = \overline{x \cdot y} = (x \cdot y) \cdot r \bmod n
$$

하지만 실제로 Montgomery 표현끼리 곱하면 다음과 같은 결과가 나옵니다.

$$
\bar{x} \cdot \bar{y} = (x \cdot y) \cdot r \cdot r \bmod n
$$

따라서, Montgomery 공간에서의 곱셈은 다음과 같이 정의되어야 합니다.

$$
\bar{x} * \bar{y} = \bar{x} \cdot \bar{y} \cdot r^{-1} \bmod n
$$

즉, Montgomery 공간에서 두 수를 곱한 후에는 결과에 $r^{-1}$을 곱하고 $n$으로 모듈러를 취하는 감산 과정이 필요하며, 이 연산을 효율적으로 수행할 수 있는 알고리즘이 존재합니다.

### Montgomery 감법

$r=2^{32}$이고 모듈러 $n$은 32비트, 그리고 감소시켜야 할 숫자 $x$가 64비트라고 가정합시다(두 32비트의 곱이라 가정). 우리의 목표는 $y = x \cdot r^{-1} \bmod n$을 계산하는 것입니다.

$r$이 $n$과 서로소이기 때문에, $[0, n)$ 범위에 다음과 같이 두 수 $r^{-1}$과 $n^\prime$가 존재하여 다음을 만족합니다.

$$
r \cdot r^{-1} + n \cdot n^\prime = 1
$$

이러한 값들은 [확장 유클리드 알고리즘](../euclid-extended)을 사용해 계산할 수 있습니다.

이 식을 통해 $r \cdot r^{-1}$을 $(1 - n \cdot n^\prime)$로 계산하고 $x \cdot r^{-1}$를 다음과 같이 정리할 수 있습니다.

$$
\begin{aligned}
x \cdot r^{-1} &= x \cdot r \cdot r^{-1} / r
\\             &= x \cdot (1 - n \cdot n^{\prime}) / r
\\             &= (x - x \cdot n \cdot n^{\prime}    ) / r
\\             &\equiv (x - x \cdot n \cdot n^{\prime} + k \cdot r \cdot n) / r &\pmod n &\;\;\text{(for any integer $k$)}
\\             &\equiv (x - (x \cdot n^{\prime} - k \cdot r) \cdot n) / r &\pmod n
\end{aligned}
$$

이제 $\lfloor x \cdot n^\prime / r \rfloor$로 선택하면, $(k \cdot r - x \cdot n^{\prime})$는 $x \cdot n^{\prime} \bmod r$와 같아지고, 다음이 성립합니다.

$$
x \cdot r^{-1} \equiv (x - x \cdot n^{\prime} \bmod r \cdot n) / r
$$

알고리즘은 이 공식을 직접 계산하며 두 번의 곱셈을 통해 $q = x \cdot n^{\prime} \bmod r$와 $m = q \cdot n$를 구한 뒤, $x-m$을 계산하여 $r$로 나눈 다음, 결과가 음수인지 확인합니다.

$$
x < n \cdot n < r \cdot n \implies x / r < n
$$


$$
m = q \cdot n < r \cdot n \implies m / r < n
$$

따라서 다음 범위가 보장됩니다.

$$
-n < (x - m) / r < n
$$

결과가 음수일 경우 $n$을 더해 $[0,n)$ 범위로 조정합니다.

```c++
typedef __uint32_t u32;
typedef __uint64_t u64;

const u32 n = 1e9 + 7, nr = inverse(n, 1ull << 32);

u32 reduce(u64 x) {
    u32 q = u32(x) * nr;      // q = x * n' mod r
    u64 m = (u64) q * n;      // m = q * n
    u32 y = (x - m) >> 32;    // y = (x - m) / r
    return x < m ? y + n : y; // if y < 0, add n to make it be in the [0, n) range
}
```

결과가 반드시 $[0,n)$에 있을 필요가 없다면 조건 분기를 제거하고 항상 $n$을 더할 수 있습니다.

```c++
u32 reduce(u64 x) {
    u32 q = u32(x) * nr;
    u64 m = (u64) q * n;
    u32 y = (x - m) >> 32;
    return y + n
}
```

`>> 32` 연산을 계산 그래프에서 한 단계 앞당기고, $(x-m)/r$ 대신 $\lfloor x / r \rfloor - \lfloor m / r \rfloor$을 계산할 수 있습니다. 이는 $x$와 $m$의 하위 32비트가 항상 같기 때문에 올바른 결과를 제공합니다.

$$
m = x \cdot n^\prime \cdot n \equiv x \pmod r
$$

하지만 왜 단순히 오른쪽 쉬프트 연산을 한 번만 수행할 수 있는데도, 두 번 수행하는 방식을 선택할까요? 이는 성능상의 이점이 있기 때문입니다.`((u64) q * n) >> 32`에서는 32비트 정수끼리의 곱셈 결과에서 상위 32비트를 취하는 연산이 필요한데, 이는 x86의 `mul` 명령어가 이미 별도의 레지스터에 상위 비트를 저장해주기 때문에 별도의 비용이 들지 않습니다. 또한, `x >> 32` 연산은 성능의 병목 지점에 포함되지 않기 때문에 부담이 되지 않습니다.

```c++
u32 reduce(u64 x) {
    u32 q = u32(x) * nr;
    u32 m = ((u64) q * n) >> 32;
    return (x >> 32) + n - m;
}
```

Montgomery 곱셈이 다른 모듈러 감소 방식보다 가지는 주요 장점 중 하나는 매우 큰 정수형 타입을 요구하지 않는다는 점입니다. 오직 $r \times r$ 크기의 곱셈과, 그 결과의 상위 및 하위 $r$ 비트를 추출하는 연산만 필요합니다. 이러한 연산은 대부분의 하드웨어에서 [특수 지원](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=7395,7392,7269,4868,7269,7269,1820,1835,6385,5051,4909,4918,5051,7269,6423,7410,150,2138,1829,1944,3009,1029,7077,519,5183,4462,4490,1944,5055,5012,5055&techs=AVX,AVX2&text=mul)되며, [SIMD](../hpc/simd/)나 더 큰 데이터 타입으로도 쉽게 일반화할 수 있다는 장점이 있습니다.

```c++
typedef __uint128_t u128;

u64 reduce(u128 x) const {
    u64 q = u64(x) * nr;
    u64 m = ((u128) q * n) >> 64;
    return (x >> 64) + n - m;
}
```

일반적인 정수 나눗셈 기법으로는 128비트의 수를 64비트로 나누는 연산이 효율적으로 불가능하므로, [컴파일러](https://godbolt.org/z/fbEE4v4qr)는 [느린 긴 산술 라이브러리 함수](https://github.com/llvm-mirror/compiler-rt/blob/69445f095c22aac2388f939bedebf224a6efcdaf/lib/builtins/udivmodti4.c#L22)를 호출하게 됩니다.

### 역 제곱근과 변환

Montgomery 곱셈 자체는 빠르지만, 몇 가지 사전 계산이 필요합니다.

- $n^\prime$을 계산하기 위해 $n$ 모듈러의 역원을 구해야 합니다.
- 숫자를 Montgomery 공간으로 변환해야 합니다.
- 숫자를 Montgomery 공간에서 원래 공간으로 변환해야 합니다.

이 중 마지막 연산은 이미 우리가 구현한 `reduce` 함수를 통해 이미 효율적으로 처리되고 있지만, 앞의 두 과정은 조금 더 최적화할 수 있습니다.

역원 계산. $n^\prime = n^{-1} \bmod r$는 $r$이 2의 거듭제곱이라는 사실을 이용하면 확장 유클리드 알고리즘보다 더 빠르게 구할 수 있습니다. 아래의 항등식을 활용합니다.

$$
a \cdot x \equiv 1 \bmod 2^k
\implies
a \cdot x \cdot (2 - a \cdot x)
\equiv
1 \bmod 2^{2k}
$$

증명은 다음과 같습니다.

$$
\begin{aligned}
a \cdot x \cdot (2 - a \cdot x)
   &= 2 \cdot a \cdot x - (a \cdot x)^2
\\ &= 2 \cdot (1 + m \cdot 2^k) - (1 + m \cdot 2^k)^2
\\ &= 2 + 2 \cdot m \cdot 2^k - 1 - 2 \cdot m \cdot 2^k - m^2 \cdot 2^{2k}
\\ &= 1 - m^2 \cdot 2^{2k}
\\ &\equiv 1 \bmod 2^{2k}.
\end{aligned}
$$

처음에는 $a$의 $2^1$에 대한 모듈러 역원 $x = 1$로 시작하고, 이 항등식을 $\log_2 r$번 반복 적용함으로써 매번 정확히 두 배의 비트를 얻을 수 있습니다. 이 방식은 [Newton 기법](../hpc/arithmetic/newton/)과 유사한 점이 있습니다.

수를 Montgomery 공간으로 변환하는 것은 해당 숫자에 $r$을 곱하고 모듈러 연산을 수행하는 것입니다([기존 방식](../hpc/arithmetic/division/) 참고). 하지만 다음과 같은 관계를 이용하면 최적화를 도모할 수 있습니다.

$$
\bar{x} = x \cdot r \bmod n = x * r^2
$$

숫자를 Montgomery 공간으로 변환하는 것은 $r^2$를 곱하는 것과 동일합니다. 따라서 $r^2 \bmod n$을 사전에 계산해 두면, 단순한 곱셈과 감소 연산으로 변환을 수행할 수 있습니다. 다만, 이것이 실제로 빠를지는 상황에 따라 달라질 수 있습니다. 왜냐하면 $r = 2^k$를 곱하는 것은 단순한 좌측 쉬프트로 구현할 수 있지만, $r^2 \bmod n$을 곱하는 연산은 시프트로 대체할 수 없기 때문입니다.

### 완전한 구현

전부 `constexpr`로 감싸는 편이 편리합니다.

```c++
struct Montgomery {
    u32 n, nr;
    
    constexpr Montgomery(u32 n) : n(n), nr(1) {
        // log(2^32) = 5
        for (int i = 0; i < 5; i++)
            nr *= 2 - n * nr;
    }

    u32 reduce(u64 x) const {
        u32 q = u32(x) * nr;
        u32 m = ((u64) q * n) >> 32;
        return (x >> 32) + n - m;
        // returns a number in the [0, 2 * n - 2] range
        // (add a "x < n ? x : x - n" type of check if you need a proper modulo)
    }

    u32 multiply(u32 x, u32 y) const {
        return reduce((u64) x * y);
    }

    u32 transform(u32 x) const {
        return (u64(x) << 32) % n;
        // can also be implemented as multiply(x, r^2 mod n)
    }
};
```

성능을 확인해보기 위해, Montgomery 곱셈을 [binary exponentiation](../hpc/number-theory/exponentiation/)에 적용해봅시다.

```c++
constexpr Montgomery space(M);

int inverse(int _a) {
    u64 a = space.transform(_a);
    u64 r = space.transform(1);
    
    #pragma GCC unroll(30)
    for (int l = 0; l < 30; l++) {
        if ( (M - 2) >> l & 1 )
            r = space.multiply(r, a);
        a = space.multiply(a, a);
    }

    return space.reduce(r);
}
```

컴파일러가 자동으로 생성한 빠른 모듈러 연산을 사용하는 일반적인 이진 거듭제곱법은 `inverse` 호출당 약 170ns가 소요됩니다. 반면, 위와 같은 Montgomery 기반 구현은 약 166ns, `transform`과 `reduce`를 생략할 경우 약 158ns까지 줄어듭니다. (`inverse` 함수가 더 큰 모듈러 연산의 하위 절차로 사용되는 경우 생략이 가능합니다.) 비록 이 차이는 작지만, Montgomery 곱셈은 SIMD 애플리케이션이나 더 큰 정수 타입에서 훨씬 더 큰 성능 향상을 제공합니다.

연습문제로 효율적인 [모듈러 행렬곱](/hpc/algorithms/matmul)을 구현해보세요.
