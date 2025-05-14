---
title: Situational Optimizations
weight: 3
---

<!--

Generally, you always want to specify the exact platform you are running and turn on `-O3`, but other optimizations, like the ones discussed [in the previous section](../assembly), are far more situational and require some input from the programmer.

-->

`-O2` 및 `-O3`로 활성화되는 대부분의 컴파일러 최적화는 성능을 향상시키거나, 적어도 심각하게 저하시키지 않는 것을 보장합니다. `-O3`에 포함되지 않은 최적화들은 일반적으로 엄격한 표준을 완전히 준수하지 않거나, 매우 특정한 상황에서만 유용하며, 사용 여부를 판단하기 위해 프로그래머의 추가적인 입력이 필요한 경우입니다.

이번 장에서는 이 책에서 앞서 다룬 것들 중 가장 자주 사용되는 최적화 기법들을 살펴보겠습니다.

### 루프 펼치기

[루프 펼치기](/hpc/architecture/loops#loop-unrolling)는 기본적으로 비활성화되어 있으며, 컴파일 시점에 반복 횟수가 작고 상수로 정해진 경우에만 자동으로 적용됩니다. 이 경우, 루프는 점프 명령 없이 반복되는 명령어 시퀀스로 대체됩니다. `-funroll-loops` 플래그를 사용하면 전역적으로 활성화할 수 있으며, 이 플래그는 반복 횟수가 컴파일 타임이나 루프 진입 시점에 결정 가능한 모든 루프를 펼칩니다.

특정 루프에 대해서는 pragma 지시문을 사용해 개별적으로 지정할 수도 있습니다.

```c++
#pragma GCC unroll 4
for (int i = 0; i < n; i++) {
    // ...
}
```

루프 펼치기는 바이너리 크기를 증가시키며, 성능이 향상될 수도, 저하될 수도 있습니다. 따라서 무분별한 사용은 금물입니다.

### 함수 인라이닝

[함수 인라이닝](/hpc/architecture/functions#inlining)은 일반적으로 컴파일러가 판단하는 것이 가장 좋지만, `inline` 키워드를 통해 어느 정도 힌트를 줄 수 있습니다.

```c++
inline int square(int x) {
    return x * x;
}
```

하지만 컴파일러가 성능 이득이 크지 않다고 판단하면 이 힌트는 무시될 수 있습니다. `always_inline` 속성을 사용하면 인라이닝을 강제로 적용할 수 있습니다.

```c++
#define FORCE_INLINE inline __attribute__((always_inline))
```

인라인 함수의 크기(명령어 수 기준)에 대한 한계를 설정할 수 있는 `-finline-limit=n` 옵션도 존재합니다. Clang에서는 이와 동일한 기능이 `inline-threshold` 옵션입니다.

### 분기 가능성

[분기 가능성](/hpc/architecture/layout#unequal-branches)에 대한 힌트를 컴파일러에게 주기 위해 `[[likely]]` 및 `[[unlikely]]` 속성을 사용할 수 있습니다.

```c++
int factorial(int n) {
    if (n > 1) [[likely]]
        return n * factorial(n - 1);
    else [[unlikely]]
        return 1;
}
```

이는 C++20에서 새롭게 도입된 기능입니다. 이전에는 조건 표현식을 감싸기 위해 컴파일러별로 제공되는 고유 함수(intrinsic)를 사용해야 했습니다. 아래는 구버전 GCC에서의 동일한 예시입니다.

```c++
int factorial(int n) {
    if (__builtin_expect(n > 1, 1))
        return n * factorial(n - 1);
    else
        return 1;
}
```

<!--
What it usually does is it swaps the branches so that the more likely one goes immediately after jump (recall that "don't jump" branch is taken by default). The performance gain is usually rather small, because for most hot spots hardware branch prediction works just fine.
-->

이처럼 컴파일러에게 명시적으로 방향을 제시해야하는 경우는 더 많지만, 그 밖의 내용은 이후에 더 적절한 시점에 다루겠습니다.

### Profile-Guided Optimization

소스 코드에 이런 모든 메타데이터를 추가하는 작업은 번거롭습니다. 사람들은 그런 것이 아니어도 C++ 작성 자체를 이미 싫어합니다.

특정 최적화가 실제로 이득이 되는지 여부도 항상 명확하지 않습니다. 분기 재배치, 함수 인라이닝, 루프 펼치기 같은 결정을 내리기 위해선 다음과 같은 질문에 답해야 합니다.

- 이 분기는 얼마나 자주 실행되는가?
- 이 함수는 얼마나 자주 호출되는가?
- 이 루프는 평균적으로 몇 번 반복되는가?

다행히도, 이런 실제 데이터를 자동으로 제공할 수 있는 방법이 있습니다.

Profile-guided optimization(PGO, 발음때문에 pogo라고도 불립니다)는 정적 분석만으로는 얻을 수 없는 성능 향상을 위해 [프로파일링 데이터](/hpc/profiling)를 활용하는 기법입니다. 요약하자면, 프로그램의 주요 지점에 타이머와 카운터를 삽입한 상태로 컴파일하고, 이를 실제 데이터를 바탕으로 실행하여 정보를 수집한 후, 이 정보를 반영해 다시 컴파일하는 방식입니다.

이 모든 과정은 최신 컴파일러에서 자동으로 수행할 수 있습니다. 예를 들어, `fprofile-generate` 플래그를 사용하면 GCC는 프로그램에 프로파일링용 코드를 삽입합니다.

```
g++ -fprofile-generate [other flags] source.cc -o binary
```

그런 다음 프로그램을 실행하면(실제 사용 사례를 잘 대표할 수 있는 입력을 사용하는 것이 좋습니다), 실행 로그 데이터가 담긴 `*.gcda` 파일들이 생성됩니다. 이 파일들을 바탕으로 `-fprofile-use` 플래그를 추가해 프로그램을 다시 빌드할 수 있습니다.

```
g++ -fprofile-use [other flags] source.cc -o binary
```

이 방법은 일반적으로 대규모 코드베이스에서 10~20% 정도의 성능 향상을 가져오며, 성능이 중요한 프로젝트의 빌드 과정에 흔히 포함되는 이유도 이 때문입니다. 견고한 벤치마킹 코드를 작성하는 데 투자해야 하는 또 하나의 이유이기도 합니다.

<!--

We will study how profiling works more deeply in the [next chapter](../../profiling).

-->
