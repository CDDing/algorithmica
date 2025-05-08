---
title: Instruction-Level Parallelism
weight: 3
---

프로그래머들이 **병렬화**라는 단어를 들으면, 대부부은 하나의 연산을 공동 문제를 해결하기 위해 서로 반독립적으로 작동하는 **쓰레드**로 명시적으로 분할하는 **멀티 코어 병렬화**를 떠올립니다.


이러한 종류의 병렬화는 주로 지연 시간을 줄이고 확장성을 확보하는 데 목적이 있지만, 효율성을 높이기 위한 것은 아닙니다. 병렬 알고리즘을 사용하면 10배 더 큰 문제를 해결할 수 있지만, 그만큼 최소 10배의 컴퓨팅 자원이 필요합니다. 병렬 하드웨어가 점점 더 [보편화](/hpc/complexity/hardware)되고 병렬 알고리즘 설계가 점점 더 중요한 분야로 떠오르고 있지만, 지금은 단일 CPU 코어만을 고려하는 것으로 범위를 제한하겠습니다.

하지만 별도의 자원 없이도 하나의 CPU 코어 내부에서 이미 존재하고 활용할 수 있는 다른 형태의 병렬화도 있습니다.

<!--

This technique only applies 

Parallel hardware is now everywhere. When you opened this page in your browser, it was retrieved by a 50-core server CPU, then parsed by an 8-core desktop CPU, and then rendered by a 400-core GPU. Not all cores were involved with serving you this page at all times — they might have been doing something else.

Parallelism helps in reducing *latency*. It is important, but for now, our main concern is not *scalability*, but *efficiency* of algorithms.

Sharing computations is an art in itself, but for now, we want to learn how to use resources that we already have more efficiently.

While multi-core parallelism is "cheating," many form of parallelism exist "for free."

Adapting algorithms for parallel hardware is important for achieving *scalability*. In the first part of this book, we will consider this technique "cheating." We only do optimizations that are truly free, and preferably don't take away resources from other processes that might be running concurrently.

-->

### 명령어 파이프라이닝

어떠한 명령어든 실행하려면, 프로세서는 먼저 다음과 같은 많은 준비 작업을 수행합니다.

- 메모리에서 기계어 코드 묶음을 가져오고,
- 이를 디코딩하여 명령어 단위로 분할하며,
- 몇몇 메모리 작업을 포함할 수 있는 명령어들을 실행하고,
- 결과를 다시 레지스터에 기록합니다.

이 전체 작업 흐름은 깁니다. 예를 들어 두 레지스터에 저장된 값을 단순히 `add`하는 연산조차도 15~20 사이클이 걸릴 수 있습니다. 이러한 지연을 숨기기 위해 현대 CPU는 파이프라이닝을 사용합니다. 즉, 하나의 명령어가 첫 번째 단계를 통과하면, 이전 명령어가 완전히 끝나기를 기다리지 않고 다음 명령어의 처리를 바로 시작하는 방식입니다.

![](img/pipeline.png)

파이프라이닝은 실제 지연 시간을 줄이지는 않지만, 실행 단계와 메모리 단계만으로 구성된 것처럼 보이게 만듭니다. 여전히 15~20 사이클의 지연은 존재하지만, 실행할 명령어 순서가 정해진 이후에는 한 번만 그 비용을 치르면 됩니다.

이 점을 고려하여, 하드웨어 제조사들은 CPU 설계의 주요 성능 지표로 '평균 명령어 지연 시간' 대신, '명령어당 사이클 수(CPI)'를 선호합니다. 유용한 명령어만을 고려한다면, CPI는 알고리즘 설계를 위한 지표로도 꽤 [훌륭합니다](/hpc/profiling/benchmarking).

완벽하게 파이프라인화된 프로세서의 CPI는 이론적으로 1에 수렴하지만, 각 파이프라인 단계를 복제하여 더 넓게 만들면 동시에 여러 명령어를 처리할 수 있어 실제 CPI는 1보다 더 낮아질 수도 있습니다. 캐시와 대부분의 산술논리장치(ALU)를 공유할 수 있기 때문에, 완전히 별도의 코어를 추가하는 것보다 비용이 덜 듭니다. 이처럼 한사이클에 여러 명령어를 실행할 수 있는 아키텍처를 **슈퍼스칼라** 라고 하며, 대부분의 현대 CPU는 이 구조를 따릅니다.

명령어 스트림에 개별적으로 처리 가능한 논리적으로 독립된 연산들이 포함되어 있을 때만 슈퍼스카라 처리의 이점을 누릴 수 있습니다. 명령어가 항상 최적의 순서로 도착하는 것은 아니기 때문에, 현대 CPU는 가능한 경우 명령어들을 순서를 바꾸어 실행(out of order execution)함으로써 전체 활용도를 높이고 파이프라인 정지를 최소화합니다. 이러한 동작이 어떻게 가능한지는 더 고급 주제에 해당하지만,<!--[a more advanced discussion](scheduling)--> 지금은 CPU가 일정 범위까지의 대기 중인 명령어들을 버퍼에 저장해 두고, 피연산자의 값이 준비되고 실행 유닛이 사용 가능해지는 즉시 해당 명령어를 실행한다고 가정하면 됩니다.

### 교육에 대한 비유

우리의 교육 시스템이 어떻게 작동하는지 생각해봅시다.

1. 같은 내용을 모든 학생에게 한 번에 전달하는 것이 더 효율적이기 때문에, 개개인이 아닌 학생 그룹을 대상으로 주제를 가르칩니다.
2. 한 기수의 학생들은 여러 선생님이 이끄는 그룹으로 나뉘며, 과제와 기타 강의 자료는 그룹 간에 공유됩니다.
3. 매년 동일한 강의가 새로운 학생들에게 반복해서 제공되어, 선생님들은 꾸준히 바쁜 상태를 유지하게 됩니다.

이러한 방식은 전체 시스템의 **처리량(throughput)**을 크게 증가시키지만, 특정 학생의 졸업까지 걸리는 **지연 시간(latency)**는 변하지 않거나, 개인 맞춤형 수업이 더 효과적일 수 있기 때문에 오히려 약간 증가할 수도 있습니다.

이러한 점들은 현대 CPU와 유사한 면이 많습니다.

1. CPU는 [SIMD 병렬화](/hpc/simd)를 사용하여, 16, 32, 64바이트 등 다양한 데이터 묶음에 동일한 연산을 수행합니다.
2. CPU에는 보통 2~4개의 실행 유닛이 있어, 기타 CPU 자원을 공유하면서 명령어들을 동시에 처리할 수 있습니다.
3. 명령어들은 파이프라인 방식으로 처리되며, 이는 마치 유치원부터 박사까지 이수하는 연수 만큼의 사이클 수를 절약하는 효과가 있습니다.

<!-- You can continue "up:" there are multiple school branches (cores), multiple schools (computers), etc. -->

이 외에도 CPU와 유사한 여러 측면이 있습니다.

- 일부 명령어는 다양한 이유로 인해 지연되기도 합니다.
- 어떤 명령어는 미리 실행되었다가 결국 폐기되기도 합니다.
- 일부 명령어는 여러 개의 독립적인 마이크로 연산으로 나뉘어 각각 따로 실행될 수도 있습니다.

파이프라인화되고 슈퍼스칼라 방식의 프로세서를 프로그래밍하는 것은 특유의 어려움이 있으며, 이번 장에서는 그 부분을 다룰 것입니다.
