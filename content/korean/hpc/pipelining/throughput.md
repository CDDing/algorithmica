---
title: Throughput Computing
weight: 4
---

**지연**을 최적화하는 것과 **처리량**을 최적화하는 것은 보통 전혀 다른 접근을 필요로 합니다.

- 자료구조 조회나 단발성 또는 분기가 많은 알고리즘을 최적화할 때는, 해당 명령어의 [지연 시간을 확인](../tables)하고, 계산의 실행 그래프를 머릿속으로 구성한 뒤, 임계 경로(critical path)를 최대한 줄이도록 재구성해야 합니다. <!-- [Binary GCD](/hpc/algorithms/gcd) is a good example of that. -->
- 반면, 반복적으로 수행되는 루프나 대규모 데이터 셋을 처리하는 알고리즘을 최적화할 때는, 명령어의 처리량을 확인하고, 루프 한 번의 반복마다 각 명령어가 몇 번 사용되는지를 세어본 다음, 어떤 명령어가 병목이 되는지를 파악하여, 해당 명령어의 사용을 줄이도록 루프 구조를 바꿔야 합니다.

이러한 처리량 기반의 최적화는 각 반복이 이전 반복과 완전히 독립적인 데이터 병렬 루프에서만 유효합니다. 반복 간 상호 의존성이 존재한다면, 이전 반복이 완료될 때가지 다음 반복이 대기해야 하므로, [데이터 위험](../hazards)으로 인해 파이프라인 정지가 발생할 수 있습니다.

### 예제

배열의 합을 계산하는 간단한 예를 보겠습니다.

```c++
int s = 0;

for (int i = 0; i < n; i++)
    s += a[i];
```

이 루프를 컴파일러가 [벡터화](/hpc/simd)하지 않고, [메모리 대역폭](/hpc/cpu-cache/bandwidth)도 문제가 되지 않으며, 루프가 [펼쳐져](/hpc/architecture/loops) 루프 변수 유지에 드는 비용도 없다고 가정합시다. 이 경우 계산은 매우 단순해집니다.

```c++
int s = 0;
s += a[0];
s += a[1];
s += a[2];
s += a[3];
// ...
```

이 코드를 얼마나 빠르게 계산할 수 있을까요? 각 원소마다 정확히 한 사이클이 걸립니다. 각 반복마다 `s`에 값을 하나 더하기 위해 `add` 명령어가 한 사이클을 차지하기 때문입니다. 메모리 접근 지연은 영향을 주지 않습니다. CPU는 메모리 읽기를 미리 시작할 수 있기 때문입니다.

하지만 여기서 더 빨라질 수도 있습니다. 예를 들어 Zen 2 아키텍처의 CPU에서는 `add` 명령어의 처리량[^throughput]이 2이므로, 이론적으로 매 사이클마다 `add` 명령어를 두 번 실행할 수 있습니다. 그러나 현재 구조에서는 그럴 수 없습니다. `s`가 $i$번째 값을 누적하는 동안, 최소 한 사이클 동안 $(i+1)$번째 값은 누적할 수 없기 때문입니다. 

[^throughput]: 레지스터 간 `add`의 처리량은 4이지만, 두 번째 피연산자를 메모리에서 읽어오기 때문에 Zen 2에서는 메모리 `mov` 명령어의 처리량인 2가 병목이 됩니다.

이 문제는 누산기를 2개로 늘려 짝수와 홀수 인덱스의 값을 따로 누적하면 해결됩니다.

```c++
int s0 = 0, s1 = 0;
s0 += a[0];
s1 += a[1];
s0 += a[2];
s1 += a[3];
// ...
int s = s0 + s1;
```

이제 슈퍼 스칼라 CPU는 두 쓰레드를 병렬로 실행할 수 있으며, 계산은 처리량에 의해 제한되지 않게 됩니다.

<!--

By the virtue of out-of-order execution

-->

### 일반적인 경우

어떤 명령어가 지연시간 $x$와 처리량 $y$를 갖는다면, 해당 명령어의 성능을 완전히 포화시키기 위해서는 $x \cdot y$개의 누산기를 사용하는 것이 이상적입니다. 이는 $x \cdot y$개의 논리 레지스터가 필요하다는 뜻이기도 하며, 이는 고지연 명령어를 위한 실행 유닛 수를 제한하는 CPU 설계에서 중요한 고려사항이 됩니다.

이러한 기법은 주로 [SIMD](/hpc/simd) 환경에서 사용되며, 스칼라 코드에는 거의 적용되지 않습니다. 위의 코드를 [일반화](/hpc/simd/reduction)하면, 합산이나 그 외의 축소 연산을 컴파일러보다 빠르게 수행할 수도 있습니다.

반복문을 최적화할 때 일반적으로는 소수의 실행 포트를 최대한 활용하는 것이 중요하며, 나머지 루프 구조는 이를 중심으로 설계됩니다. 명령어마다 사용하는 실행 포트가 다르기 때문에, 어떤 포트가 과다하게 사용될지는 쉽게 예측할 수 없습니다. 이런 경우, [기계 코드 분석기](/hpc/profiling/mca)를 사용하면 짧은 어셈블리 루프의 병목을 파악하는데 매우 유용합니다.

<!--

Compilers don't always produce the optimal code.

This only applies to the variables that you have to preserve between iterations. You can "fire and forget" instructions that compute temporary values as much as you want.

Memory operations may have [very high latencies](/hpc/cpu-cache/latency), but you don't need hundreds or registers for them because  because they are bottlenecked for different reasons.

But they are bottlenecked for different reasons.

You still need to imaging execution graph, but now loop it around. In most cases, there is one instruction that is the bottleneck.

This is different. For single-invocation procedures you essentially want to minimize the latency on the critical data path. For stuff that gets called in a loop, you need to maximize throughput.

Bandwidth is the rate at which data can be read or stored. For the purpose of designing algorithms, a more important characteristic is the bandwidth-latency product which basically tells how many cache lines you can request while waiting for the first one without queueing up. It is around 5 or more on most systems. This is like having friends whom you can send for beers asynchronously.

In the previous version, we have an inherently sequential chain of operations in the innermost loop. We accumulate the minimum in variable v by a sequence of min operations. There is no way to start the second operation before we know the result of the first operation; there is no room for parallelism here:

The result will be clearly the same, but we are calculating the operations in a different order. In essence, we split the work in two independent parts, calculating the minimum of odd elements and the minimum of even elements, and finally combining the results. If we calculate the odd minimum v0 and even minimum v1 in an interleaved manner, as shown above, we will have more opportunities for parallelism. For example, the 1st and 2nd operation could be calculated simultaneously in parallel (or they could be executed in a pipelined fashion in the same execution unit). Once these results are available, the 3rd and 4th operation could be calculated simultaneously in parallel, etc. We could potentially obtain a speedup of a factor of 2 here, and naturally the same idea could be extended to calculating, e.g., 4 minimums in an interleaved fashion.

Instruction-level parallelism is automatic Now that we know how to reorganize calculations so that there is potential for parallelism, we will need to know how to realize the potential. For example, if we have these two operations in the C++ code, how do we tell the computer that the operations can be safely executed in parallel?

The delightful answer is that it happens completely automatically, there is nothing we need to do (and nothing we can do)!

-->
