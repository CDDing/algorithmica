---
title: Instrumentation
weight: 1
published: true
---

<!-- pv in Linux, pipes -->

Instrumentation은 타이머나 기타 추적 코드를 프로그램에 삽입하는 것을 의미하는, 다소 복잡하게 들리는 용어입니다. 가장 단순한 예는 Unix 계열 시스템에서 `time` 유틸리티를 사용해 전체 프로그램의 실행 시간을 측정하는 것입니다.

보다 일반적으로는, 어떤 부분이 최적화가 필요한 지 알아야 합니다. 일부 컴파일러나 IDE에는 특정 함수의 실행 시간을 자동으로 측정하는 도구가 내장되어 있지만, 언어가 제공하는 시간 측정 기능을 사용해 직접 측정하는 것이 더 유연하고 강력한 방법입니다.

```cpp
clock_t start = clock();
do_something();
float seconds = float(clock() - start) / CLOCKS_PER_SEC;
printf("do_something() took %.4f", seconds);
```

여기서 한 가지 주의할 점은, 특히 빠르게 실행되는 함수의 경우 이 방식으로 시간을 정확히 측정할 수 없다는 것입니다. `clock` 함수는 현재 시간을 마이크로초($10^{-6}$) 단위로 반환하며, 자체 실행 시간도 수백 나노초가 걸릴 수 있습니다. 다른 시간 관련 유틸리티도 마찬가지로 최소 마이크로초 단위의 정밀도를 가지므로, 로우레벨 최적화에서는 이조차도 너무 느리게 느껴질 수 있습니다.

더 높은 정밀도를 얻기 위해선, 해당 함수를 반복문으로 여러 번 호출한 뒤 전체 실행 시간을 측정하고 반복 횟수로 나누는 방식이 있습니다.

```cpp
#include <stdio.h>
#include <time.h>

const int N = 1e6;

int main() {
    clock_t start = clock();

    for (int i = 0; i < N; i++)
        clock(); // benchmarking the clock function itself

    float duration = float(clock() - start) / CLOCKS_PER_SEC;
    printf("%.2fns per iteration\n", 1e9 * duration / N);

    return 0;
}
```

또한 캐싱되거나, 컴파일러에 의해 최적화되거나, 유사한 사이드 이펙트의 영향을 받지 않도록 보장해야 합니다. 이는 별도의 주제이며, 이 장의 [마지막](../benchmarking)에서 더 자세히 다룰 것입니다.

### 이벤트 샘플링

Instrumentation은 특정 알고리즘의 성능을 분석할 때 유용한 다른 종류의 정보를 수집하는 데에도 사용할 수 있습니다. 예를 들면 다음과 같습니다.


- 해시 함수의 경우, 입력의 평균 길이가 중요할 수 있습니다.
- 이진 트리라면 트리의 크기와 높이가 주요 관심사입니다.
- 정렬 알고리즘의 경우 얼마나 많은 비교 연산이 일어났는지를 알고 싶습니다.

이와 같이 알고리즘 특화 통계 정보를 수집하려면 코드에 카운터를 삽입해 계산할 수 있습니다.

물론 카운터를 추가하면 성능에 오버헤드가 발생할 수 있습니다. 그러나 이 문제는 호출 횟수 중 일부만 무작위로 통계를 수집하도록 설정하면 거의 완전히 줄일 수 있습니다.

```c++
void query() {
    if (rand() % 100 == 0) {
        // update statistics
    }
    // main logic
}
```

샘플링 비율이 충분히 작다면, 남은 오버헤드는 무작위 난수 생성과 조건 분기정도 뿐입니다. 흥미롭게도, 이마저도 약간의 통계 기법을 활용해 더 최적화할 수 있습니다.

수학적으로, 우리는 현재 성공할 때까지 [베르누이 분포](https://en.wikipedia.org/wiki/Bernoulli_distribution)에서 샘플링하는 방식으로 통계를 수집하고 있습니다. 여기서 첫 번째 성공까지 필요한 시행 횟수를 알려주는 분포가 [기하 분포](https://en.wikipedia.org/wiki/Geometric_distribution)입니다. 따라서 우리는 베르누이 시행 대신 기하 분포에서 샘플링해 그 값을 감소 카운터로 사용할 수 있습니다.

```c++
void query() {
    static next_sample = geometric_distribution(sample_rate);
    if (next_sample--) {
        next_sample = geometric_distribution(sample_rate);
        // ...
    }
    // ...
}
```

이 방법을 사용하면 매번 난수를 생성하지 않아도 되며, 통계를 실제로 수집할 때에만 카운터를 초기화하면 됩니다.

이런 기술은 대형 프로젝트에서 라이브러리 수준의 알고리즘 개발자들이 최종 프로그램 성능에 큰 영향을 주지 않으면서 프로파일링 데이터를 수집하는 데 자주 사용됩니다.
