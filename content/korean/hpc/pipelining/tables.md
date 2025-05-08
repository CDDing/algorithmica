---
title: Instruction Tables
weight: 3
---

<!-- This poses some additional challenges in coordinating how to execute the instructions — and also in which order. -->

실행 단계의 간섭은 디지털 전자공학에서 일반적인 개념이며, 메인 CPU 파이프라인뿐만 아니라 개별 명령어와 [메모리](/hpc/cpu-cache/mlp) 레벨에서도 적용됩니다. 대부분의 실행 유닛은 자신만의 작은 파이프라인을 가지고 있으며, 이전 명령어 후 한두 사이클만에 다른 명령어를 받을 수 있습니다.

여기서는 명령어에 대해 두 가지 다른 "[비용](/hpc/complexity)"을 사용해야 합니다.

- **지연** : 명령어의 결과를 받는 데 필요한 사이클 수
- **처리량** : 사이클당 평균적으로 실행될 수 있는 명령어 수

<!-- alternative throughput definitions, maybe in scheduling? -->

특정 아키텍처의 지연과 처리량 수치는 [명령어 테이블](https://www.agner.org/optimize/instruction_tables.pdf)이라 불리는 특수한 문서에서 확인할 수 있습니다. 필자의 Zen 2에 대한 몇 가지 예제 값은 다음과 같습니다.

| Instruction | Latency | RThroughput |
|-------------|---------|:------------|
| `jmp`       | -       | 2           |
| `mov r, r`  | -       | 1/4         |
| `mov r, m`  | 4       | 1/2         |
| `mov m, r`  | 3       | 1           |
| `add`       | 1       | 1/3         |
| `cmp`       | 1       | 1/4         |
| `popcnt`    | 1       | 1/4         |
| `mul`       | 3       | 1           |
| `div`       | 13-28   | 13-28       |


- 우리의 사고 방식은 **더 많다**는 것이 **더 나쁘다**는 비용 모델에 익숙해져 있기 때문에 대부분 사람들이 처리량 대신 처리량의 역수를 사용합니다.
- 특정 명령어가 특히 자주 발생한다면, 해당 실행 유닛이 복제되어 처리량을 증가시킬 수 있습니다(최대 1보다 많은 수 있지만, [디코드 폭(width)](/hpc/architecture/layout)를 초과할 수는 없습니다).
- 몇몇 명령어는 지연이 0입니다. 이는 이 명령어들이 스케줄러를 제어하는 데 사용되며, 실행 단계에 도달하지 않는다는 것을 의미합니다. 그러나 여전히 [CPU 프론트엔드](/hpc/architecture/layout)가 이를 처리해야 하기 때문에 처리량의 역수는 0이 아닙니다.
- 대부분의 명령어는 파이프라인화 되어 있으며, 만약 이들의 처리량 역수가 $n$이라면, 이는 실행 유닛이 $n$ 사이클 후에 또 다른 명령어를 받을 수 있다는 의미입니다. (만약 1보다 낮다면, 이는 여러 실행 유닛이 각각 다음 사이클에 다른 명령어를 받을 수 있다는 의미입니다.)예외적으로 [정수 나눗셈](/hpc/arithmetic/division)은 파이프라인화가 매우 잘 되어 있지 않거나 전혀 되지 않습니다.
- 몇몇 명령어는 피연산자의 크기 뿐만 아니라 값에 따라 지연이 달라집니다. 메모리 명령어(예: `add`와 같은 결합된 연산)의 경우, 지연은 일반적으로 최적 상황(L1 캐시 히트)에 대해 지정됩니다.

더 중요한 세부 사항들이 많지만, 현재로서는 이 모델만으로 충분합니다.

<!--

This mental model covers 80% of your needs.

Some instruction tables also list execution ports (or sometimes "pipes"). This is mostly relevant for SIMD.

This is a bit of an advanced and not well understood topic. Documentation is very obscure. people have to reverse engineer it. There are reasons to believe that folks at Intel don't know that themselves. The most comprehensive one is probably, uops.info.

There are tools like llvm-mca, but they aren't perfect either.

-->
