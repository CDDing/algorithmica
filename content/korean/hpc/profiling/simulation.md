---
title: Program Simulation
weight: 3
---

프로파일링을 위한 마지막 접근 방식은 프로그램을 실제로 실행하면서 데이터를 수집하는 것이 아니라, 특수한 도구를 사용해 시뮬레이션하여 어떤 일이 일어나는지를 분석하는 것입니다.

<!--

There are many subcategories of such profilers, differing in which aspect of computation is simulated, but the one we are going to focus on in this section is *machine code analyzers*.

The last approach (or rather a group of them) is not to gather the data by actually running the program, but to analyze what should happen by *simulating* it with specialized tools, which roughly fall into two categories.

-->

이러한 프로파일러는 시뮬레이션하는 계산 측면에 따라 여러 하위 범주로 나뉩니다. 이 글에서는 [캐싱](/hpc/cpu-cache)과 [분기 예측](/hpc/pipelining/branching)에 중점을 두고, 이를 위해 [Valgrind](https://valgrind.org/)의 일부인 [Cachegrind](https://valgrind.org/docs/manual/cg-manual.html)를 사용할 것입니다. Valgrind는 메모리 누수 탐지와 메모리 디버깅에 널리 사용되는 검증된 도구이며, Cachegrind는 그 중에서도 프로파일링에 특화된 구성 요소입니다.

### Cachegrind를 통한 프로파일링

Cachegrind는 기본적으로 바이너리에서 메모리 읽기/쓰기 또는 조건/간접 분기와 같은 의미 있는 명령어를 찾아낸 후, 해당 명령어를 소프트웨어 자료구조를 이용해 하드웨어 동작을 시뮬레이션하는 코드로 대체합니다. 따라서 소스 코드에 접근할 필요 없이 이미 컴파일된 프로그램에도 적용할 수 있으며, 다음과 같은 방식으로 어떤 실행 파일에도 사용할 수 있습니다.

```bash
valgrind --tool=cachegrind --branch-sim=yes ./run
#       also simulate branch prediction ^   ^ any command, not necessarily one process
```

이 명령은 관련된 모든 바이너리를 계측하고 실행하며, [perf stat](../events)과 유사한 요약 결과를 출력합니다.

```
I   refs:      483,664,426
I1  misses:          1,858
LLi misses:          1,788
I1  miss rate:        0.00%
LLi miss rate:        0.00%

D   refs:      115,204,359  (88,016,970 rd   + 27,187,389 wr)
D1  misses:      9,722,664  ( 9,656,463 rd   +     66,201 wr)
LLd misses:         72,587  (     8,496 rd   +     64,091 wr)
D1  miss rate:         8.4% (      11.0%     +        0.2%  )
LLd miss rate:         0.1% (       0.0%     +        0.2%  )

LL refs:         9,724,522  ( 9,658,321 rd   +     66,201 wr)
LL misses:          74,375  (    10,284 rd   +     64,091 wr)
LL miss rate:          0.0% (       0.0%     +        0.2%  )

Branches:       90,575,071  (88,569,738 cond +  2,005,333 ind)
Mispredicts:    19,922,564  (19,921,919 cond +        645 ind)
Mispred rate:         22.0% (      22.5%     +        0.0%   )
```

이전 글([perf stat을 사용한 이벤트 기반 프로파일링](../events))에서 사용했던 것과 동일한 예제 코드를 Cachegrind에 적용했습니다. 백만 개의 무작위 정수로 이루어진 배열을 만들고, 이를 정렬한 뒤, 해당 배열에서 백만 번의 이진 탐색을 수행하는 방식입니다. Cachegrind는 대체로 perf와 유사한 숫자를 보여주지만, perf에서는 [추측 실행](/hpc/pipelining) 때문에 메모리 읽기나 분기 횟수가 약간 더 많게 측정됩니다. 추측 실행은 실제 하드웨어에서 수행되며 하드웨어 카운터를 증가시키지만, 이후 폐기되고 실제 성능에는 영향을 주지 않기 때문에 Cachegrind의 시뮬레이션에서는 무시됩니다.

Cachegrind는 오직 첫 번째(데이터를 위한 `D1`, 명령어를 위한 `I1`)와 마지막(`LL`, 통합) 수준의 캐시만을 모델링하며, 이 캐시들의 특성은 시스템으로부터 유추됩니다. 하지만 명령줄 옵션을 통해 직저 설정할 수도 있기 때문에 제약을 주는 것은 아닙니다. 예를 들어 L2 캐시를 모델링하고 싶다면 다음과 같이 설정할 수 있습니다 : `-LL=<size>, <associativity>, <line size>`

현재까지는 단지 프로그램 속도만 느려졌고, `perf stat`이 제공하지 못하는 새로운 정보를 보여주지는 않는 것처럼 보입니다. 그러나 요약 정보 이상의 데이터를 얻고 싶다면, 프로파일링 정보를 담은 특별한 파일을 분석해야 합니다. 이 파일은 기본적으로 같은 디렉토리에 `cachegrind.out.<pid>` 형식으로 생성되며, 사람이 읽을 수는 있지만 일반적으로는 `cg_annotate` 명령어를 통해 읽는 것이 권장됩니다.

```bash
cg_annotate cachegrind.out.4159404 --show=Dr,D1mr,DLmr,Bc,Bcm
#                                    ^ we are only interested in data reads and branches
```

이 파일은 먼저 실행 중에 사용됐던 캐시 시스템의 특성을 보여줍니다.

```
I1 cache:         32768 B, 64 B, 8-way associative
D1 cache:         32768 B, 64 B, 8-way associative
LL cache:         8388608 B, 64 B, direct-mapped
```

L3 캐시에 대해서는 정확하게 모델링하지는 못했는데, 이는 L3가 통합 캐시가 아니며(전체 8MB이지만, 단일 코어에서는 4MB만 사용 가능), 16-way associative이기 때문입니다. 하지만 지금은 이 부분을 무시하겠습니다.

다음으로, `perf report`와 유사한 함수별 요약 정보를 출력합니다.

```
Dr         D1mr      DLmr Bc         Bcm         file:function
--------------------------------------------------------------------------------
19,951,476 8,985,458    3 41,902,938 11,005,530  ???:query()
24,832,125   585,982   65 24,712,356  7,689,480  ???:void std::__introsort_loop<...>
16,000,000        60    3  9,935,484    129,044  ???:random_r
18,000,000         2    1  6,000,000          1  ???:random
 4,690,248    61,999   17  5,690,241  1,081,230  ???:setup()
 2,000,000         0    0          0          0  ???:rand
```

정렬 단계에서 수많은 분기 오예측이 일어났으며, 이진 탐색 중 많은 L1 캐시 미스, 분기 오예측이 일어나기도 했습니다. 이처럼 세부적인 정보는 perf로는 얻을 수 없으며, 전체 프로그램 단위의 수치만 제공됩니다.

Cachegrind의 또 다른 놀라운 기능은 라인별 주석입니다. 이를 사용하려면 프로그램을 디버그 정보(`-g`)와 함께 컴파일해야 하며, `cg_annotate`에 어떤 소스 파일을 주석 처리할지 명시하거나, `--auto=yes` 옵션을 통해 접근 가능한 모든 파일(표준 라이브러리 포함)에 대해 자동으로 주석 처리할 수 있습니다.

그러므로 전체 소스 분석 과정은 다음과 같습니다.

```bash
g++ -O3 -g sort-and-search.cc -o run
valgrind --tool=cachegrind --branch-sim=yes --cachegrind-out-file=cachegrind.out ./run
cg_annotate cachegrind.out --auto=yes --show=Dr,D1mr,DLmr,Bc,Bcm
```

glibc 구현체는 읽을 수 없기 때문에, 예제를 설명하기 위해 `lower_bound` 대신 우리의 이진 탐색으로 대체합니다. 이는 다음과 같이 나타날 것입니다.

```c++
Dr         D1mr      DLmr Bc         Bcm       
         .         .    .          .         .  int binary_search(int x) {
         0         0    0          0         0      int l = 0, r = n - 1;
         0         0    0 20,951,468 1,031,609      while (l < r) {
         0         0    0          0         0          int m = (l + r) / 2;
19,951,468 8,991,917   63 19,951,468 9,973,904          if (a[m] >= x)
         .         .    .          .         .              r = m;
         .         .    .          .         .          else
         0         0    0          0         0              l = m + 1;
         .         .    .          .         .      }
         .         .    .          .         .      return l;
         .         .    .          .         .  }
```

아쉽지만 Cachegrind는 메모리 접근과 분기만을 추적합니다. 병목이 다른 곳에서 발생할 때는 [다른 시뮬레이션 도구](../mca)를 사용하는 편이 좋습니다.

