---
title: Extended Euclidean Algorithm
weight: 3
---

[Fermat 정리](../modular/#fermats-theorem)는 모듈러 곱셈 역원을 [이진 거듭제곱](..exponentiation/)을 통해 $O(\log n)$의 연산으로 계산할 수 있게 해주지만, 이는 모듈러가 소수일 때만 적용됩니다. 이를 일반화한 정리가 [Euler 정리](https://en.wikipedia.org/wiki/Euler%27s_theorem)이며, $m$과 $a$가 서로소일 경우 다음과 같이 표현됩니다.

$$
a^{\phi(m)} \equiv 1 \pmod m
$$

여기서 $\phi(m)$은 [Euler의 토션(totient) 함수](https://en.wikipedia.org/wiki/Euler%27s_totient_function)로, $m$보다 작은 양의 정수 중 $m$과 서로소인 수의 개수를 의미합니다. 특별히 $m$이 소수인 경우, $1$부터 $m-1$까지의 모든 수가 $m$과 서로소이므로 $\phi(m) = m-1$이 되고 이는 Fermat 정리로 귀결됩니다.

이 정리를 이용하면, $\phi(m)$을 알고 있을 때 $a$의 모듈러 역원을 $a^{\phi(m) - 1}$로 계산할 수 있습니다. 하지만 일반적으로 $\phi(m)$을 구하기 위해서는 $m$의 [소인수분해](/hpc/algorithms/factorization/)가 필요하므로, 계산 속도가 느릴 수 있습니다. 이보다 더 일반적으로 사용할 수 있는 방법은 [유클리드 알고리즘](/hpc/algorithms/gcd/)을 변형하여 역원을 구하는 방식입니다.

### 알고리즘

확장 유클리드 알고리즘은 $g = \gcd(a,b)$를 구할 뿐만 아니라, 다음 조건을 만족하는 정수 $x$와 $y$도 찾습니다.


$$
a \cdot x + b \cdot y = g
$$

이 식은 $b$를 $m$으로, $g$를 $1$로 바꾸면 모듈러 역원을 구하는 문제로 바뀝니다.


$$
a^{-1} \cdot a + k \cdot m = 1
$$

따라서, $a$의 모듈러 역원은 $x$가 됩니다. 단, $a$와 $m$이 서로소가 아닌 경우에는 어떤 정수 $x$, $y$의 조합으로도 $1$을 만들 수 없으므로 해가 존재하지 않습니다. 이는 $a$와 $m$의 모든 정수 조합이 $\gcd(a,m)$의 배수이기 때문입니다.

이 알고리즘은 재귀적으로 동작합니다. 즉, $\gcd(b, a \bmod b)$에 대한 계수 $x'$과 $y'$을 먼저 계산한 뒤, 이를 바탕으로 원래의 입력 $(a,b)$에 대한 해를 복원합니다. 만약 $(b,a \bmod b)$에 대해 다음과 같은 해 $(x', y')$이 존재한다면

$$
b \cdot x' + (a \bmod b) \cdot y' = g
$$

이를 위 식에 대입하면

$$
b \cdot x' + (a - \Big \lfloor \frac{a}{b} \Big \rfloor \cdot b) \cdot y' = g
$$

이제 항을 정리하면 다음과 같습니다.

$$
a \cdot \underbrace{y'}_x + b \cdot \underbrace{(x' - \Big \lfloor \frac{a}{b} \Big \rfloor \cdot y')}_y = g
$$

이를 초기 표현식과 비교하면, $a$와 $b$의 계수를 $x$와 $y$로 대체할 수 있다는 것을 추론할 수 있습니다.

### 구현

알고리즘은 재귀 함수로 구현할 수 있습니다. 출력값이 하나가 아니라 세 개(최대 공약수, 계수 $x$, 계수 $y$)이기 때문에, 계수를 참조로 전달합니다.

```c++
int gcd(int a, int b, int &x, int &y) {
    if (a == 0) {
        x = 0;
        y = 1;
        return b;
    }
    int x1, y1;
    int d = gcd(b % a, a, x1, y1);
    x = y1 - (b / a) * x1;
    y = x1;
    return d;
}
```

역원을 계산하기 위해서는 단순히 $a$와 $m$을 이 함수에 전달하고, 결과로 얻는 $x$ 계수를 반환하면 됩니다. 이때 양의 정수 두 개를 입력으로 주기 때문에, $x$와 $y$ 중 하나는 양수이고 하나는 음수가 됩니다. 어떤 쪽이 음수인지는 재귀 깊이(즉, 반복 횟수)가 홀짝이냐에 따라 달라집니다. 따라서, $x$가 음수일 경우에는 $m$을 더해 올바른 나머지 값을 구해주어야 합니다.

```c++
int inverse(int a) {
    int x, y;
    gcd(a, M, x, y);
    if (x < 0)
        x += M;
    return x;
}
```

이 구현은 약 160ns 정도의 시간에 동작하며, [이진 거듭제곱법](../exponentiation)을 사용한 역원 계산보다 약 10ns정도 더 빠릅니다. 이를 더 최적화하려면, 재귀 대신 반복문을 사용하는 방식으로 바꿀 수 있으며, 이 경우 약 135ns로 성능이 향상됩니다.

```c++
int inverse(int a) {
    int b = M, x = 1, y = 0;
    while (a != 1) {
        y -= b / a * x;
        b %= a;
        swap(a, b);
        swap(x, y);
    }
    return x < 0 ? x + M : x;
}
```

단, 이진 거듭제곱법과 달리 확장 유클리드 알고리즘의 실행 시간은 입력값 $a$에 따라 달라집니다. 예를 들어 $m = 10^9 + 7$일 때, 최악의 경우는 $a = 564400443$으로, 이 경우 알고리즘은 37번의 반복을 수행하며 약 250ns가 소요됩니다.

연습 문제로, 위와 같은 방식으로 [이진 GCD](/hpc/algorithms/gcd/#binary-gcd) 알고리즘에도 유사한 기법을 적용해보세요. 단, 별도의 고급 최적화 없이 단순히 변형하는 것만으로는 성능 향상은 기대하기 어렵습니다.
