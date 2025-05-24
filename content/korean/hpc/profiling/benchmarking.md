---
title: Benchmarking
weight: 6
---

대부분의 훌륭한 소프트웨어 공학적 실천은 어떤 방식으로든 개발 주기를 단축하는 데 목적이 있습니다. 예를 들어, 소프트웨어를 더 빠르게 컴파일하고자 하기도 하고(빌드 시스템), 버그를 최대한 빨리 잡고자 하며(정적 분석, 지속적 통합), 새로운 버전이 준비되는 즉시 배포하려 하고(지속적 배포), 사용자 피드백에 지체없이 대응하려고 합니다(애자일 개발).

성능 엔지니어링도 예외는 아닙니다. 제대로 수행된다면, 성능 최적화 또한 다음 사이클을 따르게 됩니다.

1. 프로그램을 실행하고 지표를 수집합니다.
2. 병목이 어디인지 밝혀냅니다.
3. 병목을 제거하고 1단계로 되돌아갑니다.

이번 글에서는 벤치마킹에 대해 다루고, 이 사이클을 더 짧게 만들고 더 빠르게 반복할 수 있도록 도와주는 실용적인 기법들을 소개하겠습니다. 여기서 소개하는 대부분의 조언은 실제 이 책을 작성하면서 적용해 본 것들이며, 책의 [코드 저장소](https://github.com/sslotin/ahm-code)에서 그 예제들을 직접 확인하실 수 있습니다.

### C++ 벤치마킹

벤치마킹 코드를 작성하는 데에는 여러 가지 접근 방식이 있습니다. 아마도 가장 널리 사용되는 방법은 비교하고자 하는 여러 동일 언어 구현을 하나의 파일에 포함하고, `main` 함수에서 각각을 호출하며, 동일한 소스 파일 내에서 원하는 지표들을 계산하는 것입니다.

The disadvantage of this method is that you need to write a lot of boilerplate code and duplicate it for each implementation, but it can be partially neutralized with metaprogramming. For example, when you are benchmarking multiple [gcd](/hpc/algorithms/gcd) implementations, you can reduce benchmarking code considerably with this higher-order function:
이 기법의 단점은 상당한 양의 반복 코드를 작성하고 구현마다 중복해서 작성해야 한다는 것이지만, 메타프로그래밍을 통해 부분적으로 중립화할 수 있습니다. 예를 들어, 여러 gcd 구현을 벤치마킹할 때, 이 더 높은순서 함수를 통해 상당한 양의 벤치마킹 코드를 줄일 수 있습니다.
이 방법의 단점은 구현마다 상당량의 반복적인 코드를 작성해야 한다는 점입니다. 하지만 메타프로그래밍을 사용하면 이를 어느 정도 완화할 수 있습니다. 예를 들어, 여러 gcd 구현을 벤치마킹할 때 아래와 같은 고차 함수를 사용하면 벤치마킹 코드를 상당히 줄일 수 있습니다.

```c++
const int N = 1e6, T = 1e9 / N;
int a[N], b[N];

void timeit(int (*f)(int, int)) {
    clock_t start = clock();

    int checksum = 0;

    for (int t = 0; t < T; t++)
        for (int i = 0; i < n; i++)
            checksum ^= f(a[i], b[i]);
    
    float seconds = float(clock() - start) / CLOCKS_PER_SEC;

    printf("checksum: %d\n", checksum);
    printf("%.2f ns per call\n", 1e9 * seconds / N / T);
}

int main() {
    for (int i = 0; i < N; i++)
        a[i] = rand(), b[i] = rand();
    
    timeit(std::gcd);
    timeit(my_gcd);
    timeit(my_another_gcd);
    // ...

    return 0;
}
```

이 방식은 오버헤드가 매우 낮아 더 많은 실험을 수행하고 [더 정확한 결과](../noise)를 얻는 데 유리합니다. 여전히 반복되는 작업이 존재하지만, 이런 작업들은 프레임워크를 통해 대부분 자동화할 수 있습니다. C++에서는 [Google의 Benchmark 라이브러리](https://github.com/google/benchmark)가 가장 널리 사용됩니다. 일부 프로그래밍 언어는 벤치마킹을 위한 유용한 내장 도구도 제공합니다. 그중 대표적으로는 [Python의 timeit 함수](https://docs.python.org/3/library/timeit.html)와 [Julia의 @benchmark 매크로](https://github.com/JuliaCI/BenchmarkTools.jl)가 있습니다.

C와 C++은 실행 속도 측면에서는 효율적이지만, 분석 작업과 관련해서는 가장 생산적인 언어는 아닙니다. 알고리즘이 입력 크기 같은 매개변수에 따라 달라지고, 각 구현에서 하나 이상의 데이터를 수집해야 할 경우, 벤치마킹 코드를 외부 환경과 통합하고 결과를 다른 도구로 분석하고 싶어질 수 있습니다.

### 구현부 나누기

모듈성과 재사용성을 향상시키는 한 가지 방법은 알고리즘의 실제 구현과 테스트 및 분석 코드를 분리하는 것입니다. 또한 각 버전을 서로 다른 파일에 구현하되, 동일한 인터페이스를 제공하도록 구성하면 좋습니다.

C와 C++에서는, 예를 들어 `gcd.hh`라는 단일 헤더 파일에 함수 인터페이스를 정의하고, `main` 함수 안에는 벤치마킹 코드를 작성함으로써 이를 실현할 수 있습니다.

```c++
int gcd(int a, int b); // to be implemented

// for data structures, you also need to create a setup function
// (unless the same preprocessing step for all versions would suffice)

int main() {
    const int N = 1e6, T = 1e9 / N;
    int a[N], b[N];
    // careful: local arrays are allocated on the stack and may cause stack overflow
    // for large arrays, allocate with "new" or create a global array

    for (int i = 0; i < N; i++)
        a[i] = rand(), b[i] = rand();

    int checksum = 0;

    clock_t start = clock();

    for (int t = 0; t < T; t++)
        for (int i = 0; i < n; i++)
            checksum += gcd(a[i], b[i]);
    
    float seconds = float(clock() - start) / CLOCKS_PER_SEC;

    printf("%d\n", checksum);
    printf("%.2f ns per call\n", 1e9 * seconds / N / T);
    
    return 0;
}
```

그 다음에는 각 알고리즘 버전마다 별도의 구현 파일(예: `v1.cc`, `v2.cc` 등)을 만들고, 앞서 작성한 헤더 파일을 포함하면 됩니다.

```c++
#include "gcd.hh"

int gcd(int a, int b) {
    if (b == 0)
        return a;
    else
        return gcd(b, a % b);
}
```

이렇게 하는 주된 목적은 소스 코드 파일을 직접 수정하지 않고도 명령줄에서 특정 버전의 알고리즘을 벤치마킹할 수 있게 하려는 것입니다. 이때 알고리즘이 사용하는 매개변수들을 외부에서 설정할 수 있도록 노출하는 것이 좋습니다. 예를 들어, 명령줄 인자를 파싱하는 방법이 있습니다.

```c++
int main(int argc, char* argv[]) {
    int N = (argc > 1 ? atoi(argv[1]) : 1e6);
    const int T = 1e9 / N;

    // ...
}
```

또는 C 스타일의 전역 define을 사용하고, 컴파일 시 `-D N=...` 플래그를 넘기는 방법도 있습니다.

```c++
#ifndef N
#define N 1000000
#endif

const int T = 1e9 / N;
```

이런 방식은 컴파일 타임 상수를 활용할 수 있게 해주며, 일부 알고리즘 성능을 높이는 데 큰 도움이 됩니다. 다만, 파라미터를 바꿀 때마다 프로그램을 다시 빌드해야 하므로, 다양한 파라미터 값에 대한 지표를 수집하는 데 시간이 더 걸릴 수 있다는 단점도 있습니다.

### Makefiles

<!-- TODO -->

소스 파일을 분리하면 [Make](https://en.wikipedia.org/wiki/Make_(software))와 같은 캐시 기반 빌드 시스템을 사용해 컴파일 속도를 높일 수 있습니다.

저는 보통 이 Makefile의 변형을 여러 프로젝트에서 공통적으로 사용합니다.

```c++
compile = g++ -std=c++17 -O3 -march=native -Wall

%: %.cc gcd.hh
	$(compile) $< -o $@ 

%.s: %.cc gcd.hh
	$(compile) -S -fverbose-asm $< -o $@

%.run: %
	@./$<

.PHONY: %.run
```

이제 `make example` 명령어로 `example.cc`를 컴파일할 수 있으며, `make example.run`으로 바로 실행할 수도 있습니다.

Makefile에 통계 계산용 스크립트를 추가하거나, `perf stat` 같은 도구를 연동하여 자동으로 프로파일링을 수행하도록 설정할 수도 있습니다.

### Jupyter Notebooks

고수준 분석을 더 빠르게 진행하려면, Jupyter notebook을 만들어 모든 스크립트와 시각화를 그 안에 통합하는 것이 좋습니다.

구현의 벤치마킹을 위한 래퍼 함수를 추가해 두면 편리하며, 이는 단일 스칼라 값을 반환합니다.

```python
def bench(source, n=2**20):
    !make -s {source}
    if _exit_code != 0:
        raise Exception("Compilation failed")
    res = !./{source} {n} {q}
    duration = float(res[0].split()[0])
    return duration
```

이제 이 함수를 사용해 깔끔한 분석 코드를 작성할 수 있습니다.

```python
ns = list(int(1.17**k) for k in range(30, 60))
baseline = [bench('std_lower_bound', n=n) for n in ns]
results = [bench('my_binary_search', n=n) for n in ns]

# plotting relative speedup for different array sizes
import matplotlib.pyplot as plt

plt.plot(ns, [x / y for x, y in zip(baseline, results)])
plt.show()
```

이런 워크플로우가 정착되면 훨씬 빠르게 반복할 수 있고, 알고리즘 자체의 최적화에 집중할 수 있게 됩니다.
