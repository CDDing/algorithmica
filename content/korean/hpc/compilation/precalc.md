---
title: Precomputation
weight: 8
---

컴파일러가 어떤 변수가 사용자 입력값에 의존하지 않는다는 것을 추론할 수 있으면, 해당 변수의 값을 컴파일 시점에 계산해 기계어 코드에 상수로 삽입할 수 있습니다.

이 최적화는 성능 향상에 큰 도움이 되지만, C++ 표준에 명시된 동작은 아니므로 컴파일러가 반드시 수행해야 하는 것은 아닙니다. 컴파일 타임 계산이 구현하기 까다롭거나 시간이 오래걸리는 경우, 컴파일러는 이 최적화를 생략할 수 있습니다.

### 상수 표현식

조금 더 신뢰할 수 있는 방법으로, 현대 C++에서는 함수에 `constexpr` 키워드를 붙일 수 있습니다. 이렇게 정의된 함수에 상수값을 인자로 전달하면, 해당 함수는 반드시 컴파일 시점에 계산됩니다.

```c++
constexpr int fibonacci(int n) {
    if (n <= 2)
        return 1;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

static_assert(fibonacci(10) == 55);
```

이러한 함수는 몇 가지 제약이 있습니다. 예를 들어, 다른 `constexpr` 함수만 호출할 수 있고, 동적 메모리 할당을 할 수 없습니다. 하지만 이러한 제약을 제외하면, 일반 함수처럼 있는 그대로 실행됩니다.

`constexpr` 함수는 런타임에는 비용이 들지 않지만 컴파일 시간은 증가시킨다는 점을 기억하세요. 따라서 효율성에 어느 정도 신경써야 하며, NP-complete 문제 같은 무거운 연산을 넣는 것은 피해야 합니다.

```c++
constexpr int fibonacci(int n) {
    int a = 1, b = 1;
    while (n--) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

초기 C++ 표준에서는 더 많은 제약이 있었는데, 예를 들어 내부 상태를 가질 수 없고 재귀에 의존해야 했습니다. 그래서 전체적인 프로그래밍 스타일이 C++ 보다는 Haskell에 더 가까운 느낌이었습니다. 하지만 C++17부터는 명령형 스타일로 정적 배열을 계산할 수 있게 되었고, 이는 lookup 테이블을 미리 계산할 때 유용하게 사용할 수 있습니다.

```c++
struct Precalc {
    int isqrt[1000];

    constexpr Precalc() : isqrt{} {
        for (int i = 0; i < 1000; i++)
            isqrt[i] = int(sqrt(i));
    }
};

constexpr Precalc P;

static_assert(P.isqrt[42] == 6);
```

한 가지 주의할 점은, `constexpr` 함수라 할지라도 비상수값을 인자로 전달하면 컴파일러가 컴파일 시점에 해당 함수를 계산할지 말지 선택할 수 있다는 점입니다.

```c++
for (int i = 0; i < 100; i++)
    cout << fibonacci(i) << endl;
```

이 예제에서는 반복 횟수가 상수이고, `fibonacci`에 전달되는 값도 컴파일 시점에 알려진 값이긴 하지만, 진정한 의미의 컴파일 타임 상수는 아니기 때문에 컴파일러가 이 루프를 최적화할지는 전적으로 컴파일러의 판단에 달려 있습니다. 그리고 계산량이 무거운 경우에는 종종 최적화를 수행하지 않기도 합니다.

<!--

### Code Generation

There are plenty of languages that support computing *data* during compile time, but none can produce efficient code at all times.

One huge example is generating lexers and parsers: which is usually done in.

For example, CUDA and OpenCL are mostly C, and have no support for metaprogramming.

At some point (and perhaps to this day), these languages had no way to unroll loops, so people would write a [jinja template](https://jinja.palletsprojects.com/en/3.0.x/), call the thing from Python, and then compile.

It is not uncommon to use a templating engine to generate code. For example, CUDA (a GPU programming language) has no loop unrolling

-->
