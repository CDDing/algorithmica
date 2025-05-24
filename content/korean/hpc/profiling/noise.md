---
title: Getting Accurate Results
weight: 10
published: true
---

서로 다른 벤치마킹 코드를 유지하며 자신이 더 빠르다고 주장하는 두 개의 라이브러리 알고리즘 구현이 존재하는 것은 드문 일이 아닙니다. 이는 특히 둘 중 하나를 선택해야 하는 사용자들을 포함해 모든 사람들을 혼란스럽게 만듭니다.

이러한 상황은 대개 개발자의 부정행위 때문이 아니라, 단순히 "더 빠르다"는 것에 대한 정의가 서로 다르기 때문에 발생합니다. 실제로 하나의 성능 지표만으로 속도를 정의하고 사용하는 것은 종종 문제가 될 수 있습니다.

### 올바르게 측정하기

벤치마킹에 편향을 일으킬 수 있는 요인은 매우 많습니다.

**서로 다른 데이터셋**. 알고리즘의 성능이 데이터셋의 분포에 따라 달라지는 경우가 많습니다. 예를 들어 가장 빠른 정렬 알고리즘, 최단 경로 알고리즘, 또는 이진 탐색 알고리즘이 무엇인지 정의하려면, 알고리즘이 실행되는 데이터셋을 고정해야 합니다.

이런 문제는 단일 입력값을 처리하는 알고리즘에도 해당될 수 있습니다. 예를 들어, 최대공약수 구현에 연속된 숫자를 입력하면 분기가 지나치게 예측 가능해지기 때문에 좋은 벤치마킹이 되지 않습니다.

```c++
// don't do this
int checksum = 0;

for (int a = 0; a < 1000; a++)
    for (int b = 0; b < 1000; b++)
        checksum ^= gcd(a, b);
```

하지만 동일한 숫자들을 무작위로 샘플링하면 분기 예측이 더 어려워지고, 같은 데이터를 처리하더라도 순서가 달라졌기 때문에 벤치마킹 수행 시간은 더 길어질 수 있습니다.

```c++
int a[1000], b[1000];

for (int i = 0; i < 1000; i++)
    a[i] = rand() % 1000, b[i] = rand() % 1000;

int checksum = 0;

for (int t = 0; t < 1000; t++)
    for (int i = 0; i < 1000; i++)
        checksum += gcd(a[i], b[i]);
```


대부분의 경우 데이터를 균등하게 무작위로 샘플링하는 것이 가장 논리적인 선택처럼 보이지만, 실제 애플리케이션에서는 데이터 분포가 균등하지 않은 경우가 많기 때문에, 단일한 방식으로는 충분하지 않습니다. 일반적으로 좋은 벤치마크란 특정 애플리케이션에 맞춰져 있어야 하며, 실제 사용 사례를 최대한 잘 반영하는 데이터셋을 사용하는 것이 바람직합니다.

<!--

People report things they like to report and leave out the things they don't.

To put numbers in perspective, use statistics like "ns per query" or "cycles per byte" instead of wall clock whenever it is applicable. When you start to approach very high levels of performance, it makes sense to calculate what the theoretically maximal performance is and start thinking about your algorithm performance as a fraction of it.

Similar to how Americans report pre-tax salary, Americans use non-PPP-adjusted stats, attention-seeking startups report revenue instead of profit, performance engineers report the best version of benchmark if not stated otherwise.


This happens especially often for data structures, and in general for algorithms whose performance somehow depends on the dataset distribution.

-->

**여러 목표**. 일부 알고리즘 설계 문제는 하나 이상의 핵심 목표를 가질 수 있습니다. 예를 들어, 해시 테이블은 키의 분포에 크게 영향을 받을 뿐만 아니라, 다음과 같은 요소들 간의 균형을 신중하게 맞춰야 합니다.

- 메모리 사용량
- add 쿼리의 지연
- 해시 테이블에 실제 키가 존재할 경우의 지연
- 해시 테이블에 실제 키가 존재하지 않을 경우의 지연

해시 테이블 구현들 중에서 하나를 선택하는 유일한 방법은, 여러 구현을 실제 애플리케이션에 넣어 테스트해보는 것입니다.

**지연 vs 처리량**. 사람들이 종종 간과하는 또 다른 측면은, 실행 시간이 단일 쿼리에 대해서도 여러 방식으로 정의될 수 있다는 점입니다.

다음과 같은 코드를 작성했다고 가정해봅시다.

```c++
for (int i = 0; i < N; i++)
    q[i] = rand();

int checksum = 0;

for (int i = 0; i < N; i++)
    checksum ^= lower_bound(q[i]);
```

전체 실행 시간을 측정한 뒤 반복 횟수로 나누면, 이는 쿼리의 처리량을 측정하는 것입니다. 단위 시간당 얼마나 많은 작업을 처리할 수 있는지를 의미하는 것입니다. 이 값은 실제로 하나의 연산을 독립적으로 처리하는 데 걸리는 시간보다 짧은 경우가 많습니다. 이는 연산 간의 병렬 처리(interleaving)로 인한 것입니다.

반면, 실제 지연 시간을 측정할며ㅕㄴ 각 호출 간에 의존성을 추가해야 합니다.

```c++
for (int i = 0; i < N; i++)
    checksum ^= lower_bound(checksum ^ q[i]);
```

이는 파이프라인 정지(pipeline stall) 문제가 발생할 수 있는 알고리즘, 예를 들어 분기가 있는 알고리즘과 분기가 없는 알고리즘을 비교할 때 큰 차이를 만들어냅니다.

**콜드 캐시**. 또 다른 편차의 원인은 콜드 캐시 효과입니다. 필요한 데이터가 아직 캐시에 적재되지 않은 상태에서 메모리를 처음 읽을 때 시간이 더 오래 걸리는 현상입니다.

이 문제는 벤치마크 측정을 시작하기 전에 워밍업 실행을 통해 해결할 수 있습니다.

```c++
// warm-up run

volatile checksum = 0;

for (int i = 0; i < N; i++)
    checksum ^= lower_bound(q[i]);


// actual run

clock_t start = clock();
checksum = 0;

for (int i = 0; i < N; i++)
    checksum ^= lower_bound(q[i]);
```

워밍업 실행이 단순한 체크섬 계산 이상의 작업을 수행해야 할 경우, 정답 검증 과정과 워밍업을 함께 수행하는 것도 유용할 수 있습니다.

**지나친 최적화**. 컴파일러가 벤치마크 코드를 과하게 최적화해버려 측정 결과가 왜곡되는 경우도 있습니다. 이를 방지하려면 체크섬을 추가하고 그 값을 출력하거나, 반복문이 병합되는 것을 막기 위해 `volatile` 지정자를 사용해야 합니다.

데이터를 쓰기만 하는 알고리즘의 경우, `__sync_synchronize()` 내장 함수를 사용하여 메모리 펜스를 삽입함으로써, 컴파일러가 쓰기 작업을 누적하거나 최적화하지 못하도록 막을 수 있습니다.

### 오차 줄이기

<!--

https://github.com/sosy-lab/benchexec

-->

앞서 설명한 문제들은 벤치마킹 측정에 편향을 발생시킵니다. 이로 인해 한 알고리즘이 다른 알고리즘보다 지속적으로 유리한 결과를 얻게 됩니다. 이 외에도 예측 불가능한 왜곡이나 완전히 무작위적인 노이즈를 유발하는 다른 유형의 벤치마킹 문제들이 있으며, 이는 결과의 분산을 증가시킵니다.

이러한 문제는 사이드 이펙트와 외부 노이즈에 의해 발생하며, 주로 주변 프로세스나 CPU 주파수 스케일링 같은 요소 때문입니다.

- 만약 계산 중심 알고리즘을 벤치마크하고 있다면, `perf stat`을 이용해 CPU 사이클 수로 성능을 측정하세요. 이렇게 하면 클럭 주파수의 흔들림과 무관하게 정확한 비교가 가능합니다.
- 그 외의 경우, CPU 코어의 주파수를 기대한 값으로 고정시키고, 다른 요소가 간섭하지 않도록 하세요. 리눅스에서는 `cpupower`를 이용할 수 있습니다.(예: 최소 클럭으로 설정 = `sudo cpupower frequency-set -g powersave`, 터보 부스트 활성화 = `sudo cpupower frequency-set -g ondemand`) GNOME 사용자라면 이 [확장 기능](https://extensions.gnome.org/extension/1082/cpufreq/)도 유용합니다.
- 가능하다면 하이퍼스레딩을 비활성화하고, 작업을 특정 코어에 고정시키세요. 백그라운드에서 다른 작업이 실행되지 않도록 하고, 네트워크 연결을 끄고, 마우스를 움직이지 않도록 주의하세요.

편차를 완전히 제거하는 것은 불가능합니다. 심지어 프로그램 이름조차도 성능에 영향을 줄 수 있습니다. 실행 파일이름은 환경 변수에 저장되며, 이는 콜 스택에 영향을 줍니다. 이때 이름의 길이는 스택 정렬에 영향을 미쳐 캐시라인 또는 메모리 페이지 경계를 넘어서는 데이터 접근이 발생하게 되며, 이로 인해 성능 저하가 생길 수 있습니다.

최적화 작업을 진행하거나, 특히 결과를 다른 사람에게 공유할 때 이러한 노이즈를 고려하는 것이 매우 중요합니다. 2배 이상의 성능 향상이 기대되는 경우가 아니라면, 모든 마이크로벤치마크를 A/B 테스트와 같은 방식으로 신중하게 처리해야 합니다.

노트북에서 프로그램을 1초 미만으로 실행할 경우, 성능에 ±5% 정도의 편차가 발생하는 것은 매우 일반적입니다. 따라서 1% 정도의 미세한 성능 향상 여부를 판단하고자 할 경우, 분산과 p값을 계산하여 통계적으로 유의미한 결과가 나올 때까지 반복 실험해야 합니다.

### 읽을거리

Dror Feitelson이 정리한 [실험적 컴퓨터 과학 자료 목록](https://www.cs.huji.ac.il/w~feit/exp/related.html)이나 Todd Mytkowicz 외 저자들의 논문 "[Producing Wrong Data Without Doing Anything Obviously Wrong](http://eecs.northwestern.edu/~robby/courses/322-2013-spring/mytkowicz-wrong-data.pdf)"을 살펴보는 것도 좋습니다.

Emery Berger의 [훌륭한 강연](https://www.youtube.com/watch?v=r-TLSBdHe1A)을 감상하는 것도 좋습니다.
